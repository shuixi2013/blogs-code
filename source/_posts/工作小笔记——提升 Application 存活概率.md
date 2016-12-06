title: 工作小笔记——提升 Application 存活概率
date: 2016-12-06 16:37:16
updated: 2016-12-06 16:37:16
categories: [Android Framework]
tags: [android]
---

之前由于工作上的原因需要提升应用切换到后台后存活的概率，也就是切换到后台后，不被系统杀掉的概率。这么做的目的是为了能够接收到某些 broadcast，因为某些国产系统（miui，flyme 等），限制了程序自动启动的能力。在这些系统上如果你的程序没有运行，那么 broadcast 将无法唤醒你的程序，从而无法有效的收到推送。


## 通过子进程保护主进程

最初的想法是： 通过启动一个 native process 来保护 app，定时的去唤醒 java 层的一个 service，从而达到启动 app 的目的。这个想法的最根本前提就是：app 被杀死，native process 不受影响。本来的猜想是可以达到这个目的的。但是最后发现，其实行不通。下面说的源码是 AOSP 上的 android-6.0.1_r1 这个 tag 的，设备是我的 google nexus 6p。


### 1. pid、uid 的概念

 * **pid**: 是进程的一个编号，系统通过 pid 能定位到一个具体的进程，这个是 linux 上的。
 * **uid**: 是 android 提出来的，用于隔离不用应用之类的数据，保护数据安全性。一旦 apk 安装完成，其 uid 就确定了，除非卸载重新安装，否则 uid 不会改变。在 AndroidManifest.xml 中设置 SharedUserId 能让不同的 apk 相互访问其资源，但是必须要有相同的签名，否则会被系统拒绝。


也就说，<font color="#ff0000">一个 apk 下不管运行多少个进程，都是属于同一个 uid 的，不管进程是 java 代码启动的，还是 native 代码启动的</font>。一旦一个进程启动，系统会在 '/proc' 目录下创建一个 'uid_xx' 的文件夹。例如写一个 demo apk 安装完成后，系统分配的 uid 是 10265 那么会创建 '/proc/uid_10265'。然后 app 进程启动，会分配 pid，系统会在 '/proc/uid_10265' 下创建 'pid_xx' 的文件夹，记录这个进程的状态信息，例如 demo apk 的 pid 是 16553 的话，运行 demo app 后，会创建 '/proc/uid_10265/pid_16553' 

在这个文件夹下系统会创建一个 cgroup.procs 的文件，把从这个 16553 启动的所有进程记录在这个文件内。我尝试过的启动方式有：

a. java 层: Runtime.getRuntime().exec("command")

b. native 层:
  (1). 通过 shell 启动
  (2). fork
  (3). popen

系统都会在对应的 cgroup.procs 中记录下属于这个 uid 的 pid（可以在 root 的手机上 cat 这个文件来查看）。这点很重要，**就是因为系统能记录下同一个 uid 下所有进程的 pid**，才导致这个方案行不通的。


### 2. 进程被杀的情况

下面是我这段时间研究的，可能会有遗漏：

**a. 进入后台，被系统回收**。

这个又分为几种情况：
  **(1)**. app 没有 service，没有 activity （activity 被 destory，按 back 键退出）： 这种情况 app 进入后台，am 会认为这是一个 empty 的进程，在 oom 的类型中会被分配到 cached 的列表中（adb shell dumpsys meminfo 查看）：

<pre>
   179596 kB: Cached
               51437 kB: com.estrongs.android.pop (pid 19069)
               28869 kB: com.google.android.gms (pid 9895)
               13837 kB: com.kk.demo.autolaunch (pid 18907)
               13429 kB: com.google.android.apps.docs (pid 18007)
               12676 kB: com.tencent.mobileqq:web (pid 18116)
               12114 kB: com.ss.android.article.news (pid 19233)
                9819 kB: com.baidu.carlife (pid 19022)
                7304 kB: com.myzaker.ZAKER_Phone (pid 19106)
                6019 kB: com.kk.poem:push (pid 18943)
                5021 kB: com.kk.kkyuwen:push (pid 18597)
                4403 kB: com.google.android.apps.cloudprint (pid 19275)
                4103 kB: com.zeptolab.ctrm.free.google (pid 19191)
                4008 kB: com.google.android.calendar (pid 3374)
                2686 kB: com.android.musicfx (pid 12568)
                2657 kB: com.google.android.partnersetup (pid 19143)
                1214 kB: com.google.android.gms.feedback (pid 21849)
