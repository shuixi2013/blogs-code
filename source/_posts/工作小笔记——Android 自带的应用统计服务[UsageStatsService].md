title: 工作小笔记——Android 自带的应用统计服务（UsageStatsService）
date: 2015-01-31 10:33:16
updated: 2015-03-07 17:27:16
categories: [Android Framework]
tags: [android]
---

最近要弄在 framework 中弄一个统计应用使用时长的功能。刚开始想着要怎么是不是要在 ActivityManagerService（AMS）的几个 Activity 的生命周期那埋几个统计点，后面发现 android 自带了一个 UsageStatsService（USS）的系统服务。这个东西统计的数据已经满足这边的需求了。只不过 android 好像只是统计了数据，并没怎么用（不过不排除 google 服务偷偷的用这个东西，google 服务应用是不开源的），所以还需要改造一下下。不过基本已经算是没啥难度了，不需要验证自己埋点是不是正确的（android 自己的服务准确性上还是值得信赖的）。这里把这个服务和之后自己的改造稍微分析一下，照例先把相关源码的位置啰嗦一下（4.2.2）：

```bash
# AMS 相关
frameworks/base/services/java/com/android/server/am/UsageStatsService.java
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/java/com/android/server/am/ActivityStack.java
frameworks/base/services/java/com/android/server/am/ActivityRecord.java

# WMS 相关
frameworks/base/services/java/com/android/server/wm/WindowManagerService.java
frameworks/base/services/java/com/android/server/wm/AppWindowToken.java
```

## UsageStatsService

### 初步认识

USS 虽然说是一个单独系统服务，但是从代码位置来看隶属于 AMS 家族：代码位于 frameworks/base/services/java/com/android/server/am 下面（在 services 包下能单独开一个文件夹的 SS 都是十分庞大的，例如：AM、WM、PM 之类）。它是由 AM 来启动的：

```java
// ===================== ActivityManagerService.java ========================

public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
... ...

	// USS 属于 AMS
    /*
     * information about component usage
     */
    final UsageStatsService mUsageStatsService;

... ...

    private ActivityManagerService() {
... ...

        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system"); 
        systemDir.mkdirs();
        mBatteryStatsService = new BatteryStatsService(new File(
                systemDir, "batterystats.bin").toString());
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.getActiveStatistics().writeAsyncLocked();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);

		// 在 AMS 的构造函数被 new 出来
		// 这个 systemDir 从来上面代码来看是： /data/system，所以传给 USS 的目录是 /data/system/usagestats
        mUsageStatsService = new UsageStatsService(new File(
                systemDir, "usagestats").toString());
        mHeadless = "1".equals(SystemProperties.get("ro.config.headless", "0"));

... ...

    }

... ...

	// 前面 Binder SS 篇有说到 SS 初始化的时候会调用 AMS 的 main 函数的
    public static final Context main(int factoryTest) {
        AThread thr = new AThread();
        thr.start();
       
        synchronized (thr) {
            while (thr.mService == null) {
                try {
                    thr.wait();
                } catch (InterruptedException e) {
                } 
            }
        }

        ActivityManagerService m = thr.mService;
        mSelf = m;
        ActivityThread at = ActivityThread.systemMain();
        mSystemThread = at;
        Context context = at.getSystemContext();
        context.setTheme(android.R.style.Theme_Holo);
        m.mContext = context; 
        m.mFactoryTest = factoryTest;
        m.mMainStack = new ActivityStack(m, context, true);

		// 调用 USS 的 publish 函数
        m.mBatteryStatsService.publish(context);
        m.mUsageStatsService.publish(context);
    
        synchronized (thr) {
            thr.mReady = true;
            thr.notifyAll();
        }
   
        m.startRunning(null, null, null, null);
    
        return context;
    }

... ...
}

// ===================== UsageStatsService.java ========================

/*
 * This service collects the statistics associated with usage
 * of various components, like when a particular package is launched or
 * paused and aggregates events like number of time a component is launched
 * total duration of a component launch.
 */
public final class UsageStatsService extends IUsageStats.Stub {
... ...

    public void publish(Context context) {
        mContext = context;
		// 向 SM 注册自己
        ServiceManager.addService(SERVICE_NAME, asBinder());
    }

... ...
}
```

这里如果有个我写的 Binder 系列那些知识的话是很简单的，SS 启动 AMS，然后 AMS 创建 USS，再调用 USS 的 publish 向 SM 注册 USS，这样 framework 中的各个模块（原生的 USS 不对第三方应用开放接口的）就能通过 SM 调用 USS 提供的 IPC 接口了。那我们也清楚了 USS 是在 AMS 初始化的时候启动的。

### 数据结构

我们初步认识了 USS 之后，来看下 USS 使用的数据结构。在 USS 中有2个内部类（代码不多一次贴全了）：

