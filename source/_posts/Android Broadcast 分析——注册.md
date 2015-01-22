title: Android Broadcast 分析——注册
date: 2015-01-22 10:07:16
tags: [android]
---

## 背景

我们的平板增加了一个关机闹钟的功能。就是设备在关机的情况也能唤醒然后响闹钟。实现方法是：在 kernel 中增加一个 RTC 时钟（关机情况下，板载的 RTC 时钟还在运行的，所以你关机一段时间发现电池会变少），然后提前一段时间开机（我们设定的是1分钟，得给点时间给开机，我们的平板开机要40多秒咧），然后闹钟模块接收开机广播（`ACTION_BOOT_COMPLETED`），然后把 android 应用层的闹钟设置下去，然后就能实现关机闹钟功能了。

但是最近遇到一个问题：说关机闹钟，到了时间设备能唤醒，但是闹钟不响。于是分析一下 android 广播的处理流程（这里以开机广播为例）。然后顺带记录下分析过程，以后忘记看几眼能捡起来。

android 广播的注册方式分为2种：一种是动态注册，一种是静态注册。哦，对了，在说之前，照例把相关源码位置啰嗦一下（4.2.2）：

```bash
# MemroyFile 是 ashmem java 层接口
frameworks/base/core/java/os/Parcel.java
frameworks/base/core/java/os/Parcelable.java
frameworks/base/core/java/os/ParcelFileDescriptor.java
frameworks/base/core/java/os/MemoryFile.java

# jni 相关
frameworks/base/core/jni/android_os_Parcel.h
frameworks/base/core/jni/android_os_MemoryFile.cpp
frameworks/base/core/jni/android_os_Parcel.cpp
libnativehelper/JNIHelp.cpp

# 封装了 ashmem 驱动的 c 接口
system/core/include/cutils/ashmem.h
system/core/libcutils/ashmem-dev.c

# MemoryXx 是 ashmem 的 native 接口
frameworks/native/include/binder/Parcel.h
frameworks/native/include/binder/IMemory.h
frameworks/native/include/binder/MemoryHeapBase.h
frameworks/native/include/binder/MemoryBase.h
frameworks/native/libs/binder/Parcel.cpp
frameworks/native/libs/binder/Memory.cpp
frameworks/native/libs/binder/MemoryHeapBase.cpp
frameworks/native/libs/binder/MemoryBase.cpp

# kernel binder 驱动
kernel/drivers/staging/android/binder.h
kernel/drivers/staging/android/binder.c
# kernel ashmem 驱动
kernel/include/linux/ashmem.h
kernel/mm/ashmem.c
```

## 动态注册

### 接口端

我们先来说说动态注册。动态注册是在程序当中调用 Context（ContextImpl）的接口来注册的：

```java
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter)
```

其中一般自己要继承 BoradcastReceiver，然后实现里面的 onReceive 函数：

```java
// Context 是运行该 receiver 所在在的 context
// Intent 是所接收到的广播发出的 intent 
public abstract void onReceive(Context context, Intent intent);
```

然后那个 IntentFilter 里面可以设置要接收的广播，可以设置好多种类型，具体的自己去翻 api 文档。一般简单的可以直接设一个 ACTION。然后因为这个是需要在程序运行中拿 Context 去注册的，所以这种注册就有一个特点：**接收广播的进程已经运行起来了**。

接下来我们就去看看这个注册过程是怎么样的。首先调用的是 Context（Activity、Service 里均可以调用）的 registerReceiver ，这个其实是挂 ContextImpl 里面的同名函数的马甲：

```java
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }
    
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }
```

可以看出最后是调用 registerReceiverInternal 这个函数的。然后我们用的那个没指定权限（我们先不管啥权限之类的），还有 Handler。传过去的是 null：

``` java
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    // 默认使用的是当前注册进程的主线程来处理接收到的广播
                    scheduler = mMainThread.getHandler();
                }    
                // 如果 mPackageInfo(LoadApk) 不为空，去调用其中的方法取 IIntentReceiver
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                // 否则直接 new 一个新的出来
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            // 最后还是要去 AM 里面去注册
            return ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
        } catch (RemoteException e) {
            return null;
        }
    }
```