</pre>

系统会有一个缓存这类进程的数量，如果超过这个数量了，am 就开杀了（framework/base/services/core/java/com/android/server/am/ActivityManagerService.java: updateOomAdjLocked）：

```java
                // Count the number of process types.
                switch (app.curProcState) {
                    case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY:
                    case ActivityManager.PROCESS_STATE_CACHED_ACTIVITY_CLIENT:
                        mNumCachedHiddenProcs++;
                        numCached++;
                        if (numCached > cachedProcessLimit) {
                            app.kill("cached #" + numCached, true);
                        }     
                        break;
                    case ActivityManager.PROCESS_STATE_CACHED_EMPTY:
                        if (numEmpty > ProcessList.TRIM_EMPTY_APPS
                                && app.lastActivityTime < oldTime) {
                            app.kill("empty for "
                                    + ((oldTime + ProcessList.MAX_EMPTY_TIME - app.lastActivityTime)
                                    / 1000) + "s", true);
                        } else {
                            numEmpty++;
                            if (numEmpty > emptyProcessLimit) {
                                app.kill("empty #" + numEmpty, true);
                            }     
                        }     
                        break;
                    default:
                        mNumNonCachedProcs++;
                        break;
                }     
```

app.kill 是 am 中 ProcessRecord 的一个函数，会调用 Process.killProcessGroup，接下的流程就是： 

<pre>
java: Process.killProcessGroup (framework/base/services/core/java/com/android/server/am/ProcessRecord.java)
            |
            |
           \|/
jni:  android_os_Process_killProcessGroup (framework/base/core/jni/android_util_Process.cpp)
            |
            |
           \|/
so:   killProcessGroup (system/core/libprocessgroup/processgroup.cpp)
            |
            |
           \|/
     killProcessGroupOnce
            |
            |
           \|/
unix:     kill
</pre>

这里流程简单的说就是去 '/proc/uid_xx/pid_xx/cgroup.procs' 中读取同一个 uid 下的 pid，然后循环调用 unix 的 kill 函数去终止进程（具体代码不贴了，感兴趣的可以自己去看源码）。**所以 am 中只要调用了 ProcessRecord.kill 函数去处理的，那么同一个 uid 下的所有进程都会被杀死**。这种 empty 进程进入后台也很容易被杀掉。只要用户多打开几个其他的 app，就会把它挤掉了。然后上面的 am 中的 updateOomAdjLocked 是在 trimApplications 中调用的。trimApplications 会在 am 执行 activityStopped, unregisterReceiver, finishReceiver 的时候调用。也就是说在这些时间点会执行内存收回检测。这里我也没太具体研究，如果后面有兄弟有兴趣研究清楚了希望能分享一下（am 算是 android 中比较复杂的一个 services，要全部搞明白太耗时了，我在研究中主要看个大概，然后猜测，验证一下）。


  **(2)**. app 有 service，有 activity（activity 仅仅是被 stop，按 home 退出，而不是按 back）：这种情况 app 进入后台，会被分到 Services 列表，其中又分为 A Services 和 B Services，A Services 是最近使用的，B
 Services 是之前的。A 和 B 的具体怎么区别，怎么移动的，我暂时没研究，但是有一点，am 优先回收 B 里面的。Services 列表中的进程能存活很长时间，好像不受数量限制，或者说数量比 Cached 列表的多很多，只有在内存不足的情况下才会杀这里的进程（上面的 Cached 列表，就算内存足够，但是达到数量限制也一样会被杀）：

