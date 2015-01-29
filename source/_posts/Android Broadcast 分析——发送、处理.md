title: Android Broadcast 分析——发送、处理
date: 2015-01-22 10:15:16
categories: [Android Framework]
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
普通应用发送广播的接口和注册篇注册的一样，都在 Context 里面，但是实现在 ContextImpl 里面：

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
/*
 * Information that is returned from resolving an intent
 * against an IntentFilter. This partially corresponds to
 * information collected from the AndroidManifest.xml's
 * &lt;intent&gt; tags.
 */
public class ResolveInfo implements Parcelable {
... ...
}
```

里面的具体东西我们就不看了，看下注释，是说一个 ResolverInfo 对应一个 IntentFilter。然后就是 mReceivers 这个 ActivityIntentResolver，注册篇已经提及过了。PMS 中就是它保存了静态注册的接收器。

```java
        public List<ResolveInfo> queryIntent(Intent intent, String resolvedType, int flags,
                int userId) {
            if (!sUserManager.exists(userId)) return null;
            mFlags = flags;
            // 这个差不多就是直接调用父类的同名函数
            return super.queryIntent(intent, resolvedType,
                    (flags & PackageManager.MATCH_DEFAULT_ONLY) != 0, userId);
        }
```

我们去看父类（IntentResolver）的 queryIntent：

```java
    // 前面注册篇说了么，R 是输出类型
    public List<R> queryIntent(Intent intent, String resolvedType, boolean defaultOnly,
            int userId) {
        String scheme = intent.getScheme();

        ArrayList<R> finalList = new ArrayList<R>();

        final boolean debug = localLOGV ||
                ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);

        if (debug) Slog.v(
            TAG, "Resolving type " + resolvedType + " scheme " + scheme
            + " of intent " + intent);

        // F 是输入，这几个数组分别对应注册篇说的 IntentResovler 中的那几个保存的 HashMap（Set）
        F[] firstTypeCut = null;
        F[] secondTypeCut = null;
        F[] thirdTypeCut = null;
        F[] schemeCut = null;

        // MIME type，我们的例子 BOOT_COMPLETED 是 null
        // If the intent includes a MIME type, then we want to collect all of
        // the filters that match that MIME type.
        if (resolvedType != null) {
            int slashpos = resolvedType.indexOf('/');
            if (slashpos > 0) {
                final String baseType = resolvedType.substring(0, slashpos);
                if (!baseType.equals("*")) {
                    if (resolvedType.length() != slashpos+2
                            || resolvedType.charAt(slashpos+1) != '*') {
                        // Not a wild card, so we can just look for all filters that
                        // completely match or wildcards whose base type matches.
                        firstTypeCut = mTypeToFilter.get(resolvedType);
                        if (debug) Slog.v(TAG, "First type cut: " + firstTypeCut);
                        secondTypeCut = mWildTypeToFilter.get(baseType);
                        if (debug) Slog.v(TAG, "Second type cut: " + secondTypeCut);
                    } else {
                        // We can match anything with our base type.
                        firstTypeCut = mBaseTypeToFilter.get(baseType);
                        if (debug) Slog.v(TAG, "First type cut: " + firstTypeCut);
                        secondTypeCut = mWildTypeToFilter.get(baseType);
                        if (debug) Slog.v(TAG, "Second type cut: " + secondTypeCut);
                    }
                    // Any */* types always apply, but we only need to do this
                    // if the intent type was not already */*.
                    thirdTypeCut = mWildTypeToFilter.get("*");
                    if (debug) Slog.v(TAG, "Third type cut: " + thirdTypeCut);
                } else if (intent.getAction() != null) {
                    // The intent specified any type ({@literal *}/*).  This
                    // can be a whole heck of a lot of things, so as a first
                    // cut let's use the action instead.
                    firstTypeCut = mTypedActionToFilter.get(intent.getAction());
                    if (debug) Slog.v(TAG, "Typed Action list: " + firstTypeCut);
                }   
            }           
        }   
                        
        // Scheme，我们的例子 BOOT_COMPLETED 也是 null
        // If the intent includes a data URI, then we want to collect all of
        // the filters that match its scheme (we will further refine matches
        // on the authority and path by directly matching each resulting filter).
        if (scheme != null) {   
            schemeCut = mSchemeToFilter.get(scheme);
            if (debug) Slog.v(TAG, "Scheme list: " + schemeCut);
        }

        // Action，我们的例子 BOOT_COMPLETED 本身就是 Action 啦
        // 所以这里 firstTypeCut 就是取前面注册篇中保存 Action 的列表了
        // If the intent does not specify any data -- either a MIME type or
        // a URI -- then we will only be looking for matches against empty
        // data.
        if (resolvedType == null && scheme == null && intent.getAction() != null) {
            firstTypeCut = mActionToFilter.get(intent.getAction());
            if (debug) Slog.v(TAG, "Action list: " + firstTypeCut);
        }

        // 这个 Categories，我们也暂时不管先
        FastImmutableArraySet<String> categories = getFastIntentCategories(intent);
        // 然后按照顺序依次从匹配的 F 列表中构造出要查询的 R 返回给调用者
        if (firstTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, firstTypeCut, finalList, userId);
        }
        if (secondTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, secondTypeCut, finalList, userId);
        }
        if (thirdTypeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, thirdTypeCut, finalList, userId);
        }
        if (schemeCut != null) {
            buildResolveList(intent, categories, debug, defaultOnly,
                    resolvedType, scheme, schemeCut, finalList, userId);
        }
        // 对匹配的结果进行排序
        sortResults(finalList);

        // 兼容老版本的，无视 ... ...
        if (VALIDATE) {
            List<R> oldList = mOldResolver.queryIntent(intent, resolvedType, defaultOnly, userId);
            if (oldList.size() != finalList.size()) {
                ValidationFailure here = new ValidationFailure();
                here.fillInStackTrace();
                Log.wtf(TAG, "Query result " + intent + " size is " + finalList.size()
                        + "; old implementation is " + oldList.size(), here);
            }
        }

        if (debug) {
            Slog.v(TAG, "Final result list:");
            for (R r : finalList) {
                Slog.v(TAG, "  " + r);
            }
        }
        return finalList;
    }