```java
public final class UsageStatsService extends IUsageStats.Stub {

... ...
       
    static IUsageStats sService;
    private Context mContext;
    // 这个 Map 基本就是 USS 保存的所有数据了，以 pkg name 为 key，
    // 每一个 pkg 一个 PkgUsageStatsExtended 数据。
    // 注释中说这个保存的是从最后一次 checkin 之后的数据，这个后面再说是什么意思。
    // structure used to maintain statistics since the last checkin.
    final private Map<String, PkgUsageStatsExtended> mStats; 

    // 一共有十个时间范围，下面的时间单位是 ms
    // 就是说 USS 会给一个组件（component）启动时候划分到下面几个时间段：
    // 例如说： 250ms ~ 500ms, 2000ms ~ 3000ms, > 5000ms 等
    private static final int NUM_LAUNCH_TIME_BINS = 10; 
    private static final int[] LAUNCH_TIME_BINS = {
        250, 500, 750, 1000, 1500, 2000, 3000, 4000, 5000
    }; 

... ...

    // TimeStats 主要是保存应用的组件启动时长信息
    static class TimeStats {
        // 一共启动了几次
        int count;
        // 这个是一个数组，里面存放的是 component 启动时间段分布信息，详细看上面的解说
        int[] times = new int[NUM_LAUNCH_TIME_BINS];

        TimeStats() {
        }
        
        void incCount() {
            count++;
        }

        // 这个接口传入的参数是时长，通过时长标记这个组件启动的时间属于上面哪个时间段
        // 数组的每个元素的数值分布就代表启动的时间段的分布
        void add(int val) {
            final int[] bins = LAUNCH_TIME_BINS;
            for (int i=0; i<NUM_LAUNCH_TIME_BINS-1; i++) {
                if (val < bins[i]) {
                    times[i]++;
                    return;
                }
            }
            times[NUM_LAUNCH_TIME_BINS-1]++;
        }

        // 从 Parcel 打包的数据中实例化自己
        TimeStats(Parcel in) {
            count = in.readInt();
            final int[] localTimes = times;
            for (int i=0; i<NUM_LAUNCH_TIME_BINS; i++) {
                localTimes[i] = in.readInt();
            }
        }

        // 把自己写入 Parcel 数据包中，上面的实例化是根据写的顺序来读的
        // 这里没有显式实现 Parcelable 接口，而是直接写了这2个函数让下面
        // 的业务函数来调用的，只是内部使用的话都差不多了
        void writeToParcel(Parcel out) {
            out.writeInt(count);
            final int[] localTimes = times;
            for (int i=0; i<NUM_LAUNCH_TIME_BINS; i++) {
                out.writeInt(localTimes[i]);
            }
        }
    }
    
    // 这个算是 USS 中基本数据单元，单位是包（packagename）
    private class PkgUsageStatsExtended {
        // 这个 HaspMap 以 ComponentName（String） 为 key，
        // 保存了这个 pkg 中启动过的组件的时长信息
        final HashMap<String, TimeStats> mLaunchTimes
                = new HashMap<String, TimeStats>();
        // 这个 pkg 启动过几次
        int mLaunchCount;
        // 这个 pkg 当前总共使用了多长时间（ms）
        long mUsageTime;
        // 这个 pkg 当前组件 Paused 的时间戳（ms）
        long mPausedTime;
        // 这个 pkg 当前组件 Resumed 的时间戳（ms）
        long mResumedTime;
             
        PkgUsageStatsExtended() {
            mLaunchCount = 0; 
            mUsageTime = 0; 
        }    
        
        // 从 Parcel 打包的数据中实例化自己      
        PkgUsageStatsExtended(Parcel in) {
            mLaunchCount = in.readInt();
            mUsageTime = in.readLong();
            if (localLOGV) Slog.v(TAG, "Launch count: " + mLaunchCount
                    + ", Usage time:" + mUsageTime);
                 
            final int numTimeStats = in.readInt();
            if (localLOGV) Slog.v(TAG, "Reading comps: " + numTimeStats);
            for (int i=0; i<numTimeStats; i++) {
                String comp = in.readString();
                if (localLOGV) Slog.v(TAG, "Component: " + comp);
                TimeStats times = new TimeStats(in);
                mLaunchTimes.put(comp, times);
            }    
        }   
        
        // 更新 Resume 信息， AMS 中有组件 Resume 的时候调用
        void updateResume(String comp, boolean launched) {
            if (launched) {
                // 在 launch Resume 的时候才算一次启动
                // 后面能看到 AMS 哪些地方会埋这个点的
                mLaunchCount ++;
            }
            mResumedTime = SystemClock.elapsedRealtime();
        }

        // 更新 Pause 信息
        void updatePause() {
            mPausedTime =  SystemClock.elapsedRealtime();
            // Pause 的时候更新下使用时间
            mUsageTime += (mPausedTime - mResumedTime);
        }

        // 添加 pkg 组件启动次数信息
        void addLaunchCount(String comp) {
            TimeStats times = mLaunchTimes.get(comp);
            // 如果是组件的话，new 一个 TimeStats
            if (times == null) {
                times = new TimeStats();
                mLaunchTimes.put(comp, times);
            }
            times.incCount();
        }

        // 添加 pkg 组件启动时长信息
        void addLaunchTime(String comp, int millis) {
            TimeStats times = mLaunchTimes.get(comp);
            if (times == null) {
                times = new TimeStats();
                mLaunchTimes.put(comp, times);
            }
            times.add(millis);
        }

        // 把自己写入 Parcel 数据包中，上面的实例化是根据写的顺序来读的
        void writeToParcel(Parcel out) {
            out.writeInt(mLaunchCount);
            out.writeLong(mUsageTime);
            final int numTimeStats = mLaunchTimes.size();
            out.writeInt(numTimeStats);
            if (numTimeStats > 0) {
                for (Map.Entry<String, TimeStats> ent : mLaunchTimes.entrySet()) {
                    out.writeString(ent.getKey());
                    TimeStats times = ent.getValue();
                    times.writeToParcel(out);
                }
            }
        }

        // 清除所有 pkg 信息（包括组件的 TimeStats 信息）
        void clear() {
            mLaunchTimes.clear();
            mLaunchCount = 0;
            mUsageTime = 0;
        }
    }

... ...

}
```

上面基本上把 USS 的基本数据结构介绍清楚了。其结构是一个 Map，其中以安装了的 pkg 为条，每一条包括：

* 启动总次数
* 总共时间时间
* pkg 中每个组件启动的次数以及每次启动所需要的时间

为了增加感性认识，dumpsys 一下 usagestats 这个服务能看到磁盘的保存了的 USS 的数据信息：

<pre>
Date: 20150305
  com.android.systemui: 2 times, 26577889 ms
    com.android.systemui.usb.UsbStorageActivity: 2 starts, 2000-3000ms=2
  com.eebbk.mingming.notificationtest: 1 times, 8625444 ms
    com.eebbk.mingming.notificationtest.MainActivity: 1 starts, 500-750ms=1
  com.bbk.studyos.launcher: 3 times, 11220 ms
    com.bbk.studyos.launcher.activity.Launcher: 3 starts, >=5000ms=1
Date: 20150306
  com.android.systemui: 1 times, 30961918 ms
    com.android.systemui.usb.UsbStorageActivity: 1 starts, 250-500ms=1
  com.bbk.studyos.launcher: 2 times, 8282 ms
    com.bbk.studyos.launcher.activity.Launcher: 2 starts, 2000-3000ms=1
Date: 20150307
  com.android.systemui: 3 times, 445073 ms
    com.android.systemui.usb.UsbStorageActivity: 3 starts, 250-500ms=1, 2000-3000ms=1
  com.android.providers.usagestats: 1 times, 41882 ms
    com.android.providers.usagestats.viewer.UsageStatsViewer: 1 starts, 250-500ms=1
  com.bbk.studyos.launcher: 3 times, 27290376 ms
    com.bbk.studyos.launcher.activity.Launcher: 3 starts, >=5000ms=1
Date: 20150309
  com.android.systemui: 4 times, 6534068 ms
    com.android.systemui.usb.UsbStorageActivity: 4 starts, 250-500ms=2, 2000-3000ms=2
  com.eebbk.systemuimodedemo: 2 times, 612236 ms
    com.eebbk.systemuimodedemo.MainActivity: 2 starts, 750-1000ms=1
  com.eebbk.mingming.notificationtest: 1 times, 1813374 ms
    com.eebbk.mingming.notificationtest.MainActivity: 1 starts, 250-500ms=1
  com.bbk.studyos.launcher: 7 times, 28509 ms
    com.bbk.studyos.launcher.activity.Launcher: 7 starts, 2000-3000ms=1, >=5000ms=2
Date: 20150310
  com.android.systemui: 1 times, 0 ms
    com.android.systemui.usb.UsbStorageActivity: 1 starts, 2000-3000ms=1
  com.bbk.studyos.launcher: 1 times, 1421 ms
    com.bbk.studyos.launcher.activity.Launcher: 1 starts
Date: history.xml (old data version)
</pre>