<pre>
   426671 kB: A Services
              118018 kB: com.kk.demo.autolaunch:PushService (pid 18921)
               17829 kB: com.android.vending (pid 4007)
               16766 kB: com.kk.demo.mingming2:Service2 (pid 19334)
               16743 kB: com.kk.demo.mingming2:Service1 (pid 18955)
               16701 kB: com.kk.demo.mingming3:Service4 (pid 18742)
               16670 kB: com.kk.demo.mingming2:Service3 (pid 17884)
               16668 kB: com.kk.demo.mingming3:Service9 (pid 17025)
               16664 kB: com.kk.demo.mingming:Service7 (pid 17654)
               16664 kB: com.kk.demo.mingming2:Service10 (pid 17304)
               16661 kB: com.kk.demo.mingming2:Service7 (pid 17759)
               16657 kB: com.kk.demo.mingming:Service1 (pid 17558)
               16657 kB: com.kk.demo.mingming:Service4 (pid 16917)
               16656 kB: com.kk.demo.mingming:Service10 (pid 18535)
               16651 kB: com.kk.demo.mingming4:Service3 (pid 18026)
               15764 kB: com.taobao.taobao:channel (pid 9847)
               15492 kB: sina.mobile.tianqitong (pid 23055)
               11663 kB: com.ss.android.article.news:pushservice (pid 11235)
                9795 kB: com.youdao.dict (pid 11222)
                8959 kB: com.tencent.qqpim (pid 27109)
                7814 kB: com.baidu.tieba_mini:remote (pid 8835)
                7093 kB: com.sankuai.meituan:pushservice (pid 28738)
                5481 kB: com.kk.dict:push (pid 9656)
                3363 kB: com.icbc:pushservice (pid 8321)
                3332 kB: com.happyelements.AndroidAnimal (pid 14926)
                1910 kB: com.netease.pomelo.push.l.messageservice_V2 (pid 11371)

683007 kB: B Services
               37733 kB: com.taobao.taobao (pid 10233 / activities)
               16672 kB: com.kk.demo.mingming:Service6 (pid 16490)
               16670 kB: com.kk.demo.mingming2:Service8 (pid 16812)
               16663 kB: com.kk.demo.mingming3:Service6 (pid 15985)
               16661 kB: com.kk.demo.mingming2:Service5 (pid 16583)
               16654 kB: com.kk.demo.mingming7:Service6 (pid 15878)
               16651 kB: com.kk.demo.mingming2:Service6 (pid 16696)
               16649 kB: com.kk.demo.mingming7:Service10 (pid 15924)
               16648 kB: com.kk.demo.mingming4:Service2 (pid 16286)
               16647 kB: com.kk.demo.mingming4:Service1 (pid 16089)
               16647 kB: com.kk.demo.mingming7:Service9 (pid 15916)
               16647 kB: com.kk.demo.mingming7:Service8 (pid 15904)
               16645 kB: com.kk.demo.mingming7:Service5 (pid 15865)
               16644 kB: com.kk.demo.mingming7:Service4 (pid 15850)
               16643 kB: com.kk.demo.mingming4:Service7 (pid 16393)
               16643 kB: com.kk.demo.mingming7:Service1 (pid 15808)
               16640 kB: com.kk.demo.mingming7:Service3 (pid 15838)
               16640 kB: com.kk.demo.mingming7:Service2 (pid 15824)
               16636 kB: com.kk.demo.mingming4:Service5 (pid 15557)
               16635 kB: com.kk.demo.mingming4:Service8 (pid 15587)
               16634 kB: com.kk.demo.mingming4:Service9 (pid 16187)
               16633 kB: com.kk.demo.mingming4:Service10 (pid 15715)
               16629 kB: com.kk.demo.mingming7:Service7 (pid 15890)
               16623 kB: com.kk.demo.mingming4:Service4 (pid 15608)
               16471 kB: com.kk.demo.mingming5:Service6 (pid 29587)
               16471 kB: com.kk.demo.mingming5:Service1 (pid 29522)
               16467 kB: com.kk.demo.mingming5:Service9 (pid 29626)
               16462 kB: com.kk.demo.mingming5:Service4 (pid 29561)
               16460 kB: com.kk.demo.mingming5:Service10 (pid 29638)
               16458 kB: com.kk.demo.mingming5:Service7 (pid 29600)
               16456 kB: com.kk.demo.mingming5:Service8 (pid 29613)
               16454 kB: com.kk.demo.mingming5:Service3 (pid 29548)
               16450 kB: com.kk.demo.mingming5:Service5 (pid 29574)
               16448 kB: com.kk.demo.mingming5:Service2 (pid 29535)
               15964 kB: com.zhihu.android (pid 27639)
               15913 kB: com.baidu.tieba_mini (pid 10575 / activities)
               15024 kB: cmb.pb:push (pid 30400)
                8526 kB: com.sinovatech.unicom.ui (pid 10886)
                7347 kB: com.kk.kkyuwen (pid 18196)
                6145 kB: com.MobileTicket (pid 32691)
                5589 kB: com.happyelements.AndroidAnimal:unicomuptsrv (pid 32569)
                5382 kB: android.process.media (pid 26431)
                3622 kB: com.icbc (pid 7146)
                3307 kB: ctrip.android.view:pushsdk.v1 (pid 9007)
                2381 kB: com.sinovatech.unicom.ui:remote (pid 4448)
                2206 kB: com.huawei.sarcontrolservice (pid 30062)
                1992 kB: ctrip.android.view:ctripbuprocess (pid 9080)
                1879 kB: com.qualcomm.display (pid 30161)
                1848 kB: com.happyelements.AndroidAnimal:egameCore (pid 15443)
                 698 kB: com.qualcomm.qcrilmsgtunnel (pid 12863)