```

这个函数前面是通过传过来的 Intent 从 IntentResolver 那一堆保存注册接收器数据的列表中选出匹配的 F（PMS 中的静态注册的 F 是 PackageParser.ActivityIntentInfo），一般来说匹配条件就是注册广播中 IntentFilter 可以设置的那一堆条件了：MIME type、Scheme、Action 等等（到这里的都是隐式的，显式的前面直接给解析好了）。然后我们就得去看看 buildResolveList 这个函数是怎么构造 R 出来的：

```java
    private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
            boolean debug, boolean defaultOnly,
            String resolvedType, String scheme, F[] src, List<R> dest, int userId) {
        final String action = intent.getAction();
        final Uri data = intent.getData();
        final String packageName = intent.getPackage();

        final boolean excludingStopped = intent.isExcludingStopped();

        final int N = src != null ? src.length : 0;
        boolean hasNonDefaults = false;
        int i;
        F filter;
        // 在选出的 F 数组里，循环根据 F 构造出 R
        for (i=0; i<N && (filter=src[i]) != null; i++) {
            int match;
            if (debug) Slog.v(TAG, "Matching against filter " + filter);

            if (excludingStopped && isFilterStopped(filter, userId)) {
                if (debug) {
                    Slog.v(TAG, "  Filter's target is stopped; skipping");
                }   
                continue;
            }   

            // Is delivery being limited to filters owned by a particular package?
            if (packageName != null && !packageName.equals(packageForFilter(filter))) {
                if (debug) {
                    Slog.v(TAG, "  Filter is not from package " + packageName + "; skipping");
                }   
                continue;
            }   

            // 这个 allowFilterResult 是由之类实现的，不过和我们这里的关系不大，不理它先
            // Do we already have this one?
            if (!allowFilterResult(filter, dest)) {
                if (debug) {
                    Slog.v(TAG, "  Filter's target already added");
                }   
                continue;
            }   

            // IntentFilter 里面的匹配函数，我们也先忽略一下（就是拿上面几种类型匹配一下而已）
            match = filter.match(action, resolvedType, scheme, data, categories, TAG);
            if (match >= 0) {
                if (debug) Slog.v(TAG, "  Filter matched!  match=0x" +
                        Integer.toHexString(match));
                if (!defaultOnly || filter.hasCategory(Intent.CATEGORY_DEFAULT)) {
                    // 这里 new R 出来的 newResult 也是由子类实现的，我们待会去看一下
                    final R oneResult = newResult(filter, match, userId);
                    if (oneResult != null) {
                        // 把 R 添加到输出列表中
                        dest.add(oneResult);
                    }
                } else {
                    hasNonDefaults = true;
                }
            } else {
                if (debug) {
                    String reason;
                    switch (match) {
                        case IntentFilter.NO_MATCH_ACTION: reason = "action"; break;
                        case IntentFilter.NO_MATCH_CATEGORY: reason = "category"; break;
                        case IntentFilter.NO_MATCH_DATA: reason = "data"; break;
                        case IntentFilter.NO_MATCH_TYPE: reason = "type"; break;
                        default: reason = "unknown reason"; break;
                    }
                    Slog.v(TAG, "  Filter did not match: " + reason);
                }
            }
        }

        if (dest.size() == 0 && hasNonDefaults) {
            Slog.w(TAG, "resolveIntent failed: found match, but none with Intent.CATEGORY_DEFAULT");
        }
    }