然后我们得先说 IIntentReceiver 这个东西。这个是保存广播最基本的数据了。这个是以 II 开头的，就知道这个实现了 Binder 接口支持 IPC 访问的。这个是用 aidl 写的，接口只有一个： 

```java
// 这个看注释就知道是接收到广播后触发处理的函数
/**
 * System private API for dispatching intent broadcasts.  This is given to the 
 * activity manager as part of registering for an intent broadcasts, and is
 * called when it receives intents.
 *
 * {@hide}
 */
oneway interface IIntentReceiver {
    void performReceive(in Intent intent, int resultCode, String data,
            in Bundle extras, boolean ordered, boolean sticky, int sendingUser);
}
```

然后 LoadApk 中的一个内部类的内部类(LoadedApk.ReceiverDispatcher.InnerReceiver)实现了这个接口：

```java
    static final class ReceiverDispatcher {
        
        // 这里是 Bn 端啦，注册的进程运行处理函数
        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;
            
            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }
            public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                // 这里我们先不分析处理过程，留到后面再说
                ... ...
            }
        }
        
... ...
        
        final IIntentReceiver.Stub mIIntentReceiver;
        final BroadcastReceiver mReceiver;
        final Context mContext;
        final Handler mActivityThread;
        final Instrumentation mInstrumentation;
        final boolean mRegistered;
        final IntentReceiverLeaked mLocation;
        RuntimeException mUnregisterLocation;
        boolean mForgotten;
        
... ...
        
        ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {
            if (activityThread == null) {
                throw new NullPointerException("Handler must not be null");
            }
            
            // new 这个 ReceiverDispatcher 的时候就会 new 一个 InnerReceiver
            mIIntentReceiver = new InnerReceiver(this, !registered);
            mReceiver = receiver;
            mContext = context;
            // 保存指定的处理广播的线程（Handler）
            mActivityThread = activityThread;
            mInstrumentation = instrumentation;
            mRegistered = registered;
            mLocation = new IntentReceiverLeaked(null);
            mLocation.fillInStackTrace();
        }
        
        void validate(Context context, Handler activityThread) {
            if (mContext != context) {
                throw new IllegalStateException(
                    "Receiver " + mReceiver +
                    " registered with differing Context (was " +
                    mContext + " now " + context + ")");
            }
            if (mActivityThread != activityThread) {
                throw new IllegalStateException(
                    "Receiver " + mReceiver +
                    " registered with differing handler (was " +
                    mActivityThread + " now " + activityThread + ")");
            }
        }

... ...

        IIntentReceiver getIIntentReceiver() {
            return mIIntentReceiver;
        }

... ...

    }
```

这个 LoadedApk 看名字，基本上就是和本应用程序包相关的东西（manifest 里面申明的那一堆东西基本上都在这里，当然扫描、解析是 PMS 干的）。然后它提供一个 IIntentReceiver 的 Bn 实现。它会保存调用者传递过来的 Handler，这个也就是说我们可以指定广播处理的线程（哪天有空把 Handler、Looper 也写下分析），如果我们没有指定的话，默认用注册进程的主线程（**所以如果要在广播接收中做一些耗时的操作，请开一个后台线程作为接收的处理线程**）。然后我们回到 ContextImpl 的注册函数那，有一个处理，如果 mPackageInfo 不为空，就去调用 LoadedApk 里的函数去取：

```java
    public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd = null;
            HashMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            // 动态注册传过来的是 true
            if (registered) { 
                // 其实就是内部把之前注册过的保存了起来，如果后面重复注册一样的，可以不用重复处理
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }   
            }           
            if (rd == null) {
                // 如果之前没有注册过的话，还是要创建一个新的
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        map = new HashMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                // 这个验证代码去看前面，就是检测下是不是当前运行的环境和线程而已
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            return rd.getIIntentReceiver();
        }
    }
```

然后后面就去 AMS 里面去注册了。其实 AMS 可以说是广播的中转站，应用程序通过 AMS 发送广播，然后 AMS 找到对应的接收器进行广播分发（咋感觉和消息分发差不多）。

### 服务端

来看下 AMS 中 registerReceiver ：

