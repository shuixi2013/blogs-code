title: 工作小笔记——Android 自带的应用统计服务（UsageStatsService）
date: 2015-01-31 10:33:16
updated: 2015-03-07 17:27:16
categories: [Android Framework]
tags: [android]
---

最近要弄在 framework 中弄一个统计应用使用时长的功能。刚开始想着要怎么是不是要在 ActivityManagerService（AMS）的几个 Activity 的生命周期那埋几个统计点，后面发现 android 自带了一个 UsageStatsService（USS）的系统服务。这个东西统计的数据已经满足这边的需求了。只不过 android 好像只是统计了数据，并没怎么用（不过不排除 google 服务偷偷的用这个东西，google 服务应用是不开源的），所以还需要改造一下下。不过基本已经算是没啥难度了，不需要验证自己埋点是不是正确的（android 自己的服务准确性上还是值得信赖的）。这里把这个服务和之后自己的改造稍微分析一下，照例先把相关源码的位置啰嗦一下（4.2.2）：

```bash
frameworks/base/services/java/com/android/server/am/UsageStatsService.java
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
```

## UsageStatsService

### 初步认识

USS 虽然说是一个单独系统服务，但是从代码位置来看隶属于 AMS 家族：代码位于 frameworks/base/services/java/com/android/server/am 下面（在 services 包下能单独开一个文件夹的 SS 都是十分庞大的，例如：AM、WM、PM 之类）。它是由 AM 来启动的：

```java
// ===================== ActivityManagerService.java ========================

public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
... ...

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

        mUsageStatsService = new UsageStatsService(new File(
                systemDir, "usagestats").toString());
        mHeadless = "1".equals(SystemProperties.get("ro.config.headless", "0"));

... ...

    }

... ...

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
        ServiceManager.addService(SERVICE_NAME, asBinder());
    }

... ...
}
```

### 数据结构

### 统计埋点

### 保存数据

## 改造

### 方案

### 实现

## 总结

未完待续 ... ...