```

这个 buildResolveList 中就是循环从输入的 F 数组中取出 F，然后传给一个叫 newResult 的函数构造 R。这个 newResult 是由不同的 IntentResovler 之类实现的。我们来看看 PMS 中的 ActivityIntentResolver newResult 的实现：

```java
        @Override
        protected ResolveInfo newResult(PackageParser.ActivityIntentInfo info,
                int match, int userId) {       
            // 下面一堆判断可以不用管先
            if (!sUserManager.exists(userId)) return null;
            if (!mSettings.isEnabledLPr(info.activity.info, mFlags, userId)) {
                return null;
            }
            final PackageParser.Activity activity = info.activity;
            if (mSafeMode && (activity.info.applicationInfo.flags
                    &ApplicationInfo.FLAG_SYSTEM) == 0) {
                return null;
            }
            PackageSetting ps = (PackageSetting) activity.owner.mExtras;
            if (ps == null) {
                return null;
            }
            ActivityInfo ai = PackageParser.generateActivityInfo(activity, mFlags,
                    ps.readUserState(userId), userId);
            if (ai == null) {
                return null;
            }
            // 下面就是构造 R（ResolverInfo） 的过程
            final ResolveInfo res = new ResolveInfo();
            res.activityInfo = ai;         
            if ((mFlags&PackageManager.GET_RESOLVED_FILTER) != 0) {
                res.filter = info;             
            }
            // 接收器的优先级在这里复制的哦
            res.priority = info.getPriority();
            res.preferredOrder = activity.owner.mPreferredOrder;
            //System.out.println("Result: " + res.activityInfo.className +
            //                   " = " + res.priority);
            res.match = match;
            res.isDefault = info.hasDefault;
            res.labelRes = info.labelRes;
            res.nonLocalizedLabel = info.nonLocalizedLabel;
            res.icon = info.icon;
            res.system = isSystemApp(res.activityInfo.applicationInfo);
            return res;
        }
```

上面就是 new 了一个 ResolverInfo（R），然后用 F（PackageParser.ActivityIntentInfo） 中相应的字段填充自己的字段而已。后面 AMS 从自己那里收集动态注册的接收器，也是差不多的。所以说注册的时候保存的数据，差不多可以说就是接收器数据。

#### 1.2 收集动态注册接收器
上面看过收集静态的过程，下面我们来看看收集动态注册的接收器：

```java
            registeredReceivers = mReceiverResolver.queryIntent(intent,
                    resolvedType, false, userId);
```

这个其实和上面静态流程是一样的，只不过 IntentResolver 的实现子类是 AMS 的匿名的一个类而已，我们就直接看看关键的 newResult 就行了：

```java
        @Override
        protected BroadcastFilter newResult(BroadcastFilter filter, int match, int userId) {
            if (userId == UserHandle.USER_ALL || filter.owningUserId == UserHandle.USER_ALL
                    || userId == filter.owningUserId) {
                return super.newResult(filter, match, userId);
            }
            return null;
        }

// ================ IntentResovler.java ============================

    @SuppressWarnings("unchecked")
    protected R newResult(F filter, int match, int userId) { 
        return (R)filter;
    }