（这里仔细看，发现组件启动那里的启动次数和后面的时间段分布有些时候会不对，时间段分别那有时会有少几次的情况出现，后面会知道偶尔出现这种情况的原因）

### 统计埋点

其实所有的统计流程基本上都是一样的，定下要统计的数据结构，下面就是埋点，然后上报数据，最后服务端保存数据。前面说了埋点是在 AMS 里面的。这里先看看 USS 提供给 AMS 的埋点接口，一共三个：

* noteResumeComponent: 通知有组件跑了 Resume 生命周期
* notePauseComponent: 通知有组件跑了 Pause 生命周期
* noteLaunchTime: 通知组件的启动时间

下面我们一个一个看：

```java
    public void noteResumeComponent(ComponentName componentName) {
        // 检测下调用者权限
        enforceCallingPermission();
        String pkgName;
        // 喜闻乐见的 SS 业务多线程同步锁，这次 USS 访问 mStats 数据和写文件用的是2个单独的锁
        synchronized (mStatsLock) {
            // 这个组件必须要包含 pkg name 才能被统计（绝大多数组件都包含 pkg name 的）
            if ((componentName == null) ||
                    ((pkgName = componentName.getPackageName()) == null)) {
                return;
            }

            // 这里有一个判断：如果当前 Resume 的这个组件和上一次那个是同一个，
            // 并且还没经过 Pause（后面指定经过 Pause 的话 mIsResumed 是 false 的），
            // 那么认为应该要调用下当前这个 pkg 的 updatePause 来更新下 Pause 信息。
            // 因为 Resume 要和 Pause 成对。这种情况的典型场景就是：
            // 一些启动 activity 的按钮处理没做好，用户一个不小心点击了好多下，
            // 然后就启动了好多个 activity，上面一个还没来得及 Pause 就有相同的 Resume 了。
            final boolean samePackage = pkgName.equals(mLastResumedPkg);
            if (mIsResumed) {
                if (mLastResumedPkg != null) {
                    // We last resumed some other package...  just pause it now
                    // to recover.
                    if (REPORT_UNEXPECTED) Slog.i(TAG, "Unexpected resume of " + pkgName
                            + " while already resumed in " + mLastResumedPkg);
                    PkgUsageStatsExtended pus = mStats.get(mLastResumedPkg);
                    if (pus != null) {
                        pus.updatePause();
                    }
                }
            }

            final boolean sameComp = samePackage
                    && componentName.getClassName().equals(mLastResumedComp);

            // 把 mIsResumed 标志设置为 true
            mIsResumed = true;
            // 保存下最后一次 Resume 的 pkg 和 comp 的名字
            mLastResumedPkg = pkgName;
            mLastResumedComp = componentName.getClassName();

            if (localLOGV) Slog.i(TAG, "started component:" + pkgName);
            // 如果 pkg 是第一次，new 一个 PkgUsageStatsExtended 保存
            PkgUsageStatsExtended pus = mStats.get(pkgName);
            if (pus == null) {
                pus = new PkgUsageStatsExtended();
                mStats.put(pkgName, pus);
            }
            // 调用 updateResume 更新 Resume 信息，
            // 注意第二个参数，如果 Resume 是不同的 pkg，那么算这个 pkg 的一次启动次数
            pus.updateResume(mLastResumedComp, !samePackage);
            // 如果 Resume 的是不同的组件，那么算这个 pkg 组件的一次启动次数
            if (!sameComp) {
                pus.addLaunchCount(mLastResumedComp);
            }

            // 这个 mLastResumeTimes 的统计数据我们暂时不管它
            Map<String, Long> componentResumeTimes = mLastResumeTimes.get(pkgName);
            if (componentResumeTimes == null) {
                componentResumeTimes = new HashMap<String, Long>();
                mLastResumeTimes.put(pkgName, componentResumeTimes);
            }
            componentResumeTimes.put(mLastResumedComp, System.currentTimeMillis());
        }
    }
```

然后下面是通知 Pause 的：

```java
    public void notePauseComponent(ComponentName componentName) {
        // 同样是检测调用者的权限
        enforceCallingPermission();

        // 同样是喜闻乐见的同步锁
        synchronized (mStatsLock) {
            // 同样得判断这个组件有没有带 pkg name 信息
            String pkgName;
            if ((componentName == null) ||
                    ((pkgName = componentName.getPackageName()) == null)) {
                return;
            }
            // 这个为了保证 Resume、Pause 的成对性，判断 mIsResumed 如果不是 true 就返回
            if (!mIsResumed) {
                if (REPORT_UNEXPECTED) Slog.i(TAG, "Something wrong here, didn't expect "
                        + pkgName + " to be paused");
                return;
            }
            // 设置 mIsResumed 为 false
            mIsResumed = false;

            if (localLOGV) Slog.i(TAG, "paused component:"+pkgName);

            // 调用 updatePause 更新 Pause 信息
            PkgUsageStatsExtended pus = mStats.get(pkgName);
            if (pus == null) {
                // Weird some error here
                Slog.i(TAG, "No package stats for pkg:"+pkgName);
                return;
            }
            pus.updatePause();
        }   
             
        // 在收到组件 Pause 信息的时候保存统计数据到文件中，这个我们后面再具体分析。
        // 这里稍微注意一下这个函数没在上面的 mStatsLock 的范围中。
        // 因为 USS 拿了2个锁处理多线程同步问题，读写文件的锁是另外一个，在这个函数会用的。  
        // Persist current data to file if needed.
        writeStatsToFile(false, false);
    }
```

最后是通知启动需要的时间的：

```java
    public void noteLaunchTime(ComponentName componentName, int millis) {
        // 还是检测调用者的权限
        enforceCallingPermission();
        String pkgName;
        if ((componentName == null) ||
                ((pkgName = componentName.getPackageName()) == null)) {
            return;
        }

        // 在收到组件启动时间的信息的时候也会激发写入文件的操作
        // 和上面的一样不在 mStatsLock 的范围内
        // Persist current data to file if needed.
        writeStatsToFile(false, false);

        // 还是喜闻乐见的同步锁
        synchronized (mStatsLock) {
            // 调用 addLaunchTime 添加组件启动时长信息
            PkgUsageStatsExtended pus = mStats.get(pkgName);
            if (pus != null) {
                pus.addLaunchTime(componentName.getClassName(), millis);
            }
        }
    }
```

这里能看到上面的 noteResume 和 notePause 中会有成对的判断（以后自己处理组件生命周期问题的时候也要多加注意）。还有这里组件启动次数和添加组件启动所需时长信息不在同一个地方，这就能解释为什么偶尔会发现这2个数据个数对不上的问题（埋点的地方不一样，总有几次漏的）。还有上面访问 mStats 和读写文件的锁是分开的，可以学习一下多线程互斥问题的时候，要多想想，不要一遇到互斥问题就所有的代码用一个锁锁住。细分可重入的代码块能提高程序并发访问的能力。

上面埋点的接口，下面我们来看下哪些地方埋了这些点，基本上在 AMS 和 WMS 中（AMS、WMS 的代码十分庞大，请善用 grep 神器）。我们还是分成3个部分来说：

