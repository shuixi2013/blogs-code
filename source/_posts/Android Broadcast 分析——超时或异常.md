title: Android Broadcast 分析——超时或异常
date: 2015-01-22 10:17:16
updated: 2017-02-07 21:47:53
categories: [Android Framework]
tags: [android]
---

上一篇（处理篇）最后留了一个问题：串行广播需要一个接着一个执行，如果其中某一个接收器处理的时候挂掉了，或者耗时很长还没处理完，是不是后面的接收器就无法处理了呢。我们在这里看一下 android 是不是已经想到这些了。我们先照例把相关代码位置啰嗦一下（4.2.2）：

```bash
# AM 广播相关代码
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/java/com/android/server/am/BroadcastQueue.java
frameworks/base/services/java/com/android/server/am/BroadcastRecord.java
```

## 超时处理

在 BroadcastQueue 执行串行广播的时候，会设置一个超时广播：

```java
    final void processNextBroadcast(boolean fromMsg) {
        synchronized(mService) {
            BroadcastRecord r;
... ...

            // 前面一点开始处理串行广播
            // 如果还没设置超时就设置一个
            if (! mPendingBroadcastTimeoutMessage) {
                // 当前时间+设置的超时等待时间
                long timeoutTime = r.receiverTime + mTimeoutPeriod;
                if (DEBUG_BROADCAST) Slog.v(TAG,
                        "Submitting BROADCAST_TIMEOUT_MSG ["
                        + mQueueName + "] for " + r + " at " + timeoutTime);
                setBroadcastTimeoutLocked(timeoutTime);
            }

... ...
        }
    }
```

然后我们来看下 setBroadcastTimeoutLocked 这个函数：

```java 
    final Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG, "Received BROADCAST_INTENT_MSG");
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: { 
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
            }
        }   
    };

    final void setBroadcastTimeoutLocked(long timeoutTime) {
        if (! mPendingBroadcastTimeoutMessage) {
            Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
            mHandler.sendMessageAtTime(msg, timeoutTime);
            mPendingBroadcastTimeoutMessage = true;
        }
    }
```

android framework 里面的很多超时都是通过发送延迟消息来实现的。其实感觉这个方法还是挺好用的，以后如果自己要实现一些超时功能可以借鉴一下。然后我们稍微去看下一个超时时间设置的是多少，在 AMS 初始化的时候设置的：

```java
// ================ ActivityManagerService.java ===================

    // How long we allow a receiver to run before giving up on it.
    static final int BROADCAST_FG_TIMEOUT = 10*1000; 
    static final int BROADCAST_BG_TIMEOUT = 60*1000; 

    private ActivityManagerService() {
        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());
    
        mFgBroadcastQueue = new BroadcastQueue(this, "foreground", BROADCAST_FG_TIMEOUT);
        mBgBroadcastQueue = new BroadcastQueue(this, "background", BROADCAST_BG_TIMEOUT);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;

... ...

    }

// ================ BroadcastQueue.java ===================

    BroadcastQueue(ActivityManagerService service, String name, long timeoutPeriod) {
        mService = service;
        mQueueName = name;
        mTimeoutPeriod = timeoutPeriod;
    }
```

可以看到前台的广播超时只有 10s，后台的广播超时有 60s。我们这里以后台广播为例，如果在串行广播中（并行广播不需要超时）一个接收器霸占了 60s 还没处理完（对 AMS 发送 finishReceiver），那么接下来会怎么样咧，我们看看超时处理：