```

AMS 里面的除去前面那个判断，就是直接 new 了一个 R，也难怪，AMS 里面的 F 和 R 都是 BroadcastFilter。到这里 AMS 的 broadcastIntentLocked 就收集到了2个 List：

<pre>
List receivers = null;
List<BroadcastFilter> registeredReceivers = null;
</pre>

分别是静态注册的接收器（ResolveInfo）和动态注册的接收器（BoradcastFilter）。

### 2. 分发广播给动态注册接收器
收集完广播匹配的接收器（静态和动态），就要开始分发了。我们接着往下看：

```java
    private final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle map, String requiredPermission,
            boolean ordered, boolean sticky, int callingPid, int callingUid,
            int userId) {
... ...
            
        int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        // 判断如果不是串行广播并且有动态注册的接收器，就走下面的代码
        // 我们的例子（BOOT_COMPLETED）是并行广播，并且有不少动态注册的接收器的
        if (!ordered && NR > 0) {
            // If we are not serializing this broadcast, then send the
            // registered receivers separately so they don't wait for the
            // components to be launched.
            // 取广播对应的广播队列：分为前台广播队列和后台广播队列
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            // new 一个 BroadcastRecord，注意有传动态注册接收器的列表进去
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, requiredPermission,
                    registeredReceivers, resultTo, resultCode, resultData, map,
                    ordered, sticky, false, userId);
            if (DEBUG_BROADCAST) Slog.v(
                    TAG, "Enqueueing parallel broadcast " + r);
            // 这个 replaced 大概就是检测下现在发的这个 Intent 是不是和广播队列中的有一样的，
            // 如果有的话，说明之前发过一次了，但是还没处理，所以就忽略这次发送（代码不贴了）
            final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
            if (!replaced) {
                // 把前面 new 出来的 BroadcastRecord 送去并行的广播队列排队，然后执行发送
                queue.enqueueParallelBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
            registeredReceivers = null;
            NR = 0;
        }

... ....

        return ActivityManager.BROADCAST_SUCCESS;
    }
```

这里出现了2个新的数据结构：BroadcastQueue 和 BroadcastRecord。我们来一个一个看，首先是 BroadcastQueue：

```java
/*
 * BROADCASTS
 *
 * We keep two broadcast queues and associated bookkeeping, one for those at
 * foreground priority, and one for normal (background-priority) broadcasts.
 */
public class BroadcastQueue {
    static final String TAG = "BroadcastQueue";
    static final String TAG_MU = ActivityManagerService.TAG_MU;
    static final boolean DEBUG_BROADCAST = ActivityManagerService.DEBUG_BROADCAST;
    static final boolean DEBUG_BROADCAST_LIGHT = ActivityManagerService.DEBUG_BROADCAST_LIGHT;
    static final boolean DEBUG_MU = ActivityManagerService.DEBUG_MU;

    static final int MAX_BROADCAST_HISTORY = 25;
    static final int MAX_BROADCAST_SUMMARY_HISTORY = 100; 

    final ActivityManagerService mService;

    /**
     * Recognizable moniker for this queue
     */
    final String mQueueName;

    /**
     * Timeout period for this queue's broadcasts
     */
    final long mTimeoutPeriod;

    /**
     * Lists of all active broadcasts that are to be executed immediately
     * (without waiting for another broadcast to finish).  Currently this only
     * contains broadcasts to registered receivers, to avoid spinning up
     * a bunch of processes to execute IntentReceiver components.  Background-
     * and foreground-priority broadcasts are queued separately.
     */
    final ArrayList<BroadcastRecord> mParallelBroadcasts
            = new ArrayList<BroadcastRecord>();
    /**
     * List of all active broadcasts that are to be executed one at a time.
     * The object at the top of the list is the currently activity broadcasts;
     * those after it are waiting for the top to finish.  As with parallel
     * broadcasts, separate background- and foreground-priority queues are
     * maintained.
     */
    final ArrayList<BroadcastRecord> mOrderedBroadcasts
            = new ArrayList<BroadcastRecord>();

... ...

}
```

看注释，说 AMS 中有2个这种 BroacastQueue，一个是前台的，一个是后台的，前台的处理优先级比后台的高一些。然后里面有2个比较重要的 ArrayList：mParallelBroadcasts 和 mOrderedBroadcasts。看名字就很明显了，一个是串行广播记录的，一个是并行广播记录的。然后我们再来看下 BroadcastRecord 的：

```java
/*
 * An active intent broadcast.
 */
