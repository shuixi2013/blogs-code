title: Android Broadcast 分析——发送、处理
date: 2015-01-22 10:15:16
tags: [android]
---

上一篇分析了广播的注册流程，这篇来分析下广播的发送、处理流程。这里为什么把发送和处理和来一起说咧，那是因为其实这是一个过程，发送接口里面差不多就是处理过程了。我们先照例把相关代码位置啰嗦一下（4.2.2）：

```bash
# Content 广播相关的代码
frameworks/base/core/java/android/app/ContextImpl.java

frameworks/base/core/java/android/content/Intent.java
frameworks/base/core/java/android/content/IntentFilter.java
frameworks/base/core/java/android/content/BroadcastReceiver.java
frameworks/base/core/java/android/content/IIntentReceiver.aidl

# 广播解析相关代码
frameworks/base/services/java/com/android/server/IntentResolver.java

# AM 广播相关代码
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/java/com/android/server/am/RecevierList.java
frameworks/base/services/java/com/android/server/am/BroadcastFilter.java

# PM 广播相关代码
frameworks/base/services/java/com/android/server/pm/PackageManagerService.java
frameworks/base/services/java/com/android/server/pm/BroadcastFilter.java
```

## 发送接口

### 应用接口

普通应用发送广播的接口和上一篇注册的一样，都在 Context 里面，但是实现在 ContextImpl 里面：

```java
    @Override
    public void sendBroadcast(Intent intent) { 
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.setAllowFds(false);     
            ActivityManagerNative.getDefault().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, false, false,
                getUserId());
        } catch (RemoteException e) {  
        }
    }
```

这里我们看最简单，也是最常见的那种，直接一个 Intent 发出去的。其实最后是调用到 AMS 里面的同名接口（前面一篇说过了，广播的处理是在 AMS 中的）：

```java
    public final int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle map,
            String requiredPermission, boolean serialized, boolean sticky, int userId) {
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized(this) {
            intent = verifyBroadcastLocked(intent);

            // 获取调用者的进程相关信息
            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            // 下面这个带 Locked 的函数才是关键
            int res = broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null, 
                    intent, resolvedType, resultTo,
                    resultCode, resultData, map, requiredPermission, serialized, sticky,
                    callingPid, callingUid, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }     
    }
```

AMS 里面的这个接口，有一个 boolean 参数 serialized（还有个 sticky 的，我们不管这个 stciky）。true 的话，表示发送的串行广播，false 表示发送并行广播。串行广播就是说，一个广播来了，有一对接收器， AMS 会等前一个执行完，才会发给下一个处理，**接收器处理广播是一个接着一个处理的**。并行的就是 AMS 把广播发给一个接收器之后，会马上返回，然后再发给下一个，直到发送完，**接收器处理广播是并行的（同时处理）**。用简单参数的接口，**默认发送的是并行广播**。

回到接口上，加了多线程互斥锁之后，调用 broadcastIntentLocked 处理。这个函数就是看名字是发送广播（Intent），其实就是处理过程，而且非常长，我们发到后面慢慢说。

### 系统发送接口

我们来看看系统是怎么发送的。我们以前面说的 BOOT_COMPLETED 广播来看。这个广播是 AMS 中发出来的：

```java
    final void finishBooting() {
        IntentFilter pkgFilter = new IntentFilter();
        pkgFilter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
        pkgFilter.addDataScheme("package");

... ...

        synchronized (this) {
            // Ensure that any processes we had put on hold are now started
            // up.

... ...

            if (mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {
                // Start looking for apps that are abusing wake locks.
                Message nmsg = mHandler.obtainMessage(CHECK_EXCESSIVE_WAKE_LOCKS_MSG);
                mHandler.sendMessageDelayed(nmsg, POWER_CHECK_DELAY);
                // Tell anyone interested that we are done booting!
                SystemProperties.set("sys.boot_completed", "1");
                SystemProperties.set("dev.bootcomplete", "1");
                for (int i=0; i<mStartedUsers.size(); i++) {
                    UserStartedState uss = mStartedUsers.valueAt(i);
                    if (uss.mState == UserStartedState.STATE_BOOTING) {
                        uss.mState = UserStartedState.STATE_RUNNING;
                        final int userId = mStartedUsers.keyAt(i);
                        Intent intent = new Intent(Intent.ACTION_BOOT_COMPLETED, null);
                        intent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
                        // AMS 里面自己发，直接调用这个函数了
                        // serialized 是 false，这个是一个并行广播
                        broadcastIntentLocked(null, null, intent,
                                null, null, 0, null, null,
                                android.Manifest.permission.RECEIVE_BOOT_COMPLETED,
                                false, false, MY_PID, Process.SYSTEM_UID, userId);
                    }
                }
            }
        }
    }
```

AMS 里面自己发广播，直接调用 broadcastIntentLocked 了。估计其他系统服务里面还是调用 AMS 的 broadcastIntent 接口的吧。

## 处理流程

AMS 中处理广播的流程就是 broadcastIntentLocked 这个函数，这个函数也是非常长的（差不多6、7百行），我们分段慢慢来（会跳过一些非重要的部分）：

### 1. 收集广播接收器