```java
    // 从上面 Handler 调用的 fromMsg 是 true
    final void broadcastTimeoutLocked(boolean fromMsg) {
        // post 到 Handler 那的处理了，就可以标志关掉
        if (fromMsg) {
            mPendingBroadcastTimeoutMessage = false;
        }    

        // 如果没有串行广播就返回
        if (mOrderedBroadcasts.size() == 0) { 
            return;
        }    

        // 取当前时间和当前等待的接收器广播记录
        long now = SystemClock.uptimeMillis();
        BroadcastRecord r = mOrderedBroadcasts.get(0);
        if (fromMsg) {
            if (mService.mDidDexOpt) {
                // Delay timeouts until dexopt finishes.
                mService.mDidDexOpt = false;
                long timeoutTime = SystemClock.uptimeMillis() + mTimeoutPeriod;
                setBroadcastTimeoutLocked(timeoutTime);
                return;
            }    
            if (!mService.mProcessesReady) {
                // Only process broadcast timeouts if the system is ready. That way
                // PRE_BOOT_COMPLETED broadcasts can't timeout as they are intended
                // to do heavy lifting for system up.
                return;
            }    

            // 这里下面有注释，可能其他地方也会触发这个超时操作，
            // 判断下如果超时时间还没到，重新设一个进去，然后忽略这次
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            if (timeoutTime > now) {
                // We can observe premature timeouts because we do not cancel and reset the
                // broadcast timeout message after each receiver finishes.  Instead, we set up
                // an initial timeout then kick it down the road a little further as needed
                // when it expires.
                if (DEBUG_BROADCAST) Slog.v(TAG,
                        "Premature timeout ["
                        + mQueueName + "] @ " + now + ": resetting BROADCAST_TIMEOUT_MSG for "
                        + timeoutTime);
                setBroadcastTimeoutLocked(timeoutTime);
                return;
            }
        }

        r.receiverTime = now;
        r.anrCount++;

        // 如果当前的广播没接收器需要处理了，也返回
        // Current receiver has passed its expiration date.
        if (r.nextReceiver <= 0) {
            Slog.w(TAG, "Timeout on receiver with nextReceiver <= 0");
            return;
        }

        ProcessRecord app = null;
        String anrMessage = null;

        // 这里取当前接收器的进程信息
        Object curReceiver = r.receivers.get(r.nextReceiver-1);
        Slog.w(TAG, "Receiver during timeout: " + curReceiver);
        logBroadcastReceiverDiscardLocked(r);
        // 这个是动态注册的接收器的进程信息（动态注册的接收器也可能串行处理的）
        if (curReceiver instanceof BroadcastFilter) {
            BroadcastFilter bf = (BroadcastFilter)curReceiver;
            if (bf.receiverList.pid != 0
                    && bf.receiverList.pid != ActivityManagerService.MY_PID) {
                synchronized (mService.mPidsSelfLocked) {
                    app = mService.mPidsSelfLocked.get(
                            bf.receiverList.pid);
                }
            }
        } else {
            // 这是静态注册的接收器的进程信息
            app = r.curApp;
        }

        if (app != null) {
            anrMessage = "Broadcast of " + r.intent.toString();
        }

        // 如果当前接收器还在等待目标进程的启动，不用等了 -_-||
        if (mPendingBroadcast == r) {
            mPendingBroadcast = null;
        }

        // 经过处理篇的分析下面2个函数的调用就相当于认为当前的接收器已经处理了，
        // 让 BroadcastQueue 接着处理一下接收器
        // Move on to the next receiver.
        finishReceiverLocked(r, r.resultCode, r.resultData,
                r.resultExtras, r.resultAbort, true);
        scheduleBroadcastsLocked();

        // 前面如果能够获取接收器的进程信息，
        // 给接收器所在进程发一个 ANR 消息（看样子后台处理也有可能有 ANR 的）
        if (anrMessage != null) {
            // Post the ANR to the handler since we do not want to process ANRs while
            // potentially holding our lock.
            mHandler.post(new AppNotResponding(app, anrMessage));
        }
    }
```

看上面的处理，如果当前接收器发生了超时（超过 60s 没给 AMS 发送 finishReceiver 消息），那么 AMS（BroadcastQueue）就会直接对当前的接收器进行处理完成操作，以便激发 BroadcastQueue 把广播分发给下一个等待处理的接收器（同时给接收器进程发 ANR 消息）。所以前面处理篇中担心的在串行广播中如果有某一个接收器搅屎，长时间不处理完广播，广播处理是不是就卡在那了是多余的。因为 AMS 会“等得不耐烦”直接不等，直接让下一个接收器处理（就算需要启动进程，60s 启动一个就没什么事）。

从上面可以看得出，某些靠静态注册广播启动进程的功能，有些时候发现某些事件发生了（例如网络可用），但是自己的代码好像很久还没跑，其实并不是广播没发出来，而已有可能前面有些应用长时间“占着茅坑”而已。然后自己程序的接收器优先级又低一点，就悲剧吧。

## 异常处理