#### noteResumeComponent

这里我能省略的尽量省略，因为要如何分析 AMS 和 WMS 至少要好几篇才够，先上张图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Worknote-usagestats/noteResumeComponent.png)

Resume 的埋点在 AMS 的 ActivityStack（AS） 的 completeResumeLocked 中。前面一些分析说过 AMS 有一个 activity 的堆栈，相关操作（start activity、resume activity、pause activity 等等）集中在 AS 中。其中 AMS 好像有好几个 AS，但是这里和统计相关的只有一个叫 mMainStack 的主堆栈。大体可以分为2条路：

* **branch1:**

第一条分支，是比较常见的调用 AMS 的 startActivity 发起的。然后和之前 Binder 的 SS 篇和 Broadcast 篇类似，分为 activity 的进程时候存在。如果不存在，继续走 branch1 后面的。如果进程已经存在，就算 branch 2 前面各种场景调用中的一种，可以直接调用 mMainStack 的 startActivity 相关函数，最后会调用 AS 的 resumeTopActivityLocked 然后去 branch2。这里我们先看进程不存在的情况，告诉 Zygote fork 出进程，然后会跑 ActivityThread（AT） 里面的 main 函数，然后 main 里面会 new AT，然后调用 AT 的 attach 函数，然后会 IPC 调用 AMS 的 attachApplication(Locked)（这些前面那些篇章都分析过了，省略贴代码了）。这里稍微说下 AMS 的 mMainStack（AS） 和 ActivityRecord（AR）这2个东西。在 AMS 的 main 里会创建一个 AS 这里就是主堆栈：

```java
    public static final Context main(int factoryTest) {
        AThread thr = new AThread();   
        thr.start();

        synchronized (thr) {
            while (thr.mService == null) { 
                try {
                    thr.wait();                    
                } catch (InterruptedException e) {
                }             
            }                 
        }

        ActivityManagerService m = thr.mService;
        mSelf = m;
        ActivityThread at = ActivityThread.systemMain(); 
        mSystemThread = at;
        Context context = at.getSystemContext();
        context.setTheme(android.R.style.Theme_Holo);
        m.mContext = context;
        m.mFactoryTest = factoryTest;  
        // new 出来的 AS，最后一参数 mainStack 是 true
        m.mMainStack = new ActivityStack(m, context, true);

        m.mBatteryStatsService.publish(context);
        m.mUsageStatsService.publish(context);

        synchronized (thr) {
            thr.mReady = true;
            thr.notifyAll();
        }

        m.startRunning(null, null, null, null);

        return context;
    } 
```

然后 AR 这个东西么，经过前面一些 SS 的分析都知道 SS 都喜欢管自己的一些数据结构叫 XxxxRecord（ProcessRecord, ServiceRecord, TaskRecord, BroadcastRecord ... ...），所以这里的 AR 对应的就是 activity 的记录了。然后在 AS 中的 startActivityLocked 会 new 一个 AR：

```java
    final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo, IBinder resultTo,
            String resultWho, int requestCode,
            int callingPid, int callingUid, int startFlags, Bundle options,
            boolean componentSpecified, ActivityRecord[] outActivity) {

        int err = ActivityManager.START_SUCCESS;

... ...

        // 启动一个 activity 对应就有一个 AR
        ActivityRecord r = new ActivityRecord(mService, this, callerApp, callingUid,
                intent, resolvedType, aInfo, mService.mConfiguration,
                resultRecord, resultWho, requestCode, componentSpecified);
        if (outActivity != null) {
            outActivity[0] = r;
        }
        
        if (mMainStack) {
            if (mResumedActivity == null
                    || mResumedActivity.info.applicationInfo.uid != callingUid) {
                if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid, "Activity start")) {
                    PendingActivityLaunch pal = new PendingActivityLaunch();
                    pal.r = r;
                    pal.sourceRecord = sourceRecord;
                    pal.startFlags = startFlags;
                    mService.mPendingActivityLaunches.add(pal);
                    mDismissKeyguardOnNextActivity = false;
                    ActivityOptions.abort(options);
                    return ActivityManager.START_SWITCHES_CANCELED;
                }
            }
            
            if (mService.mDidAppSwitch) {
                // This is the second allowed switch since we stopped switches,
                // so now just generally allow switches.  Use case: user presses
                // home (switches disabled, switch to home, mDidAppSwitch now true);
                // user taps a home icon (coming from home so allowed, we hit here
                // and now allow anyone to switch again).
                mService.mAppSwitchesAllowedTime = 0;
            } else {
                mService.mDidAppSwitch = true;
            }

            mService.doPendingActivityLaunchesLocked(false);
        }

        err = startActivityUncheckedLocked(r, sourceRecord,
                startFlags, true, options);
        if (mDismissKeyguardOnNextActivity && mPausingActivity == null) {
            // Someone asked to have the keyguard dismissed on the next
            // activity start, but we are not actually doing an activity
            // switch...  just dismiss the keyguard now, because we
            // probably want to see whatever is behind it.
            mDismissKeyguardOnNextActivity = false;
            mService.mWindowManager.dismissKeyguard();
        }
        return err;
    }
```

这里虽然没具体分析 startActivity 的过程，但是我们可以猜到一个 activity 对应一个 AR 记录。并且是在 startActivity 的时候创建的，所以等 AT attach AMS 的时候 AR 取最顶层的 AR 就是当前 startActivity 对应的 AR 记录。所以 AMS 会先调用 mMainStack(AS) topRunningActivityLocked 取顶层的 AR，然后 realStartActivityLocked 去真正的启动 activity：

```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
... ...

        // 取当前的启动 activity 的 AR 记录（顶层）
        // See if the top visible activity is waiting to run in this process...
        ActivityRecord hr = mMainStack.topRunningActivityLocked(null);
        if (hr != null && normalMode) {
            if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                    && processName.equals(hr.processName)) {
                try {
                    if (mHeadless) {
                        // 调用 mMainStack 的 realStartActivityLocked
                        // 传过去的 andResume 是 true
                        Slog.e(TAG, "Starting activities not supported on headless device: " + hr);
                    } else if (mMainStack.realStartActivityLocked(hr, app, true, true)) {
                        didSomething = true;
                    }
                } catch (Exception e) {
                    Slog.w(TAG, "Exception in new application when starting activity "
                          + hr.intent.getComponent().flattenToShortString(), e);
                    badApp = true;
                }
            } else {
                mMainStack.ensureActivitiesVisibleLocked(hr, null, processName, 0);
            }
        }   

... ...

        return true;
    }
```

然后是 AS 里的处理：