</pre>

可以看到这里缓存的后台进程比 Cached 列表多了许多，然后能分配的内存也更多。这里我发现一些应用很贼，例如说淘宝（com.taobao.taobao (pid 10233 / activities)）和百度贴吧（com.baidu.tieba_mini (pid 10575 / activities)）。他们都是带有 activities 标志的，这个表示这些进程还留存有 activity。这些应用我都是按 back 键退出的（个人没有按 home 退出的习惯），他们应该是在主界面拦截了 back 键，改成了只是把 task 移动到后台了而已，并没有真正的执行 activity 的 destroy 操作，activity 仅仅是 onStop 了而已。**所以他们的主进程被分配到了 Services 列表，在后台能够更长时间的存活，提升下次用户进入主界面的速度。这种做法虽然可恶 ... 但是可能我们可以借鉴**。

Services 列表我没具体定位到在那里激发杀进程的（代码不太好找），但是我试验出，在内存比较紧张的情况下，am 就开始杀占用内存大的进程。遗憾的是杀死进程的同时也是会去 cgroup.procs 那里走一遍，把同属 uid 下的一起干掉了。


**b. app 自杀**
app 自己调用 Process.killProcess(Process.mypid()) 或者 System.exit(0) 退出自杀（在前面可以设置一个时间很短的 alaram 重新启动自己）。Process.killProcess 最后是通过 jni 调用 unix 的 kill 发送 SIGNAL_KILL 给进程，这种情况系统也会去把这个进程下保存的 uid 进程杀掉。具体代码我没找到，但是实验出来的结果是这样，这个和在 shell 中调用 kill -9 是一样的效果。


**c. 各种内存清理**

  **(1)**. 第三方软件（例如 xx内存清理），调用 am 的 killBackgroundProcesses api：

<pre>
 killPackageProcessesLocked
            |
            |
           \|/
 removeProcessLocked
            |
            |
           \|/
         app.kill
</pre>

看到 ProcessRecord.kill 就知道我们启动的 nativie 进程也会被杀掉了。


  **(2)**. 长按 home 键，调用出最近任务列表，删除任务（全部删除）： 原生系统是通过 SystemUI 调用 am 的 removeTask 接口去处理的：

<pre>
        removeTask
            |
            |
           \|/
  removeTaskByIdLocked 
            |
            |
           \|/
  cleanUpRemovedTaskLocked
            |
            |
           \|/
   ProcessRecord.kill
</pre>

还是和上面的一样。


  **(3)**. 在系统设置里面“强制终止”： 这个是调用 am 中的 forceStopPackage 去处理的：

<pre>
    forceStopPackageLocked
            |
            |
           \|/
   killPackageProcessesLocked
</pre>