```java
    private final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle map, String requiredPermission,
            boolean ordered, boolean sticky, int callingPid, int callingUid,
            int userId) {
        intent = new Intent(intent);

        // By default broadcasts do not go to stopped apps.
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);

        if (DEBUG_BROADCAST_LIGHT) Slog.v(
            TAG, (sticky ? "Broadcast sticky: ": "Broadcast: ") + intent
            + " ordered=" + ordered + " userid=" + userId);
        if ((resultTo != null) && !ordered) {
            Slog.w(TAG, "Broadcast " + intent + " not ordered but result callback requested!");
        }

        // 获取用户信息（用户中，不同用户的广播是分开的）
        userId = handleIncomingUser(callingPid, callingUid, userId,
                true, false, "broadcast", callerPackage);

        // Make sure that the user who is receiving this broadcast is started.
        // If not, we will just skip it.
        if (userId != UserHandle.USER_ALL && mStartedUsers.get(userId) == null) {
            if (callingUid != Process.SYSTEM_UID || (intent.getFlags()
                    & Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0) {
                Slog.w(TAG, "Skipping broadcast of " + intent
                        + ": user " + userId + " is stopped");
                return ActivityManager.BROADCAST_SUCCESS;
            }     
        }
        
        // 这里是一些发送权限，检测和 sticky 功能，我们通通跳过 
... ...

        int[] users;
        if (userId == UserHandle.USER_ALL) {
            // Caller wants broadcast to go to all started users.
            users = mStartedUserArray;
        } else {
            // Caller wants broadcast to go to one specific user.
            users = new int[] {userId};
        }

        // Figure out who all will receive this broadcast.
        List receivers = null;
        List<BroadcastFilter> registeredReceivers = null;
        // Need to resolve the intent to interested receivers...
        // 这里还有个判断，如果发送的广播并不是只能动态注册的才跑下面的（也是说静态的可以，真绕口）
        if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
                 == 0) {
            // 收集静态注册的接收器
            receivers = collectReceiverComponents(intent, resolvedType, users);
        }
        // 看样子发一条动态注册接收器能处理的广播，发的 Intent 必须要包含 Component 信息
        if (intent.getComponent() == null) {
            // 收集动态注册的接收器
            registeredReceivers = mReceiverResolver.queryIntent(intent,
                    resolvedType, false, userId);
        }

        final boolean replacePending =
                (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;

        if (DEBUG_BROADCAST) Slog.v(TAG, "Enqueing broadcast: " + intent.getAction()
                + " replacePending=" + replacePending);

... ...

        return ActivityManager.BROADCAST_SUCCESS;
    }
```

一个广播发出来了，AMS 要做的第一步，首先是要找到有哪些接收器要接收这条广播。前面一篇说了，注册接收器有动态注册和静态注册2种。这里看代码果然也是分开2步来收集的。我们先来看收集静态的：

#### 1.1 收集静态注册接收器

静态注册接收器收集由 collectReceiverComponents 处理：

```java
    private List<ResolveInfo> collectReceiverComponents(Intent intent, String resolvedType,
            int[] users) {
        List<ResolveInfo> receivers = null;
        try {
            HashSet<ComponentName> singleUserReceivers = null;
            boolean scannedFirstReceivers = false;
            for (int user : users) {
                // 去 PM 中取静态注册的接收器列表
                List<ResolveInfo> newReceivers = AppGlobals.getPackageManager()
                        .queryIntentReceivers(intent, resolvedType, STOCK_PM_FLAGS, user);

                // 下面那一堆是关于多用户处理的，我们不管它们
... ...

            }
        } catch (RemoteException ex) {
            // pm is in same process, this will never happen.
        }
        return receivers;
    }
```

这个函数虽然有 100 多行，但是我们关心的只有调用 PM 的那句 queryIntentReceivers 而已。前面一篇说了，静态注册的广播数据保存在 PMS 中，所以这里要调用 PM 的接口去 PMS 里面去取。所以我去 PMS 里面去看看：

```java
    @Override
    public List<ResolveInfo> queryIntentReceivers(Intent intent, String resolvedType, int flags,
            int userId) {
        if (!sUserManager.exists(userId)) return Collections.emptyList(); 
        ComponentName comp = intent.getComponent();
        if (comp == null) {
            if (intent.getSelector() != null) {    
                intent = intent.getSelector();
                comp = intent.getComponent();  
            }       
        }
        // 如果发送的广播带 ComponentName 信息，差不多就是显式广播了（指定了接收器包名、类名的）
        // 所以只要有一个接收器就行了
        if (comp != null) {
            List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);
            ActivityInfo ai = getReceiverInfo(comp, flags, userId);
            if (ai != null) {
                ResolveInfo ri = new ResolveInfo();
                ri.activityInfo = ai;
                list.add(ri);
            }           
            return list;    
        }                   
                                
        // 但是一般的都是走下面这里的
        // 我们的例子 BOOT_COMPLETED 属于这种
        // reader                       
        synchronized (mPackages) { 
            String pkgName = intent.getPackage();
            // 一般隐式的也是不指定包名的
            // 我们的例子 BOOT_COMPLETED 属于这种
            if (pkgName == null) {
                // 所以一般是用这个收集静态注册接收器的                             
                return mReceivers.queryIntent(intent, resolvedType, flags, userId);
            }
            final PackageParser.Package pkg = mPackages.get(pkgName);
            if (pkg != null) {
                return mReceivers.queryIntentForPackage(intent, resolvedType, flags, pkg.receivers,
                        userId);
            }
            return null;
        }
    }
```

我们在说之前先看看 ResolveInfo 这个东西，好歹返回的列表里面是这个东西：

```java
/**
 * Information that is returned from resolving an intent
 * against an IntentFilter. This partially corresponds to
 * information collected from the AndroidManifest.xml's
 * &lt;intent&gt; tags.
 */
public class ResolveInfo implements Parcelable {
... ...
}
```

里面的具体东西我们就不看了，看下注释，是说一个 ResolverInfo 对应一个 IntentFilter。然后就是 mReceivers 这个 ActivityIntentResolver，前一篇已经提及过了。PMS 中就是它保存了静态注册的接收器。

未完待续 ... ...