```java
    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

... ...
        
        // 从 AMS 那里传过来的 andResume 是 true     
        if (andResume) {
            // As part of the process of launching, ActivityThread also performs
            // a resume.
            r.state = ActivityState.RESUMED;
            if (DEBUG_STATES) Slog.v(TAG, "Moving to RESUMED: " + r
                    + " (starting new instance)");
            r.stopped = false;
            // 保存当前 resume 的 AR 记录
            mResumedActivity = r;
            r.task.touchActiveTime();
            if (mMainStack) {
                mService.addRecentTaskLocked(r.task);
            }       
            // AMS Resume 的集中埋点处
            completeResumeLocked(r);
            checkReadyForSleepLocked();
            if (DEBUG_SAVED_STATE) Slog.i(TAG, "Launch completed; removing icicle of " + r.icicle);
        } else {
            // This activity is not starting in the resumed state... which
            // should look like we asked it to pause+stop (but remain visible),
            // and it has done so and reported back the current icicle and
            // other state.
            if (DEBUG_STATES) Slog.v(TAG, "Moving to STOPPED: " + r
                    + " (starting in stopped state)");
            r.state = ActivityState.STOPPED;
            r.stopped = true;
        }                       
                                
        // Launch the new version setup screen if needed.  We do this -after-
        // launching the initial activity (that is, home), so that it can have
        // a chance to initialize itself while in the background, making the
        // switch back to it faster and look better.
        if (mMainStack) {
            mService.startSetupActivityLocked();
        }       
            
        return true;
    }
```

然后剩下的流程比较简单，我把代码合并在一起了：

```java
// =================== ActivityStack.java ======================

    /*
     * Once we know that we have asked an application to put an activity in
     * the resumed state (either by launching it or explicitly telling it),
     * this function updates the rest of our state to match that fact.
     */
    private final void completeResumeLocked(ActivityRecord next) {
        next.idle = false;
        next.results = null;
        next.newIntents = null;

        // schedule an idle timeout in case the app doesn't do it for us.
        Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG);
        msg.obj = next;
        mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT);
                    
        if (false) {
            // The activity was never told to pause, so just keep
            // things going as-is.  To maintain our own state,
            // we need to emulate it coming back and saying it is
            // idle.
            msg = mHandler.obtainMessage(IDLE_NOW_MSG);
            msg.obj = next;
            mHandler.sendMessage(msg);
        }   
        
        // 调用 AMS 的 reportResumedActivityLocked，传递当前的 AR 记录
        if (mMainStack) {
            mService.reportResumedActivityLocked(next);
        }
       
        if (mMainStack) {
            mService.setFocusedActivityLocked(next);
        }
        next.resumeKeyDispatchingLocked();
        ensureActivitiesVisibleLocked(null, 0);
        mService.mWindowManager.executeAppTransition();
        mNoAnimActivities.clear();

        // Mark the point when the activity is resuming
        // TODO: To be more accurate, the mark should be before the onCreate,
        //       not after the onResume. But for subsequent starts, onResume is fine.
        if (next.app != null) {
            synchronized (mService.mProcessStatsThread) {
                next.cpuTimeAtResume = mService.mProcessStats.getCpuTimeForPid(next.app.pid);
            }
        } else {
            next.cpuTimeAtResume = 0; // Couldn't get the cpu time of process
        }
    }

// =================== ActivityManagerService.java ======================

    void reportResumedActivityLocked(ActivityRecord r) {
        //Slog.i(TAG, "**** REPORT RESUME: " + r);
        updateUsageStats(r, true);
    }

    void updateUsageStats(ActivityRecord resumedComponent, boolean resumed) {
        if (resumed) {
            // 调用 USS 的 noteResumeComponent，当前的 AR 中有 ComponentName 信息
            mUsageStatsService.noteResumeComponent(resumedComponent.realActivity);
        } else {
            mUsageStatsService.notePauseComponent(resumedComponent.realActivity);
        }
    }
```

其实 AS 是 AMS 中分出去的一部分，AS 中还有不少回调用 AMS 的地方的。

* **branch2:**

分支2的话主要是 AMS 和 AS 中有很多种情况会调用 AS 的 resumeTopActivityLocked，例如说按 Home 键回桌面；当前应用挂了，要恢复前一个应用；或者七七八八别的情况。由于这里主要不是分析 AMS 的地方，所以把这里省略了，主要从 resumeTopActivityLocked 这个入口开始说起（分支1中 startActivity 进程已经存在的情况最后也会走这个地方的）。从这里说其实就比较简单了：

```java
    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        // 取最顶端的 AR 记录，作为下一个要 resume 的 activity
        // Find the first activity that is not finishing.
        ActivityRecord next = topRunningActivityLocked(null);

... ...

        // 要 resume 的 activity 进程还在
        if (next.app != null && next.app.thread != null) {
            if (DEBUG_SWITCH) Slog.v(TAG, "Resume running: " + next);

            // This activity is now becoming visible.
            mService.mWindowManager.setAppVisibility(next.appToken, true);

            // schedule launch ticks to collect information about slow apps.
            next.startLaunchTickingLocked();

            ActivityRecord lastResumedActivity = mResumedActivity;
            ActivityState lastState = next.state;

            mService.updateCpuStats();     
            
            if (DEBUG_STATES) Slog.v(TAG, "Moving to RESUMED: " + next + " (in existing)");
            next.state = ActivityState.RESUMED;
            // 保存当前的 resume 的 AR 信息
            mResumedActivity = next;       
            next.task.touchActiveTime();   
            if (mMainStack) {
                mService.addRecentTaskLocked(next.task);
            }
            mService.updateLruProcessLocked(next.app, true);
            updateLRUListLocked(next);

... ...

            try {
                // Deliver all pending results.
                ArrayList a = next.results;
                if (a != null) {
                    final int N = a.size();
                    if (!next.finishing && N > 0) {
                        if (DEBUG_RESULTS) Slog.v(
                                TAG, "Delivering results to " + next
                                + ": " + a);
                        next.app.thread.scheduleSendResult(next.appToken, a);
                    }
                }

                if (next.newIntents != null) {
                    next.app.thread.scheduleNewIntent(next.newIntents, next.appToken);
                }

                EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY,
                        next.userId, System.identityHashCode(next),
                        next.task.taskId, next.shortComponentName);

                next.sleeping = false;
                showAskCompatModeDialogLocked(next);
                next.app.pendingUiClean = true;
                // 执行 activity 的 Resume 生命周期
                next.app.thread.scheduleResumeActivity(next.appToken,
                        mService.isNextTransitionForward());

                checkReadyForSleepLocked();

            } catch (Exception e) {
                // Whoops, need to restart this activity!
                if (DEBUG_STATES) Slog.v(TAG, "Resume failed; resetting state to "
                        + lastState + ": " + next);
                next.state = lastState;
                mResumedActivity = lastResumedActivity;
                Slog.i(TAG, "Restarting because process died: " + next);
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else {
                    if (SHOW_APP_STARTING_PREVIEW && mMainStack) {
                        mService.mWindowManager.setAppStartingWindow(
                                next.appToken, next.packageName, next.theme,
                                mService.compatibilityInfoForPackageLocked(
                                        next.info.applicationInfo),
                                next.nonLocalizedLabel,
                                next.labelRes, next.icon, next.windowFlags,
                                null, true);
                    }
                }
                // 如果执行 activity Resume 生命周期出现错误的话（IPC 通信错误之类的）
                // 调用 startSpecificActivityLocked 重新启动这个 activity 
                // 然后里面会调用 realStartActivityLocked 就和分支1后面一样了
                startSpecificActivityLocked(next, true, false);
                return true;
            }
            
            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.visible = true;
                // 如果执行 activity Resume 生命周期没有错误的话就调用 completeResumeLocked
                // 之后的流程和分支1一样的
                completeResumeLocked(next);
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
                return true;
            }
            next.stopped = false;

        } else {
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    mService.mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(
                                    next.info.applicationInfo),
                            next.nonLocalizedLabel,
                            next.labelRes, next.icon, next.windowFlags,
                            null, true);
                }
                if (DEBUG_SWITCH) Slog.v(TAG, "Restarting: " + next);
            }
            // 如果要 resume 的 activity 的进程不在了，也需要重新启动
            startSpecificActivityLocked(next, true, true);
        }

        return true;
    }
```