```java
    // caller: 客户端传过来标示客户进程的
    // receiver: 客户端准备好的接收广播的对象，这里传过来已经是 Bp 端了
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
        enforceNotIsolatedCaller("registerReceiver");
        int callingUid;
        int callingPid;
        // IPC 服务端多线程，接口实现照例加锁
        synchronized(this) {
            // 这里是去取注册调用者的进程信息，pid、uid 之类的
            ProcessRecord callerApp = null;
            if (caller != null) {
                callerApp = getRecordForAppLocked(caller);
                if (callerApp == null) {
                    throw new SecurityException(
                            "Unable to find app for caller " + caller
                            + " (pid=" + Binder.getCallingPid()
                            + ") when registering receiver " + receiver);
                }
                if (callerApp.info.uid != Process.SYSTEM_UID &&
                        !callerApp.pkgList.contains(callerPackage)) {
                    throw new SecurityException("Given caller package " + callerPackage
                            + " is not running in process " + callerApp);
                }
                callingUid = callerApp.info.uid;
                callingPid = callerApp.pid;
            } else {
                callerPackage = null;
                callingUid = Binder.getCallingUid();
                callingPid = Binder.getCallingPid();
            }

            // 把用户 Id 也取到了，android 中多用户的广播是分开的，能够避免一些公共广播发出来，
            // 把其他用户安装的应用也启动起来
            userId = this.handleIncomingUser(callingPid, callingUid, userId,
                    true, true, "registerReceiver", callerPackage);

            List allSticky = null;

            // 这个和 sticky 功能相关我们不管先
            // Look for any matching sticky broadcasts...
            Iterator actions = filter.actionsIterator();
            if (actions != null) {
                while (actions.hasNext()) {
                    String action = (String)actions.next();
                    allSticky = getStickiesLocked(action, filter, allSticky,
                            UserHandle.USER_ALL);
                    allSticky = getStickiesLocked(action, filter, allSticky,
                            UserHandle.getUserId(callingUid));
                }
            } else {
                allSticky = getStickiesLocked(null, filter, allSticky,
                        UserHandle.USER_ALL);
                allSticky = getStickiesLocked(null, filter, allSticky,
                        UserHandle.getUserId(callingUid));
            }

            // The first sticky in the list is returned directly back to
            // the client.
            Intent sticky = allSticky != null ? (Intent)allSticky.get(0) : null;

            if (DEBUG_BROADCAST) Slog.v(TAG, "Register receiver " + filter
                    + ": " + sticky);

            if (receiver == null) {
                return sticky;
            }

            // 这个根据传过来的 IIntentReceiver(receiver) 构造一个 ReceiverList
            // 这个其实一个数组，一个 receiver 可能可以对应多个 Intent（广播）
            ReceiverList rl
                = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } else if (rl.uid != callingUid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for uid " + callingUid
                        + " was previously registered for uid " + rl.uid);
            } else if (rl.pid != callingPid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for pid " + callingPid
                        + " was previously registered for pid " + rl.pid);
            } else if (rl.userId != userId) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for user " + userId
                        + " was previously registered for user " + rl.userId);
            }
            // 然后再根据上面的 ReceiverList 构造出一个 BroadcastFilter
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);
            rl.add(bf);
            if (!bf.debugCheck()) {
                Slog.w(TAG, "==> For Dynamic broadast");
            }
            // 这里是关键步骤了，把上面那个 BroadcastFilter add 到 IntentResolver 这个对象中。
            // 这个东西后面分发广播的时候会用到（其实分发的时候查询这个里面的数据）。
            mReceiverResolver.addFilter(bf);

            // sticky 相关的继续无视
            // Enqueue broadcasts for all existing stickies that match
            // this filter.
            if (allSticky != null) {
                ArrayList receivers = new ArrayList();
                receivers.add(bf);

                int N = allSticky.size();
                for (int i=0; i<N; i++) {
                    Intent intent = (Intent)allSticky.get(i);
                    BroadcastQueue queue = broadcastQueueForIntent(intent);
                    BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                            null, -1, -1, null, receivers, null, 0, null, null,
                            false, true, true, -1);
                    queue.enqueueParallelBroadcastLocked(r);
                    queue.scheduleBroadcastsLocked();
                }
            }

            return sticky;
        }
    }
```

一开始先获取注册调用者的进程记录（ProcessRecord，这个东西在 Binder 篇有说过，这里不多说）。然后广播有个 sticky 的功能，我们先不管这些花哨功能先。后面出现了一个 ReceiverList 的对象，我们来看看：

```java

```

未完待续 ... ...