class BroadcastRecord extends Binder {
    final Intent intent;    // the original intent that generated us
    final ProcessRecord callerApp; // process that sent this
    final String callerPackage; // who sent this
    final int callingPid;   // the pid of who sent this
    final int callingUid;   // the uid of who sent this
    final boolean ordered;  // serialize the send to receivers?
    final boolean sticky;   // originated from existing sticky data?
    final boolean initialSticky; // initial broadcast from register to sticky?
    final int userId;       // user id this broadcast was for
    final String requiredPermission; // a permission the caller has required
    final List receivers;   // contains BroadcastFilter and ResolveInfo
    IIntentReceiver resultTo; // who receives final result if non-null
    long dispatchTime;      // when dispatch started on this set of receivers
    long dispatchClockTime; // the clock time the dispatch started
    long receiverTime;      // when current receiver started for timeouts.
    long finishTime;        // when we finished the broadcast.
    int resultCode;         // current result code value.
    String resultData;      // current result data value.
    Bundle resultExtras;    // current result extra data values.
    boolean resultAbort;    // current result abortBroadcast value.
    int nextReceiver;       // next receiver to be executed.
    IBinder receiver;       // who is currently running, null if none.
    int state;
    int anrCount;           // has this broadcast record hit any ANRs?
    BroadcastQueue queue;   // the outbound queue handling this broadcast

... ...

}
```

这个类很简单，除了构造函数给上面一堆字段赋值以外，就没别的的啥操作了。然后上面注意那个 List 记录了 AMS 收集到的广播接收器的信息。接下来我们回到 broadcastIntentLocked 中看看 broadcastQueueForIntent 是怎么决定用哪个广播队列的：

```java
    // 顺带把初始化的地方贴出来
    private ActivityManagerService() {
        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());
                    
        // 这里创建广播队列的时候就设置好了广播超时时间（这个时间的作用，后面再说）
        mFgBroadcastQueue = new BroadcastQueue(this, "foreground", BROADCAST_FG_TIMEOUT);
        mBgBroadcastQueue = new BroadcastQueue(this, "background", BROADCAST_BG_TIMEOUT);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
... ...
       
    } 

    BroadcastQueue broadcastQueueForIntent(Intent intent) {
        final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
        if (DEBUG_BACKGROUND_BROADCAST) {
            Slog.i(TAG, "Broadcast intent " + intent + " on "
                    + (isFg ? "foreground" : "background")
                    + " queue");
        }   
        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    } 
```

看样子是根据发广播的时候的 Intent 的 flags 来决定的，默认不设置就是后台广播队列。我搜了下代码，发现就只有 AMS 中会发几个是前台的。我们确定了我们的列子 BOOT_COMPLETED 是用后台广播队列之后，就来看看 BroadcastQueue 中的加入队列和执行操作：

```java
    public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
        mParallelBroadcasts.add(r);
    }