分支2从各种情况调用到 AS 的 resumeTopActivityLocked，然后最后辗转几次还是到了 resume 的埋点函数 completeResumeLocked 这里。然后最后调用 USS 的接口，收集数据。

#### notePauseComponent

接下来是调用通知 Pause 的接口，也是先上图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Worknote-usagestats/notePauseComponent.png)

首先从图上看，埋点函数是 AS 的 startPausingLocked:

```java
    private final void startPausingLocked(boolean userLeaving, boolean uiSleeping) {
        // 检测下当前是不是有 AR 正处于 Pause 的处理中，有的话就打印一个错误信息
        // 因为一次只能有一个 AR 处于 Pause 的处理中
        if (mPausingActivity != null) {
            RuntimeException e = new RuntimeException();
            Slog.e(TAG, "Trying to pause when pause is already pending for "
                  + mPausingActivity, e);
        }
        // 要 Pause 是当前 Resume 的 AR，所以说 Resume 和 Pause 是成对的
        ActivityRecord prev = mResumedActivity;
        if (prev == null) {
            // 如果当前 Resume 的 AR 是 null 的话，说明有点问题，
            // 调用 resumeTopActivityLocked 去让一个 AR 处于 Resume 状态
            RuntimeException e = new RuntimeException();
            Slog.e(TAG, "Trying to pause when nothing is resumed", e);
            resumeTopActivityLocked(null);
            return;
        }
        if (DEBUG_STATES) Slog.v(TAG, "Moving to PAUSING: " + prev);
        else if (DEBUG_PAUSE) Slog.v(TAG, "Start pausing: " + prev);
        // 把 mResumedActivity 设置为 null，
        // 之前那个 Resume 的 AR 保存为 mPausingActivity，顺带更新下 AR 的状态
        // mResumedActivity 的话在前面 resumeTopActivityLocked 和 realStartActivityLocked 有设置，
        // 可以倒回去看一下，所以说只有保证 activity 的 Resume 在 Pause 前面才真正常运作
        mResumedActivity = null;
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        prev.state = ActivityState.PAUSING;
        prev.task.touchActiveTime();
        prev.updateThumbnail(screenshotActivities(prev), null);

        mService.updateCpuStats();

        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG, "Enqueueing pending pause: " + prev);
            try {
                EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                        prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName);
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags);
                if (mMainStack) {
                    // 调用 AMS 的 updateUsageStats 接口，传递 Pause 的 AR，第二参数 resumed 为 false
                    // resumed 为 false 的话会调用 USS 的 notePauseComponent 接口，这里不贴代码了
                    mService.updateUsageStats(prev, false);
                }
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
        }

... ...

    }
```

然后调用 AS 埋点函数的大致分为3个分支。每个分支也是 AMS 里面有很多有种情况会调用，照样省略：

* **branch1:**

分支1 AMS 里面会有很多种情况调用 AS 的 finishActivityLocked，最常见的就是按 back 键或是调用 finishActivity 结束 activity，那按生命周期首先得进入 Pause 状态处理：

```java
    /*
     * @return Returns true if this activity has been removed from the history
     * list, or false if it is still in the list and will be removed later.
     */
    final boolean finishActivityLocked(ActivityRecord r, int index,
            int resultCode, Intent resultData, String reason, boolean oomAdj) {
        return finishActivityLocked(r, index, resultCode, resultData, reason, false, oomAdj);
    }

    /*
     * @return Returns true if this activity has been removed from the history
     * list, or false if it is still in the list and will be removed later.
     */
    final boolean finishActivityLocked(ActivityRecord r, int index, int resultCode,
            Intent resultData, String reason, boolean immediate, boolean oomAdj) {
        if (r.finishing) {
            Slog.w(TAG, "Duplicate finish request for " + r);
            return false;
        }

... ...

        if (immediate) {
            return finishCurrentActivityLocked(r, index,
                    FINISH_IMMEDIATELY, oomAdj) == null;
        } else if (mResumedActivity == r) {
            boolean endTask = index <= 0   
                    || (mHistory.get(index-1)).task != r.task;
            if (DEBUG_TRANSITION) Slog.v(TAG, 
                    "Prepare close transition: finishing " + r);
            mService.mWindowManager.prepareAppTransition(endTask
                    ? WindowManagerPolicy.TRANSIT_TASK_CLOSE
                    : WindowManagerPolicy.TRANSIT_ACTIVITY_CLOSE, false);
       
            // Tell window manager to prepare for this one to be removed.
            mService.mWindowManager.setAppVisibility(r.appToken, false);
            
            // 如果没有 AR 正在处于 Pause 状态，调用 startPausingLocked 开始 Pause 状态处理 
            if (mPausingActivity == null) {
                if (DEBUG_PAUSE) Slog.v(TAG, "Finish needs to pause: " + r);
                if (DEBUG_USER_LEAVING) Slog.v(TAG, "finish() => pause with userLeaving=false");
                startPausingLocked(false, false);
            }

        } else if (r.state != ActivityState.PAUSING) {
            // If the activity is PAUSING, we will complete the finish once
            // it is done pausing; else we can just directly finish it here.
            if (DEBUG_PAUSE) Slog.v(TAG, "Finish not pausing: " + r);
            return finishCurrentActivityLocked(r, index,
                    FINISH_AFTER_PAUSE, oomAdj) == null;
        } else {
            if (DEBUG_PAUSE) Slog.v(TAG, "Finish waiting for pause of: " + r);
        }

        return false;
    }


```

* **branch2:**

分支2的话是前面说过的很多种情况会调用的 AS 的 resumeTopActivityLocked，例如启动一个新的 activity（前面说了会调用 resumeTopActivityLocked 的），就会先让当前处于前台的 activity 进入 Pause 状态：