到这里就包含了上面 killBackgroundProcesses 的处理了，所以 native 进程也会被杀，而且这个函数威力巨大，还会清除 app 设置过的 alaram，具体我以前的博客有记录，感兴趣的可以看一下： [forceStopPackage 的副作用](http://light3moon.com/2015/01/28/forceStopPackage 的副作用 "forceStopPackage 的副作用")


经过上面一系列的尝试和翻源代码，得出结论：在 android 上通过正常手段（非 root），由于 uid 的存在，无法避免系统杀死程序的所有进程，所以通过偷偷摸摸起一个进程的方式来保护 appliction 是行不通的。但是可以启动一个轻量级别的 service（在子进程中），来保护主进程，这样在一定程度上能提升后台 application 的存活率。


## 通过前台服务提升存活率


后来发现某些应用的进程很坚挺，很难被系统清除掉，后来发现方式是：**通过 service 绑定一个 ongoing（不可清除） 的 notification 来使 service 变成 foreground service(fg-service) 来保护自己的进程尽量不被系统杀掉**。

### fg-service 

fg-service 的好处有下面几个：

**1**. 在 oom 中回收等级更加低（framework/base/services/core/java/com/android/server/am/ProcessList.java）：

    // This is a process only hosting components that are perceptible to the
    // user, and we really want to avoid killing them, but they are not
    // immediately visible. An example is background music playback.
    static final int PERCEPTIBLE_APP_ADJ = 2;

数值越低，就越不容易被回收，am 是根据从高到低回收的。


**2**. 阻挡某些清理函数杀掉自己：

  **a**. 第三方清理软件调用 killBackgroundProcesses 无法杀死 fg-service（framework/base/services/core/java/com/android/server/am/ActivityManagerService.java）：

```java
    @Override
    public void killBackgroundProcesses(final String packageName, int userId) {
        if (checkCallingPermission(android.Manifest.permission.KILL_BACKGROUND_PROCESSES)
                != PackageManager.PERMISSION_GRANTED &&
                checkCallingPermission(android.Manifest.permission.RESTART_PACKAGES)
                        != PackageManager.PERMISSION_GRANTED) {
            String msg = "Permission Denial: killBackgroundProcesses() from pid="
                    + Binder.getCallingPid()
                    + ", uid=" + Binder.getCallingUid()
                    + " requires " + android.Manifest.permission.KILL_BACKGROUND_PROCESSES;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }   
            
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, true, ALLOW_FULL_ONLY, "killBackgroundProcesses", null);
        long callingId = Binder.clearCallingIdentity();
        try {
            IPackageManager pm = AppGlobals.getPackageManager();
            synchronized(this) {
                int appId = -1;
                try {
                    appId = UserHandle.getAppId(pm.getPackageUid(packageName, 0));
                } catch (RemoteException e) {
                }
                if (appId == -1) {
                    Slog.w(TAG, "Invalid packageName: " + packageName);
                    return;
                }   
                // killBackgroundProcesses 只允许杀 A service 以上的进程
                killPackageProcessesLocked(packageName, appId, userId,
                        ProcessList.SERVICE_ADJ, false, true, true, false, "kill background");
            }       
        } finally {     
            Binder.restoreCallingIdentity(callingId);
        }
    }


    private final boolean killPackageProcessesLocked(String packageName, int appId,
            int userId, int minOomAdj, boolean callerWillRestart, boolean allowRestart,
            boolean doit, boolean evenPersistent, String reason) {
        ArrayList<ProcessRecord> procs = new ArrayList<>();
        
        // Remove all processes this package may have touched: all with the
        // same UID (except for the system or root user), and all whose name
        // matches the package name.
        final int NP = mProcessNames.getMap().size();
        for (int ip=0; ip<NP; ip++) {
            SparseArray<ProcessRecord> apps = mProcessNames.getMap().valueAt(ip);
            final int NA = apps.size();
            for (int ia=0; ia<NA; ia++) {
                ProcessRecord app = apps.valueAt(ia);
                if (app.persistent && !evenPersistent) {
                    // we don't kill persistent processes
                    continue;
                }
                if (app.removed) {
                    if (doit) {
                        procs.add(app);
                    }
                    continue;
                }   
                    
                // Skip process if it doesn't meet our oom adj requirement.
                if (app.setAdj < minOomAdj) {
                    continue;
                }

                ... ...

            }
        }   
            
        int N = procs.size();
        for (int i=0; i<N; i++) {
            removeProcessLocked(procs.get(i), callerWillRestart, allowRestart, reason);
        }
        updateOomAdjLocked();
        return N > 0;
    }
```


  **b**. 原生系统在任务列表调用 removeTask 也无法杀死 fg-service（framework/base/services/core/java/com/android/server/am/ActivityManagerService.java）：

```java
    @Override
    public boolean removeTask(int taskId) {
        synchronized (this) {
            enforceCallingPermission(android.Manifest.permission.REMOVE_TASKS,
                    "removeTask()");               
            long ident = Binder.clearCallingIdentity(); 
            try {
                return removeTaskByIdLocked(taskId, true);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }
    }

    /**
     * Removes the task with the specified task id.
     *
     * @param taskId Identifier of the task to be removed.
     * @param killProcess Kill any process associated with the task if possible.
     * @return Returns true if the given task was found and removed.
     */
    private boolean removeTaskByIdLocked(int taskId, boolean killProcess) { 
        TaskRecord tr = mStackSupervisor.anyTaskForIdLocked(taskId, false);
        if (tr != null) {
            tr.removeTaskActivitiesLocked();
            cleanUpRemovedTaskLocked(tr, killProcess);
            if (tr.isPersistable) {        
                notifyTaskPersisterLocked(null, true);
            }
            return true;
        }
        Slog.w(TAG, "Request to remove task ignored for non-existent task " + taskId);
        return false;
    }


    private void cleanUpRemovedTaskLocked(TaskRecord tr, boolean killProcess) {
        mRecentTasks.remove(tr);
        tr.removedFromRecents();
        ComponentName component = tr.getBaseIntent().getComponent();
        if (component == null) {
            Slog.w(TAG, "No component for base intent of task: " + tr);
            return;
        }

        // Find any running services associated with this app and stop if needed.
        mServices.cleanUpRemovedTaskLocked(tr, component, new Intent(tr.getBaseIntent()));

        if (!killProcess) {
            return;
        }

        // Determine if the process(es) for this task should be killed.
        final String pkg = component.getPackageName();
        ArrayList<ProcessRecord> procsToKill = new ArrayList<>();
        ArrayMap<String, SparseArray<ProcessRecord>> pmap = mProcessNames.getMap();
        for (int i = 0; i < pmap.size(); i++) {

            SparseArray<ProcessRecord> uids = pmap.valueAt(i);
            for (int j = 0; j < uids.size(); j++) {
                ProcessRecord proc = uids.valueAt(j);
                if (proc.userId != tr.userId) {
                    // Don't kill process for a different user.
                    continue;
                }
                if (proc == mHomeProcess) {
                    // Don't kill the home process along with tasks from the same package.
                    continue;
                }
                if (!proc.pkgList.containsKey(pkg)) {
                    // Don't kill process that is not associated with this task.
                    continue;
                }

                for (int k = 0; k < proc.activities.size(); k++) {
                    TaskRecord otherTask = proc.activities.get(k).task;
                    if (tr.taskId != otherTask.taskId && otherTask.inRecents) {
                        // Don't kill process(es) that has an activity in a different task that is
                        // also in recents.
                        return;
                    }
                }

                // 不杀死 fg-service 
                if (proc.foregroundServices) {
                    // Don't kill process(es) with foreground service.
                    return;
                }

                // Add process to kill list.
                procsToKill.add(proc);
            }
        }

        // Kill the running processes.
        for (int i = 0; i < procsToKill.size(); i++) {
            ProcessRecord pr = procsToKill.get(i);
            if (pr.setSchedGroup == Process.THREAD_GROUP_BG_NONINTERACTIVE
                    && pr.curReceiver == null) {
                pr.kill("remove task", true);
            } else {
                // We delay killing processes that are not in the background or running a receiver.
                pr.waitingToKill = "remove task";
            }
        }
    }
```

但是 miui 的任务管理界面被改过了，改成调用 forceStopPakcage ... 前面说过了，这个系统接口威力无穷 ... 这个就能杀死 fg-service


### 限制条件

**成为 fg-service 需要绑定一个 ongoing 的通知**。下面是官方解释（Service）：

<pre>
public final void startForeground (int id, Notification notification)  Added in API level 5

Make this service run in the foreground, supplying the ongoing notification to be shown to the user while in this state. By default services are background, meaning that if the system needs to kill them to reclaim more memory (such as to display a large page in a web browser), they can be killed without too much harm. You can set this flag if killing your service would be disruptive to the user, such as if your service is performing background music playback, so the user would notice if their music stopped playing. 
</pre>

官方的设计思路是类似于后台的音乐播放服务，下载服务，是当前用户相关的服务，比较重要，不应该那么容易被杀死。怪不得某些提示用户说，如果要正常使用某些功能，就需要在通知栏上显示一条信息，原来是为了成为 fg-service 让应用在后台存活的时间更长。am 的一些策略对于我们以后的一些优化感觉也比较有用，以后哪位兄弟感兴趣可以一起研究一下。


最后附上 adb shell dumpsys meminfo 和 adb shell dumpsys activity 的结果（有几个系统工具感觉还是挺好用的，特别在分析一些问题上面）：

<pre>
memeinfo:
-----------------
   332856 kB: Perceptible^M
               95989 kB: com.tencent.mm (pid 9818 / activities)^M
               // 注意和下面的 com.kk.dict:push 子进程对比，这个进程的某个 service 绑定了一条 ongoing 通知
               76209 kB: com.kk.dict (pid 24702)^M
               47410 kB: com.iflytek.inputmethod (pid 7386)^M
               26531 kB: com.eg.android.AlipayGphone (pid 10267)^M
               // 下面这个是我调研的应用，琥珀天气
               23666 kB: mobi.infolife.ezweather (pid 23966)^M
               23018 kB: com.tencent.mm:push (pid 10069)^M
               21956 kB: com.eg.android.AlipayGphone:push (pid 8959)^M
               11547 kB: com.sankuai.mtmp.push (pid 8468)^M
                6530 kB: com.iflytek.inputmethod.assist (pid 8273)^M
   128834 kB: A Services^M
               36675 kB: com.taobao.taobao (pid 6360 / activities)^M
               26853 kB: com.taobao.taobao:channel (pid 9847)^M
               20148 kB: com.android.vending (pid 24325)^M
               15947 kB: com.tencent.mobileqq:MSF (pid 10558)^M
               // 普通的 service 在下面的级别
               10074 kB: com.kk.dict:push (pid 24684)^M
                7346 kB: com.sankuai.meituan:pushservice (pid 28738)^M
                6067 kB: com.happyelements.AndroidAnimal (pid 14926)^M
                5724 kB: com.netease.pomelo.push.l.messageservice_V2 (pid 21540)^M


activity: 
-----------------
  Process LRU list (sorted by oom_adj, 66 total, non-act at 10, non-svc at 10):^M
    PERS #65: pers  F/ /P  trm: 0 1687:system/1000 (fixed)^M
    PERS #64: pers  F/ /P  trm: 0 2733:com.android.systemui/1000 (fixed)^M
    PERS #63: pers  F/ /P  trm: 0 2849:com.xiaomi.xmsf/u0a77 (fixed)^M
    PERS #62: pers  F/ /P  trm: 0 2864:com.miui.core/u0a81 (fixed)^M
    PERS #61: pers  F/ /P  trm: 0 3122:com.securespaces.android.ssm.service/1000 (fixed)^M
    PERS #60: pers  F/ /P  trm: 0 3330:com.quicinc.cne.CNEService/1000 (fixed)^M
    PERS #59: pers  F/ /P  trm: 0 3363:com.android.nfc/1027 (fixed)^M
    PERS #58: pers  F/ /P  trm: 0 3393:com.qualcomm.qcrilmsgtunnel/1001 (fixed)^M
    PERS #57: pers  F/ /P  trm: 0 3415:com.miui.whetstone/1000 (fixed)^M
    PERS #56: pers  F/ /P  trm: 0 3474:com.xiaomi.finddevice/u0a133 (fixed)^M
    PERS #55: pers  F/ /P  trm: 0 3480:com.android.phone/1001 (fixed)^M
    Proc #37: fore  F/ /SB trm: 0 3742:com.miui.antispam:provider/1000 (provider)^M
    // 应用变成 fg-service 了
    Proc #13: prcp  F/S/SF trm: 0 6673:com.kk.dict/u0a277 (fg-service)^M
    // 这个是琥珀天气
    Proc #10: prcp  F/S/SF trm: 0 6356:mobi.infolife.ezweather/u0a95 (fg-service)^M
    // 这个是应用的推送进程 cch-empty 比 fg-service 的回收级别高很多
    Proc #16: cch+2 B/ /CE trm: 0 7019:com.kk.dict:push/u0a277 (cch-empty)^M
</pre>


## 通过同步框架定时唤醒

android 有一个帐号同步的框架，具体可以参见：官方文档的 Training --> Transferring Data Using Sync Adapters 有详细的介绍。这个框架的目的本来是提供一个定时从自己的服务器同步（上传，拉取）应用数据功能的。但是其实如果你向这个里面添加了一个自己应用的同步帐号，但是却什么事都不做的，就可以达到定时唤醒你的应用，从而达到提高应用存活率的目的。是不是觉得点恶心，但是很多应用就是这么做的，例如说网易的有道辞典，你可以去系统的设置帐号里查看，看得到有道辞典添加了一个同步帐号，但是你点进去，发现里面什么都没有，是个空白的页面。它就只是利用系统的同步框架来定时唤醒自己的应用，来接收推送消息，弹通知栏而已 ... 

因为这个框架是系统的，所以目前的某些国产系统（miui，flyme）没有屏蔽通过这个框架来唤醒应用。所以这个方法某些程度上比绑定 ongoing 通知还管用 ... 
