```

入队操作很简单，直接插入到并行广播记录的列表中。排好队之后就到执行广播了：

```java
    public void scheduleBroadcastsLocked() {
        RuntimeException here = new RuntimeException("here");
        here.fillInStackTrace();
        Slog.d(TAG, "call statck is", here);

        if (DEBUG_BROADCAST) Slog.v(TAG, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);

        // 如果当前队列正在执行广播操作，则返回
        // 也就是说一个广播队列要等上一次广播操作执行完才会接收新的执行命令
        if (mBroadcastsScheduled) {
            return;
        }    
        // 发了一个消息去 Handler 里面
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        // 把正在执行的标志设置为 true
        mBroadcastsScheduled = true;
    }

    // 这个 Handler 没有使用额外的线程，
    // 所以这里的作用应该是为了能够马上返回吧
    final Handler mHandler = new Handler() {
        //public Handler() {
        //    if (localLOGV) Slog.v(TAG, "Handler started!");
        //}

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
```

最后执行操作是由 processNextBroadcast 这个函数来完成的。这个函数非常的长（将近 500 行），我们得分功能一段一段来：

```java
    final void processNextBroadcast(boolean fromMsg) {
        synchronized(mService) {
            BroadcastRecord r;

            if (DEBUG_BROADCAST) Slog.v(TAG, "processNextBroadcast ["
                    + mQueueName + "]: "
                    + mParallelBroadcasts.size() + " broadcasts, "
                    + mOrderedBroadcasts.size() + " ordered broadcasts");

            mService.updateCpuStats();

            // 如果从 scheduleBroadcastsLocked 发 Handler 过来的，
            // 这里就把正在执行的标志设置为 false 了。
            // Handler 处理了一个就可以让下一个在 Handler 的消息队列里面排队了
            if (fromMsg) {
                mBroadcastsScheduled = false;
            }    

            // 这里一个循环把并行广播列表中的广播记录全部处理了
            // First, deliver any non-serialized broadcasts right away.
            while (mParallelBroadcasts.size() > 0) { 
                r = mParallelBroadcasts.remove(0);
                r.dispatchTime = SystemClock.uptimeMillis();
                r.dispatchClockTime = System.currentTimeMillis();
                final int N = r.receivers.size();
                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG, "Processing parallel broadcast ["
                        + mQueueName + "] " + r);
                // 一个广播会有多个接收器（前面 AMS 收集的 List），一个一个的分发
                for (int i=0; i<N; i++) {
                    Object target = r.receivers.get(i);
                    if (DEBUG_BROADCAST)  Slog.v(TAG,
                            "Delivering non-ordered on [" + mQueueName + "] to registered "
                            + target + ": " + r);
                    // 分发广播给接收器
                    // 看样子在并行列表里面的必须是动态注册的，因为这里写死认为是 BroadcastFilter 了
                    // 静态注册的是 ResolveInfo
                    deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
                }    
                addBroadcastToHistoryLocked(r);
                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG, "Done with parallel broadcast ["
                        + mQueueName + "] " + r);
            }

... ...

}
```

我们接下去看 deliverToRegisteredReceiverLocked：

```java
    private final void deliverToRegisteredReceiverLocked(BroadcastRecord r,
            BroadcastFilter filter, boolean ordered) {
        // 权限检测，有些广播只能有权限的接收器（进程）才能接收的
        // 我们先不管这些
        boolean skip = false;
        if (filter.requiredPermission != null) {
            int perm = mService.checkComponentPermission(filter.requiredPermission,
                    r.callingPid, r.callingUid, -1, true);
            if (perm != PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, "Permission Denial: broadcasting "
                        + r.intent.toString()
                        + " from " + r.callerPackage + " (pid="
                        + r.callingPid + ", uid=" + r.callingUid + ")"
                        + " requires " + filter.requiredPermission
                        + " due to registered receiver " + filter);
                skip = true;
            }    
        }    
        if (!skip && r.requiredPermission != null) {
            int perm = mService.checkComponentPermission(r.requiredPermission,
                    filter.receiverList.pid, filter.receiverList.uid, -1, true);
            if (perm != PackageManager.PERMISSION_GRANTED) {
                Slog.w(TAG, "Permission Denial: receiving "
                        + r.intent.toString()
                        + " to " + filter.receiverList.app
                        + " (pid=" + filter.receiverList.pid
                        + ", uid=" + filter.receiverList.uid + ")"
                        + " requires " + r.requiredPermission
                        + " due to sender " + r.callerPackage
                        + " (uid " + r.callingUid + ")");
                skip = true;
            }    
        }   

        // 我们就当有权限的情况
        if (!skip) {
            // 串行广播好像会保持原来的什么状态，先不管这个先
            // If this is not being sent as an ordered broadcast, then we
            // don't want to touch the fields that keep track of the current
            // state of ordered broadcasts.
            if (ordered) {
                r.receiver = filter.receiverList.receiver.asBinder();
                r.curFilter = filter;
                filter.receiverList.curBroadcast = r; 
                r.state = BroadcastRecord.CALL_IN_RECEIVE;
                if (filter.receiverList.app != null) {
                    // Bump hosting application to no longer be in background
                    // scheduling class.  Note that we can't do that if there
                    // isn't an app...  but we can only be in that case for
                    // things that directly call the IActivityManager API, which
                    // are already core system stuff so don't matter for this.
                    r.curApp = filter.receiverList.app;
                    filter.receiverList.app.curReceiver = r;
                    mService.updateOomAdjLocked();
                }
            }
            try {
                if (DEBUG_BROADCAST_LIGHT) {
                    int seq = r.intent.getIntExtra("seq", -1);
                    Slog.i(TAG, "Delivering to " + filter
                            + " (seq=" + seq + "): " + r);
                }
                // 这个才是真正的执行操作
                performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                    new Intent(r.intent), r.resultCode, r.resultData,
                    r.resultExtras, r.ordered, r.initialSticky, r.userId);
                if (ordered) {
                    r.state = BroadcastRecord.CALL_DONE_RECEIVE;
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Failure sending broadcast " + r.intent, e);
                if (ordered) {
                    r.receiver = null;
                    r.curFilter = null;
                    filter.receiverList.curBroadcast = null;
                    if (filter.receiverList.app != null) {
                        filter.receiverList.app.curReceiver = null;
                    }
                }
            }
        }
    }