```java
    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        // Find the first activity that is not finishing.
        ActivityRecord next = topRunningActivityLocked(null);

... ...

        // 如果当前有正处于 Resume 的 AR，调用 startPausingLocked 把当前 Resume 的 AR
        // 变成 Pause 状态
        // We need to start pausing the current activity so the top one
        // can be resumed...
        if (mResumedActivity != null) {
            if (DEBUG_SWITCH) Slog.v(TAG, "Skip resume: need to start pausing");
            // At this point we want to put the upcoming activity's process
            // at the top of the LRU list, since we know we will be needing it
            // very soon and it would be a waste to let it get killed if it
            // happens to be sitting towards the end.
            if (next.app != null && next.app.thread != null) {
                // No reason to do full oom adj update here; we'll let that
                // happen whenever it needs to later.
                mService.updateLruProcessLocked(next.app, false);
            }               
            startPausingLocked(userLeaving, false);
            return true;
        }  

... ...

        return true;
    }
```

* **branch3:**

分支3是很多种情况会调用的 AS 的 checkReadyForSleepLocked，好像是用处理休眠的。例如说放一段时间，设备会进入休眠状态，当然得把前台正在运行的 activity 变成 Pause 状态：

```java
    void checkReadyForSleepLocked() {
        if (!mService.isSleeping()) {
            // Do not care.
            return;
        }

        if (!mSleepTimeout) {
            if (mResumedActivity != null) {
                // Still have something resumed; can't sleep until it is paused.
                if (DEBUG_PAUSE) Slog.v(TAG, "Sleep needs to pause " + mResumedActivity);
                if (DEBUG_USER_LEAVING) Slog.v(TAG, "Sleep => pause with userLeaving=false");
                // 休眠没超时的会把当前处于前台的 activity 变成 Pause 状态
                startPausingLocked(false, true);
                return;
            }
            if (mPausingActivity != null) {
                // Still waiting for something to pause; can't sleep yet.
                if (DEBUG_PAUSE) Slog.v(TAG, "Sleep still waiting to pause " + mPausingActivity);
                return;
            }

... ...

        }

        mHandler.removeMessages(SLEEP_TIMEOUT_MSG);

        if (mGoingToSleep.isHeld()) {
            mGoingToSleep.release();
        }
        if (mService.mShuttingDown) {
            mService.notifyAll();
        }
    }
```

#### addLaunchTime

最后是记录启动时长的接口，也先上图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Worknote-usagestats/addLaunchTime.png)

启动所需时长的埋点函数是 AR 中的 windowsDraw：

```java
    // stack(AS), service(AMS), realActivity(ComponentName) 都在 AS 的 startActivityLocked 的时候
    // new AR 传过去的，代码前面有，可以倒回去看一下
    public void windowsDrawn() {
        synchronized(service) {
            if (launchTime != 0) {
                // 取当前的时间
                final long curTime = SystemClock.uptimeMillis();
                // 拿当前时间减启动时候的时间
                final long thisTime = curTime - launchTime;
                // 看下 AS 的 mInitialStartTime 是不是 0，如果不是的话拿 AS 的 mInitialStartTime 来计算，
                // 如果 mInitialStartTime 是 0，那么当 launchTime 来计算
                final long totalTime = stack.mInitialStartTime != 0
                        ? (curTime - stack.mInitialStartTime) : thisTime;
                if (ActivityManagerService.SHOW_ACTIVITY_START_TIME) {
                    EventLog.writeEvent(EventLogTags.AM_ACTIVITY_LAUNCH_TIME,
                            userId, System.identityHashCode(this), shortComponentName,
                            thisTime, totalTime);
                    StringBuilder sb = service.mStringBuilder;
                    sb.setLength(0);
                    sb.append("Displayed ");
                    sb.append(shortComponentName);
                    sb.append(": ");
                    TimeUtils.formatDuration(thisTime, sb);
                    if (thisTime != totalTime) {
                        sb.append(" (total ");
                        TimeUtils.formatDuration(totalTime, sb);
                        sb.append(")");
                    }   
                    Log.i(ActivityManagerService.TAG, sb.toString());
                }   
                stack.reportActivityLaunchedLocked(false, this, thisTime, totalTime);
                // 通过前面计算得到的时间，传给 USS 保存
                if (totalTime > 0) {
                    service.mUsageStatsService.noteLaunchTime(realActivity, (int)totalTime);
                }   
                launchTime = 0;
                stack.mInitialStartTime = 0;
            }   
            startTime = 0;
            finishLaunchTickingLocked();
        }   
    }
```

这里 AR 的 launchTime 和对应的 AS（mMainStack） 的 mInitialStartTime 是在 AS 的 startSpecificActivityLocked 中设置的：

```java
    private final void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid);
        
        // 如果要启动的 activity 对应的 AR 的 launchTime 为 0，取当前时间
        if (r.launchTime == 0) {     
            r.launchTime = SystemClock.uptimeMillis();
            if (mInitialStartTime == 0) {
                mInitialStartTime = r.launchTime;
            }
        } else if (mInitialStartTime == 0) {
            // 或者如果 mInitialStartTime 为 0，也取当前时间
            mInitialStartTime = SystemClock.uptimeMillis();
        }

... ...

    }
```

然后我们来看下在哪里调用了 startSpecificActivityLocked 这个函数。从 AMS 的 startActivity 开始，如果是第一次启动 activity，AS 中的 startActivityLocked 会创建新的 AR，然后转几下（什么 startActivityUncheckedLocked 什么之类的，中间很麻烦的，以后分析 AMS 的时候再具体说），最后到 resumeTopActivityLocked，由于这个时候 AR 的 app 信息是 null （第一次，进程还没跑）的就会调用到 startSpecificActivityLocked，然后由于 app 信息为 null，最后再调用到 AMS 的 startProcessLocked 去让 Zygote 去 fork 进程，然后就是前面 noteResumeComponent 的分支1流程了。如果不是第一次启动，然后进程信息还在（也就是 noteResumeComponent 的分支2），那么也会跑 startSpecificActivityLocked 的。

所以不管时候需要启动进程，startActivity 都要跑 startSpecificActivityLocked 这个函数，然后开始启动 activity 的计时（从这里可以看得出如果需要启动进程的话，计时肯定会很长的）。

弄清楚了从那开始计算时间，那么我看看在那结束计时的。从上面图来看，是在 WMS 中结束统计时长的，WMS 每一个 window 都有一个 AppWindowToken（AWT） 的结构，这个结构里面有一个 updateReportedVisibilityLocked 函数：