一开始接收器异常（例如说挂掉），我以为可以一起超时用处理的，但是 android 没这么干。其实也是可以的，但是有点不好，因为如果一起用超时来处理的，需要等待 60s 广播才能接着处理，可能接收器进程早就挂掉了。所以 android 单独考虑了这种情况。我们来一起看下这种情况的处理。

首先一个进程由 AMS startProcess 启动起来后，ActivityThread 会调用 AMS 的 attachApplication（我们这里跳过马甲，直接看真正做事的函数）：

```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

... ...

        String processName = app.processName;
        try {
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            app.resetPackageList();
            startProcessLocked(app, "link fail", processName);
            return false;
        }

... ...

        return true;
    } 
```

binder 死亡通知篇里面知道对 IApplicationThread 的 Bp（Bn 是接收器进程）注册一个死亡通知回调，当这个 Bp 的 Bn 所在进程（接收器进程啦）挂掉的时候，这个 AMS 中的这个回调会被调用。我们先来看看这个回调是啥：

```java
    private final class AppDeathRecipient implements IBinder.DeathRecipient {
        final ProcessRecord mApp;
        final int mPid;
        final IApplicationThread mAppThread;
        
        // 构造的时候保存了崩溃进程的 pid、IApplicationThread 接口
        AppDeathRecipient(ProcessRecord app, int pid,
                IApplicationThread thread) {
            if (localLOGV) Slog.v(
                TAG, "New death recipient " + this
                + " for thread " + thread.asBinder());
            mApp = app;
            mPid = pid; 
            mAppThread = thread;
        }   
            
        public void binderDied() {
            if (localLOGV) Slog.v(
                TAG, "Death received in " + this
                + " for thread " + mAppThread.asBinder());
            synchronized(ActivityManagerService.this) {
                appDiedLocked(mApp, mPid, mAppThread);
            }
        }   
    }
```

前面 new AppDeathRecipient 的时候把 attachApplication 进程的 pid 和 IBinder 接口保存了一下，然后去看看 appDiedLocked 做了什么：

```java
// 如果一个进程挂了 AMS 要做不少工作的，这里我们只选和广播相关的
// 我是怎么挑选出这些函数的咧，加点 log 把函数调用堆栈打出来就行了 ... ...

    // 从个这个函数开始
    final void appDiedLocked(ProcessRecord app, int pid,
            IApplicationThread thread) {

... ...

        // Clean up already done if the process has been re-started.
        if (app.pid == pid && app.thread != null &&
                app.thread.asBinder() == thread.asBinder()) {
            if (!app.killedBackground) {
                Slog.i(TAG, "Process " + app.processName + " (pid " + pid
                        + ") has died.");
            }
            EventLog.writeEvent(EventLogTags.AM_PROC_DIED, app.userId, app.pid, app.processName);
            if (DEBUG_CLEANUP) Slog.v(
                TAG, "Dying app: " + app + ", pid: " + pid
                + ", thread: " + thread.asBinder());
            boolean doLowMem = app.instrumentationClass == null;
            // 然后是这里
            handleAppDiedLocked(app, false, true);

... ...

        } else if (app.pid != pid) {
            // A new process has already been started.
            Slog.i(TAG, "Process " + app.processName + " (pid " + pid
                    + ") has died and restarted (pid " + app.pid + ").");
            EventLog.writeEvent(EventLogTags.AM_PROC_DIED, app.userId, app.pid, app.processName);
        } else if (DEBUG_PROCESSES) {
            Slog.d(TAG, "Received spurious death notification for thread "
                    + thread.asBinder());
        }
    }

            
    /*
     * Main function for removing an existing process from the activity manager
     * as a result of that process going away.  Clears out all connections
     * to the process.
     */
    private final void handleAppDiedLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart) {
        // 再然后是这里，清理程序的一些记录（里面就有广播记录）
        cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1);
        if (!restarting) {
            mLruProcesses.remove(app);
        }  

... ...

    }

    /*
     * Main code for cleaning up a process when it has gone away.  This is
     * called both as a result of the process dying, or directly when stopping
     * a process when running in single process mode.
     */ 
    private final void cleanUpApplicationRecordLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart, int index) {
        if (index >= 0) {
            mLruProcesses.remove(index);
        }  

... ...

        // 接着是这里，名字很形象，跳过当前（这个进程的）的接收器
        skipCurrentReceiverLocked(app);

... ...

    }

    void skipCurrentReceiverLocked(ProcessRecord app) {
        // 最后还是调用 BroadcastQueue 来处理的
        for (BroadcastQueue queue : mBroadcastQueues) { 
            queue.skipCurrentReceiverLocked(app);
        }
    }
```

