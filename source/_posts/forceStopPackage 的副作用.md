title: forceStopPackage 的副作用
date: 2015-01-28 19:35:16
updated: 2015-01-28 19:35:16
categories: [Android Framework]
tags: [android]
---

ActivityManager 有个一个函数 forceStopPackage: 

```java
    /**
     * Have the system perform a force stop of everything associated with
     * the given application package.  All processes that share its uid
     * will be killed, all services it has running stopped, all activities
     * removed, etc.  In addition, a {@link Intent#ACTION_PACKAGE_RESTARTED}
     * broadcast will be sent, so that any of its registered alarms can
     * be stopped, notifications removed, etc.
     * 
     * <p>You must hold the permission
     * {@link android.Manifest.permission#FORCE_STOP_PACKAGES} to be able to
     * call this method.
     * 
     * @param packageName The name of the package to be stopped.
     * 
     * @hide This is not available to third party applications due to
     * it allowing them to break other applications by stopping their
     * services, removing their alarms, etc.
     */
    public void forceStopPackage(String packageName) {
        try {
            ActivityManagerNative.getDefault().forceStopPackage(packageName,
                    UserHandle.myUserId());
        } catch (RemoteException e) { 
        }    
    }
```

函数具体的实现流程在 ActivityManagerService 中，具体实现这里不分析（后面可以分析一下）。这个函数用于强制终止指定的 package，包括这个 package 所在的进程，包括的 Service。并且终止 Service 不会被当作 crash 而被系统重新启动起来。就是俗称杀得干净，不像某些 xx 助手的一键清理，那些一键清理调用的是 killBackgroundProcesses(String packageName) 。这个函数虽然能杀死一些后台进程，但是其所属的 Service 会由于异常终止，会被系统重新启动起来。所以那些 xx 助手的清理功能基本上是没用的。

这个函数虽然猛，但是对调用者有严格限制，不仅要求调用者的进程具有系统权限（在 /system/app 下），而且必需具有系统签名（用编译系统 rom 的签名进行编译）。否则调用报权限异常。所以第三方应用想通过反射、root 之类调用的就省省吧，目前官方系统只有系统的 Setting 里面的强制终止应用会调用这个函数。

不过我工作上定制 rom，弄了个帐号切换的功能，然后切换不同帐号要清理下当前的任务，我于是就调用这个函数来清理。清理倒是清理得很干净，但是最近发现一个问题：这个函数强制终止应用，还会把应用设置了的 alarm 给清理掉。这点要特别注意，怪不得在 Setting 里面强制终止应用的时候，会弹一个提示框告诉你可能会出现应用异常。这还真异常了。简单的贴下代码：

在 ActivityManagerService 里面：

```java
    public void forceStopPackage(final String packageName, int userId) {
        if (checkCallingPermission(android.Manifest.permission.FORCE_STOP_PACKAGES)
                != PackageManager.PERMISSION_GRANTED) {
            String msg = "Permission Denial: forceStopPackage() from pid="
                    + Binder.getCallingPid()
                    + ", uid=" + Binder.getCallingUid()
                    + " requires " + android.Manifest.permission.FORCE_STOP_PACKAGES;
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, true, true, "forceStopPackage", null);
        long callingId = Binder.clearCallingIdentity();
        try {
            IPackageManager pm = AppGlobals.getPackageManager();
            synchronized(this) {
                int[] users = userId == UserHandle.USER_ALL
                        ? getUsersLocked() : new int[] { userId };
                for (int user : users) {
                    int pkgUid = -1;
                    try {
                        pkgUid = pm.getPackageUid(packageName, user);
                    } catch (RemoteException e) {
                    }
                    if (pkgUid == -1) {
                        Slog.w(TAG, "Invalid packageName: " + packageName);
                        continue;
                    }
                    try {
                        pm.setPackageStoppedState(packageName, true, user);
                    } catch (RemoteException e) {
                    } catch (IllegalArgumentException e) {
                        Slog.w(TAG, "Failed trying to unstop package "
                                + packageName + ": " + e);
                    }
                    if (isUserRunningLocked(user, false)) {
                        forceStopPackageLocked(packageName, pkgUid);
                    }
                }
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }


    private void forceStopPackageLocked(final String packageName, int uid) {
        forceStopPackageLocked(packageName, UserHandle.getAppId(uid), false,
                false, true, false, UserHandle.getUserId(uid));
        // 发送 ACTION_PACKAGE_RESTARTED 广播
        Intent intent = new Intent(Intent.ACTION_PACKAGE_RESTARTED,
                Uri.fromParts("package", packageName, null));
        if (!mProcessesReady) {
            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                    | Intent.FLAG_RECEIVER_FOREGROUND);
        }
        intent.putExtra(Intent.EXTRA_UID, uid);
        intent.putExtra(Intent.EXTRA_USER_HANDLE, UserHandle.getUserId(uid));
        broadcastIntentLocked(null, null, intent,
                null, null, 0, null, null, null,
                false, false,
                MY_PID, Process.SYSTEM_UID, UserHandle.getUserId(uid));
    }
```