```java
    void updateReportedVisibilityLocked() {
        if (appToken == null) {
            return; 
        }       

        int numInteresting = 0;
        int numVisible = 0;
        int numDrawn = 0;
        boolean nowGone = true;

        if (WindowManagerService.DEBUG_VISIBILITY) Slog.v(WindowManagerService.TAG,
                "Update reported visibility: " + this);
        final int N = allAppWindows.size();
        for (int i=0; i<N; i++) {
            WindowState win = allAppWindows.get(i);
            if (win == startingWindow || win.mAppFreezing
                    || win.mViewVisibility != View.VISIBLE
                    || win.mAttrs.type == TYPE_APPLICATION_STARTING
                    || win.mDestroying) {
                continue;
            }
            if (WindowManagerService.DEBUG_VISIBILITY) {
                Slog.v(WindowManagerService.TAG, "Win " + win + ": isDrawn="
                        + win.isDrawnLw()
                        + ", isAnimating=" + win.mWinAnimator.isAnimating());
                if (!win.isDrawnLw()) {
                    Slog.v(WindowManagerService.TAG, "Not displayed: s=" + win.mWinAnimator.mSurface
                            + " pv=" + win.mPolicyVisibility
                            + " mDrawState=" + win.mWinAnimator.mDrawState
                            + " ah=" + win.mAttachedHidden
                            + " th="
                            + (win.mAppToken != null
                                    ? win.mAppToken.hiddenRequested : false)
                            + " a=" + win.mWinAnimator.mAnimating);
                }
            }
            numInteresting++;
            if (win.isDrawnLw()) {
                numDrawn++;
                if (!win.mWinAnimator.isAnimating()) {
                    numVisible++;
                }
                nowGone = false;
            } else if (win.mWinAnimator.isAnimating()) {
                nowGone = false;
            }
        }

        boolean nowDrawn = numInteresting > 0 && numDrawn >= numInteresting;
        boolean nowVisible = numInteresting > 0 && numVisible >= numInteresting;
        if (!nowGone) {
            // If the app is not yet gone, then it can only become visible/drawn.
            if (!nowDrawn) {
                nowDrawn = reportedDrawn;
            }
            if (!nowVisible) {
                nowVisible = reportedVisible;
            }
        }
        if (WindowManagerService.DEBUG_VISIBILITY) Slog.v(WindowManagerService.TAG, "VIS " + this + ": interesting="
                + numInteresting + " visible=" + numVisible);
        // 前面那一堆 nowDrawn, reportedDrawn 有空再理会它们，
        // 反正这个 REPORT_APPLICATION_TOKEN_DRAWN 肯定能发出去的
        if (nowDrawn != reportedDrawn) {
            if (nowDrawn) {
                Message m = service.mH.obtainMessage(
                        H.REPORT_APPLICATION_TOKEN_DRAWN, this);
                service.mH.sendMessage(m);
            }
            reportedDrawn = nowDrawn;
        }
        if (nowVisible != reportedVisible) {
            if (WindowManagerService.DEBUG_VISIBILITY) Slog.v(
                    WindowManagerService.TAG, "Visibility changed in " + this
                    + ": vis=" + nowVisible);
            reportedVisible = nowVisible;
            Message m = service.mH.obtainMessage(
                    H.REPORT_APPLICATION_TOKEN_WINDOWS,
                    nowVisible ? 1 : 0,
                    nowGone ? 1 : 0,
                    this);
            service.mH.sendMessage(m);
        }
    }
```

然后是 WMS 里面的 Handler 处理：

```java
                case REPORT_APPLICATION_TOKEN_DRAWN: {
                    final AppWindowToken wtoken = (AppWindowToken)msg.obj;
               
                    try {
                        if (DEBUG_VISIBILITY) Slog.v(
                                TAG, "Reporting drawn in " + wtoken);
                        // 这个 appToken 是 AR 那里实现的一个 Binder 对象
                        wtoken.appToken.windowsDrawn();
                    } catch (RemoteException ex) { 
                    } 
                } break;
```

懒得说 Binder 那些东西了，忘了的自觉去 Binder 篇相关的东西，直接上 AR 中相关的东西：

```java
    static class Token extends IApplicationToken.Stub {
        final WeakReference<ActivityRecord> weakActivity;

        Token(ActivityRecord activity) {
            weakActivity = new WeakReference<ActivityRecord>(activity);
        }

        @Override public void windowsDrawn() throws RemoteException {
            // 最后还是调用 AR 中的 windowsDrawn 了
            ActivityRecord activity = weakActivity.get(); 
            if (activity != null) {        
                activity.windowsDrawn();       
            }
        }

... ...

    }
```

上面 AWP 的 updateReportedVisibilityLocked 在 WMS 中也是有很多地方会调用的，例如 relayoutWindow、开始窗口变化动画等等地方。从这里可以看得出，USS 中统计 activity 启动需要的时间，是从发起 startActivity 请求（startSpecificActivityLocked）开始，直到对应 activity 的窗口开始绘制（窗口可见）这段时间。


这里算是把3个接口的埋点分析完了，AMS 和 WMS 真的是太复杂了，从代码上看很费劲。最后上一个官方 sdk doc 上的 activity 的生命周期图，结合上面埋点的分析，对 android 对 activity 生命周期实现、管理有更加深入的理解：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Worknote-usagestats/activity_lifecycle.png)

然后再附上一段官方的说明：

* <font color="#ff0000">The entire lifetime</font> of an activity happens between the first call to **onCreate(Bundle)** through to a single final call to onDestroy(). An activity will do all setup of "global" state in onCreate(), and release all remaining resources in **onDestroy()**. For example, if it has a thread running in the background to download data from the network, it may create that thread in onCreate() and then stop the thread in onDestroy().

* <font color="#ff0000">The visible lifetime</font> of an activity happens between a call to **onStart()** until a corresponding call to **onStop()**. During this time the user can see the activity on-screen, though it may not be in the foreground and interacting with the user. Between these two methods you can maintain resources that are needed to show the activity to the user. For example, you can register a BroadcastReceiver in onStart() to monitor for changes that impact your UI, and unregister it in onStop() when the user no longer sees what you are displaying. The onStart() and onStop() methods can be called multiple times, as the activity becomes visible and hidden to the user.

* <font color="#ff0000">The foreground lifetime</font> of an activity happens between a call to **onResume()** until a corresponding call to **onPause()**. During this time the activity is in front of all other activities and interacting with the user. An activity can frequently go between the resumed and paused states -- for example when the device goes to sleep, when an activity result is delivered, when a new intent is delivered -- so the code in these methods should be fairly lightweight. 

从 onCreate 到 onDestroy 算是 activity 的所有时间，从 onStart 到 onStop 算是 activity 处于可见的时间，从 onResume 到 onPause 算是 activity 的处于前台的时间。这里比较迷惑的是可见时间和处于前台的时候。这里稍微说明下，所谓处于前台是处于当前 activity 堆栈（AS）的最顶部，能够接收输入焦点，例如说正在交互的 activity。那所谓的可见时间呢，其实有些时候就算 activity 不在 AS 的最顶部，也是可见的，最典型的情况上当前正在交互的 activity 是个半透明的，所以它下面那个 activity 是可见的，但是不处于前台（处于 onPause 状态，但是没还到 onStop 状态）。所以上面的图，可见时间比前台时间要长就是这个原因。从这个来看 USS 统计的启动时长，感觉像是从 onCreate 到 onStart 那段时间。

### 保存数据

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Worknote-usagestats/usage-data.png)

## 改造

### 方案

### 实现

## 总结

未完待续 ... ...