好，到 BroadcastQueue 这里我们停一下（AMS 中和广播相关的操作，最后都是由 BroadcastQueue 处理的）。然后我们仔细来看看 BroadcastQueue 中 skipCurrentReceiverLocked 函数（为什么要仔细，后面会知道）：

```java
    public void skipCurrentReceiverLocked(ProcessRecord app) {
        boolean reschedule = false;
        // 取异常进程当前正在执行的广播记录
        BroadcastRecord r = app.curReceiver;
        if (r != null ) {
            // The current broadcast is waiting for this app's receiver
            // to be finished.  Looks like that's not going to happen, so
            // let the broadcast continue.
            logBroadcastReceiverDiscardLocked(r);
            // 把当前广播记录的状态设置为结束状态
            finishReceiverLocked(r, r.resultCode, r.resultData,
                    r.resultExtras, r.resultAbort, true);
            reschedule = true;
        }

        // 上面那个是当前正在执行的，这里是等待这个进程启动的
        // （不巧这个进程挂了）
        r = mPendingBroadcast;
        if (r != null && r.curApp == app) {
            if (DEBUG_BROADCAST) Slog.v(TAG,
                    "[" + mQueueName + "] skip & discard pending app " + r);
            logBroadcastReceiverDiscardLocked(r);
            // 同样设置一下结束状态
            finishReceiverLocked(r, r.resultCode, r.resultData,
                    r.resultExtras, r.resultAbort, true);
            reschedule = true;
        }
        // 然后如果上面任何一个有接收器的话，
        // 就要让 BroadcastQueue 接着去分发广播给下一个接收器
        if (reschedule) {
            scheduleBroadcastsLocked();
        }
    }
```

上面的代码咋一看是 android 帮我们考虑到了接收器进程挂了的情况，让后面的接收器能更快速的收到广播。但是回到 AMS skipCurrentReceiverLocked 的仔细琢磨下那个 BroadcastQueue 队列的循环，就发现有点点不对劲的地方： 这个循环是，循环然 BroadcastQueue 执行 skipCurrentReceiverLocked，然后再执行看一下 BroadcastQueue 的 skipCurrentReceiverLocked，就会发现前面一部分是去取崩溃进程的当前的广播记录，那么问题来了，这里完全没判断这个 BroadcastRecord 记录是属于哪个广播队列的（AMS 里面有分前台广播队列和后台广播队列的），直接拿去给 finishReceiverLocked 处理了，然后 finishReceiverLocked 里有这么一句：

```java
    public boolean finishReceiverLocked(BroadcastRecord r, int resultCode,
            String resultData, Bundle resultExtras, boolean resultAbort,
            boolean explicit) {            
... ...
        r.receiver = null;    
        r.intent.setComponent(null);
        // 把这个广播所在进程当前处理的广播记录设置为 null
        if (r.curApp != null) {        
            r.curApp.curReceiver = null;   
        }
... ...
    }
```

这会有什么问题咧。让我来举一个例子，其实就拿最开始背景里面那个 `BOOT_COMPLETED` 广播来说。很多应用都会静态注册开机广播的，假设有一个进程的接收器在处理开机广播的时候挂了。那么 AMS 执行到 skipCurrentReceiverLocked 那里。AMS 在构造函数那里初始化 mBroadcastQueues 的时候前台的广播队列在前面（0位置），后台广播在后面（1位置，代码前面说超时设置那里有）。那这里就会先执行前台广播队列的 skipCurrentReceiverLocked 处理。但是 BraodcastQueue 里面的 skipCurrentReceiverLocked 并没有判断挂掉进程当前的这个 BroadcastRecord 是属于谁的（这里是属于后台队列的），所以在前面队列中就处理起来了。然后调用了前台的 scheduleBroadcastsLocked（进而调用 processNextBroadcast），但是前台广播队列并没有正在处理的串行广播（假设没有），所以就啥事也没做。这里前台广播队列虽然啥事也没做，但是却把崩溃进程正在处理的 BroadcastRecord 扔给 finishReceiverLocked 设置结束状态去了，所以这里 app.curReceiver = null 。然后等循环到后台广播队列的时候，由于 app.curReceiver 被前面前台误设置为 null ， reschedule 这个标志就无法被设置为 true，所以无法调用后台广播队列的 scheduleBroadcastsLocked 接着把 BOOT_COMPLETED 的广播分发给后面的接收器处理。后面的接收器就收不到广播了（那个闹钟不响的问题就是前面有个应用捣乱导致闹钟接收不到开机广播了）。