这里会发一个 `ACTION_PACKAGE_RESTARTED` 的广播，在 AlarmManagerService 里面会接收这个广播，判断这个广播的 package name 如果在设置过了的 alarm list 中，就会把对应的 alarm 给删掉（AlarmManagerSerivce 用了几个 ArrayList 来保存不同 type 的 alarm）。

```cpp
    // 这个是 AlarmManagerService 里面注册的广播接收器
    class UninstallReceiver extends BroadcastReceiver {
        public UninstallReceiver() {
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_PACKAGE_REMOVED);
            // 接受 forceStopPackage 发送的广播
            filter.addAction(Intent.ACTION_PACKAGE_RESTARTED);
            filter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
            filter.addDataScheme("package");
            mContext.registerReceiver(this, filter);
             // Register for events related to sdcard installation.
            IntentFilter sdFilter = new IntentFilter();
            sdFilter.addAction(Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE);
            sdFilter.addAction(Intent.ACTION_USER_STOPPED);
            mContext.registerReceiver(this, sdFilter);
        }    

        @Override
        public void onReceive(Context context, Intent intent) {
            synchronized (mLock) {
                String action = intent.getAction();
                Slog.v(TAG, "uninstall recevier action=" + action);
                String pkgList[] = null;
                if (Intent.ACTION_QUERY_PACKAGE_RESTART.equals(action)) {
                    pkgList = intent.getStringArrayExtra(Intent.EXTRA_PACKAGES);
                    for (String packageName : pkgList) {
                        if (lookForPackageLocked(packageName)) {
                            setResultCode(Activity.RESULT_OK);
                            return;
                        }    
                    }    
                    return;
                } else if (Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE.equals(action)) {
                    pkgList = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
                } else if (Intent.ACTION_USER_STOPPED.equals(action)) {
                    int userHandle = intent.getIntExtra(Intent.EXTRA_USER_HANDLE, -1); 
                    if (userHandle >= 0) { 
                        removeUserLocked(userHandle);
                    }    
                } else {
                    if (Intent.ACTION_PACKAGE_REMOVED.equals(action)
                            && intent.getBooleanExtra(Intent.EXTRA_REPLACING, false)) {
                        // This package is being updated; don't kill its alarms.
                        return;
                    }
                    Uri data = intent.getData();
                    if (data != null) {  
                        String pkg = data.getSchemeSpecificPart();
                        if (pkg != null) {
                            pkgList = new String[]{pkg};
                        }
                    }
                }
                if (pkgList != null && (pkgList.length > 0)) {
                    for (String pkg : pkgList) {
                        // 删除配置的 alarm
                        removeLocked(pkg);
                        mBroadcastStats.remove(pkg);
                    }
                }
            }
        }
}
```

其中最后那里那个 removeLocked(String packageName) 就是去删除广播中发的包设置过的 alarm（具体代码这里不贴了，可以去查看 AlarmManagerService）。

这里记录这篇文章就是提醒下，调用这个函数会有这个“副作用”，免得下次应用的 alarm 没反应，啥头绪都没用。