```

这个分发函数把前面的权限检测和那个串行处理忽略掉之后，就剩下 performReceiveLocked 这个处理了：

```java
    private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        // Send the intent to the receiver asynchronously using one-way binder calls.
        if (app != null && app.thread != null) {
            // If we have an app thread, do the call through that so it is
            // correctly ordered with other one-way calls.
            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                    data, extras, ordered, sticky, sendingUser);
        } else {    
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }   
    }
```

这里首先，我们复习下传过来的参数： ProcessRecord 是动态注册的时候构造 ReceiverList 保存的接收器注册者的进程记录信息；IIntentReceiver 这个东西也是在动态注册的时候构造 BroadcastFilter 保存的接收器注册者的 LoadedApk.ReceiverDispatcher.InnerReceiver 这个对象的 Bp 端（这个的 Bn 端就有接收器最终的 onReceiver 回调，忘记了去注册篇复习下）。还记得 [Android Binder 分析——普通服务 Binder 对象的传递](http://mingming-killer.diandian.com/post/2014-11-08/40063333232 "Android Binder 分析——普通服务 Binder 对象的传递") 这个里面分析的，绑定普通应用服务的要分应用进程是否已经启动的情况么，没错这里也是要分接收器的进程是否已经正在运行的情况的：

#### 接收器进程正在运行
这个就是上面 app != null $ app.thread != null 的情况。这个时候就可以直接调用 IApplicationThread 的 scheduleRegisteredReceiver 函数执行接收器函数。看这个名字就知道 IPC 调用了，这个是当然的，接收器在另外的进程里面（广播处理的是在 AMS 中，system_server 进程）。然后我们去 Bn 端看看（这个时候我们是在接收器的进程了，）：

```java
// ================== ActivityThread.java =======================

    // 看这个继承类的名字，没有用 aidl，Bp 和 Interface 实现都是自己写的
    private class ApplicationThread extends ApplicationThreadNative {

... ...

        // This function exists to make sure all receiver dispatching is
        // correctly ordered, since these are one-way calls and the binder driver
        // applies transaction ordering per object for such calls.
        public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser) throws RemoteException {
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }  

... ... 
                
    } 

```

IIntentReceiver 再经过 IPC 传到接收器进程，已经是 Bn 端了，我们终于可以去看注册篇里面那个 LoadedApk 的内部类的内部类的 InnerReceiver 了：

```java
        final static class InnerReceiver extends IIntentReceiver.Stub { 
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }
            public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                LoadedApk.ReceiverDispatcher rd = mDispatcher.get();
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = intent.getIntExtra("seq", -1); 
                    Slog.i(ActivityThread.TAG, "Receiving broadcast " + intent.getAction() + " seq=" + seq
                            + " to " + (rd != null ? rd.mReceiver : null));
                }
                // 还得绕一下
                if (rd != null) {              
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser); 
                } else {
                    // 看注释，可能在 AMS 分发广播给这个接收器之前，这个接收器就被注销了
                    // 所以这里发送完成消息，注意这个完成消息对于串行广播的处理很关键的
                    // The activity manager dispatched a broadcast to a registered
                    // receiver in this process, but before it could be delivered the
                    // receiver was unregistered.  Acknowledge the broadcast on its
                    // behalf so that the system's broadcast sequence can continue.
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing broadcast to unregistered receiver");
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    try {
                        if (extras != null) {          
                            extras.setAllowFds(false);     
                        }
                        mgr.finishReceiver(this, resultCode, data, extras, false);
                    } catch (RemoteException e) {  
                        Slog.w(ActivityThread.TAG, "Couldn't finish broadcast to unregistered receiver");
                    }
                }
            }
        }
```

前面的注册

未完待续 ... ...