所以说 android 在接收器异常处理这里有个小 bug，应该要在 BroadcastQueue skipCurrentReceiverLocked 那里判断下崩溃进程当前的这个 BroadcastRecord 是不是属于自己队列里面的，如果是才处理，否则就不处理。可以这样修复一下：

```java
    public void skipCurrentReceiverLocked(ProcessRecord app) {
        boolean reschedule = false;    
        BroadcastRecord r = app.curReceiver;
        // 这里判断下这个广播记录是否是属于自己队列里面的，不要把别人队列里面的处理了
        //if (r != null ) {
        if (r != null && this.equals(r.queue)) {
            // The current broadcast is waiting for this app's receiver
            // to be finished.  Looks like that's not going to happen, so
            // let the broadcast continue. 
            logBroadcastReceiverDiscardLocked(r);
            finishReceiverLocked(r, r.resultCode, r.resultData,
                    r.resultExtras, r.resultAbort, true);
            reschedule = true;
        }

... ...
    }
```

如果没这个 bug 的话，本来有接收器挂掉了，后面等待的接收器是能马上接着处理的。但是实际上，现在只要前面有接收器搅屎，歇菜了就悲剧了（其实也不一定，如果马上又有一个后台广播发过来，就又能激发上次的串行广播接着分发，因为接收器的记录还在 BroadcastQueue 里面排着队，所以这个 bug 是随机性的，查起来更加悲剧）。我看了下 4.2.2 和 4.4 这里的代码，都还没修复（没人给 google 反馈这个 bug 么），不知道 5.0 修复了没。

## 解决问题

到了这里就可以解释情况开篇（注册篇）提到的那个我们的平板上开机闹钟为什么不响了。原因是这样的，我们自己有个应用（appA），也静态注册了开机广播，由于一开始不知道可以系统应用可以设置接收器优先级，所以都没设置，结果 PMS 扫描的结果是 appA 比闹钟靠前。所以开机广播就 appA 先处理，结果写这个 appA 的哥们在处理完之后直接在 onReceive 中写了一个 System.exit(0) ！！

我去，这就直接强制终于了接收器的进程，并且 log 里面报没有任何异常，只有一句 process xx has died （因为这个不是因为错误退出的，而是自己调用 System.exit 退出的）。然后上面分析了，正好 android 这里有个小 bug 就导致后面的接收器收不到开机广播了，所以闹钟没办法在 AlaramMananger 里面设置闹钟，所以设备虽然被 kernel 的 RTC 唤醒开机了，但是闹钟却没有响。而且这个问题还是随机的，前面说了如果后面正好来了一个后台广播，那么后面的接收器又能收到了。SQA 测试的时候开着 wifi 的情况下闹钟可以响，正好网络变得可以访问会发一个后台广播 -_-||。

所以这里根本的解决办法是修改 framework，改正这个 bug。但是这样要 OTA 升级系统才行，这边头头最头痛发放 OTA。所以还是有一个办法就是改应用：

1. 要么把 appA 给改了，onReceive 处理完之后不要强制退出进程
2. 但是你改得了一个，却防不住全部的，指不定那个应用也抽风一下（现在应用注册开机广播和吃白菜一样），由于我们的应用是系统应用，所以我们可以改闹钟接收器的优先级，把优先级调高点就比较保险了（最后还是采用这个方案的）。

## 总结

这里正好趁查这个问题的机会把 Broadcast 流程摸一下，之前还有查过一个关不了机的问题的，是关机广播。那个时候就稍微看了下 AMS 中 Broadcast 的处理，但是那个时候没坚持记录一下，结果查这个问题的时候又要从头开始。这里记录一下，理解了 Broadcast 的原理就能更好的应用它。当然这里其实还忽略了 Broadcast 的别的一些功能，例如那个 Sticky 功能，以后有时间再补一下了。 


