title: Android Broadcast 分析——发送、处理
date: 2015-01-22 10:15:16
updated: 2017-02-07 21:47:54
categories: [Android Framework]
tags: [android]
---

上一篇分析了广播的注册流程，这篇来分析下广播的发送、处理流程。这里为什么把发送和处理和来一起说咧，那是因为其实这是一个过程，发送接口里面差不多就是处理过程了。我们先照例把相关代码位置啰嗦一下（4.2.2）：

```bash
# Content 广播相关的代码
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/LoadedApk.java
frameworks/base/core/java/android/app/ActivityThread.java

frameworks/base/core/java/android/content/Intent.java
frameworks/base/core/java/android/content/IntentFilter.java
frameworks/base/core/java/android/content/BroadcastReceiver.java
frameworks/base/core/java/android/content/IIntentReceiver.aidl

# 广播解析相关代码
frameworks/base/services/java/com/android/server/IntentResolver.java

# AM 广播相关代码
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/java/com/android/server/am/RecevierList.java
frameworks/base/services/java/com/android/server/am/BroadcastQueue.java
frameworks/base/services/java/com/android/server/am/BroadcastFilter.java
frameworks/base/services/java/com/android/server/am/BroadcastRecord.java

# PM 广播相关代码
frameworks/base/services/java/com/android/server/pm/PackageManagerService.java
```

然后把处理流程贴张图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Broadcast-send/broadcast-send.png)
(这个图很夸张吧，拉一下本篇的滚动条就知道为啥这么夸张了；本来想分情况画几张的，画到后面还是合在一起了)

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


接下来说下广播处理流程。AMS 中处理广播的流程就是 broadcastIntentLocked 这个函数，这个函数也是非常长的（差不多6、7百行），我们分段慢慢来（会跳过一些非重要的部分）：

## a. 收集广播接收器

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

### a.a 收集静态注册接收器

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

构造完 R 之后，queryIntent 最后对匹配的 R 进行了一下排序：

```java
    @SuppressWarnings("unchecked")
    protected void sortResults(List<R> results) {
        Collections.sort(results, mResolvePrioritySorter);
    } 

    // 注释说得很清楚了，按照优先级进行排序
    // Sorts a List of IntentFilter objects into descending priority order.
    @SuppressWarnings("rawtypes")
    private static final Comparator mResolvePrioritySorter = new Comparator() {
        public int compare(Object o1, Object o2) {
            final int q1 = ((IntentFilter) o1).getPriority();
            final int q2 = ((IntentFilter) o2).getPriority();
            return (q1 > q2) ? -1 : ((q1 < q2) ? 1 : 0);
        }
    };
```

这个优先级在注册篇说过了，在 manifest 里面声明 receiver 的 filter 那里设置的（只有系统应用有权限设置）。

### a.b 收集动态注册接收器

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

## b. 分发广播给动态注册接收器

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

这里首先，我们复习下传过来的参数： ProcessRecord 是动态注册的时候构造 ReceiverList 保存的接收器注册者的进程记录信息；IIntentReceiver 这个东西也是在动态注册的时候构造 BroadcastFilter 保存的接收器注册者的 LoadedApk.ReceiverDispatcher.InnerReceiver 这个对象的 Bp 端（这个的 Bn 端就有接收器最终的 onReceiver 回调，忘记了去注册篇复习下）。这里虽然分了2种情况：

1. 接收器进程还在运行（app != null && app.thread != null）
2. 接收器进程已经挂了

我个人认为正常情况，只有会第一种，第二种是不会有的，就算有这次的广播也无法正常发送。为什么呢，下面来解释一下（这里我没做验证，是光看代码的，但是感觉应该没错）：

第一，如果发生了第二种情况，IIntentReceiver 这个接口的实现是用 aidl 弄的，自己去 out 下翻一下，会发现这个接口没有判断 Bn 不在的情况下，启动进程的情况，也就是说如果接收器进程挂了，调用这个 IPC 接口，根本不会重新启动接收器进程。那就是说这种情况下 IPC 通信是会失败的，因为 Bn 没了。

第二，其实正常情况下，上面那种情况不会出现。去注册篇仔细看下 AMS 的 registerReceiver 注册了一个 IIntentReceiver 的死亡通知回调：

<pre>
receiver.asBinder().linkToDeath(rl, 0);
</pre>

当动态动态的接收器进程挂了，会调用这个一个东西：

```java
class ReceiverList extends ArrayList<BroadcastFilter>
        implements IBinder.DeathRecipient {
    final ActivityManagerService owner;
    public final IIntentReceiver receiver;

... ...

    public void binderDied() {
        linkedToDeath = false;
        owner.unregisterReceiver(receiver);
    }

... ....
}
```

就相当于如果动态注册的接收器进程挂了，AMS 会自动注销它的，我们稍微看下 AMS 的 unregisterReceiver：

```java
    public void unregisterReceiver(IIntentReceiver receiver) {
        if (DEBUG_BROADCAST) Slog.v(TAG, "Unregister receiver: " + receiver);

        final long origId = Binder.clearCallingIdentity();
        try { 
            boolean doTrim = false;

            synchronized(this) {
                ReceiverList rl
                = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());
                if (rl != null) {
                    // 这里是后面串行广播处理相关的，这里可以不用管先
                    if (rl.curBroadcast != null) {
                        BroadcastRecord r = rl.curBroadcast;
                        final boolean doNext = finishReceiverLocked(
                                receiver.asBinder(), r.resultCode, r.resultData,
                                r.resultExtras, r.resultAbort, true);
                        if (doNext) {
                            doTrim = true; 
                            r.queue.processNextBroadcast(false);
                        }     
                    }     

                    if (rl.app != null) {
                        rl.app.receivers.remove(rl);
                    }     
                    // 接下去看下这个 remove 函数
                    removeReceiverLocked(rl);
                    if (rl.linkedToDeath) {
                        rl.linkedToDeath = false;
                        rl.receiver.asBinder().unlinkToDeath(rl, 0);
                    }     
                }     
            }     

            // If we actually concluded any broadcasts, we might now be able
            // to trim the recipients' apps from our working set
            if (doTrim) {
                trimApplications();
                return;
            }

        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }

    void removeReceiverLocked(ReceiverList rl) {
        mRegisteredReceivers.remove(rl.receiver.asBinder());
        int N = rl.size();
        for (int i=0; i<N; i++) {
            // 调用 IntentResolver 的 removeFilter 去删除
            // 和 addFilter 对应的，这里不多分析这个了，
            // 反正就是删掉之后，查询的时候就查不到这个接收器的 filter 了
            mReceiverResolver.removeFilter(rl.get(i));
        }
    }
```

所以说正常情况下，动态注册的接收器进程挂了，broadcastIntentLocked 的时候根本就查询不到有这个接收器的。所以第二种情况基本上不会发生吧。这里我们直接看第一种情况，就是接收器进程正在运行（这才是正常情况，要不然怎么叫动态注册咧）。这个时候就可以直接调用 IApplicationThread 的 scheduleRegisteredReceiver 函数执行接收器函数。看这个名字就知道 IPC 调用了，这个是当然的，接收器在另外的进程里面（广播处理的是在 AMS 中，system_server 进程）。然后我们去 Bn 端看看（这个时候我们是在接收器的进程了，）：

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

这里继续去看 LoadedApk.ReceiverDispatcher 的 performReceive：

```java
        public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            if (ActivityThread.DEBUG_BROADCAST) {
                int seq = intent.getIntExtra("seq", -1); 
                Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction() + " seq=" + seq
                        + " to " + mReceiver);         
            }
            // 这里 post 了一个 Runnable（Args 实现了 Runnable）
            Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);          
            if (!mActivityThread.post(args)) {
                if (mRegistered && ordered) {  
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing sync broadcast to " + mReceiver);
                    // 对于注册的接收器并且是串行广播的话
                    // 如果 post 失败，发送完成消息，前面说这个对于串行广播很关键的
                    args.sendFinished(mgr);        
                }
            }
        }
```

还记得注册篇中注册的接口时候有一个 Handler 的可选参数的么，那个是可以让调用者指定接收器运行的线程（Handler 所运行的线程，如果没指定的话，默认使用调用者所在进程的主线程）。这个 mActivityThread 就是保存了注册时候的那个 Handler：

<pre>
final Handler mActivityThread; 
</pre>

叫 mActivityThread 其实是一个 Handler，所以这里其实是**把接收器扔到注册的时候指定的线程去执行去了**。然后我们接下去看 Args 这个东西：

```java
        // 果然是实现了 Runnable
        final class Args extends BroadcastReceiver.PendingResult implements Runnable {
            private Intent mCurIntent;     
            private final boolean mOrdered;

            public Args(Intent intent, int resultCode, String resultData, Bundle resultExtras,
                    boolean ordered, boolean sticky, int sendingUser) {
                super(resultCode, resultData, resultExtras,
                        mRegistered ? TYPE_REGISTERED : TYPE_UNREGISTERED,
                        ordered, sticky, mIIntentReceiver.asBinder(), sendingUser);
                mCurIntent = intent;           
                mOrdered = ordered;            
            }
            
            public void run() {            
                // Args 是 LoadedApk.ReceiverDispatcher 的内部类
                // 这个 mReceiver 就是当初注册接收器的时候 new LoadedApk.ReceiverDispatcher 
                // 保存传过来的 BroadcastRecevier 对象
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered;
                
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = mCurIntent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Dispatching broadcast " + mCurIntent.getAction()
                            + " seq=" + seq + " to " + mReceiver);
                    Slog.i(ActivityThread.TAG, "  mRegistered=" + mRegistered 
                            + " mOrderedHint=" + ordered); 
                }
                
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                final Intent intent = mCurIntent;
                mCurIntent = null;             
                
                if (receiver == null || mForgotten) {
                    if (mRegistered && ordered) {
                        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                "Finishing null broadcast to " + mReceiver);
                    }
                    return;
                }

                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveReg");
                try {
                    // 暂时没明白这里保存接收器的 ClassLoader 有什么用
                    // 可能要 new 什么东西出来吧，这里先不管这么多了
                    ClassLoader cl =  mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    setExtrasClassLoader(cl);
                    // 广播好像可以带返回信息的，这里也先不管先
                    receiver.setPendingResult(this);
                    // 反正最后终于调用到接收器的 onReceive 了
                    receiver.onReceive(mContext, intent);
                } catch (Exception e) {
                    // 如果前面执行出了什么异常，记得要发送完成信息
                    if (mRegistered && ordered) {
                        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                "Finishing failed broadcast to " + mReceiver);
                        sendFinished(mgr);
                    }
                    if (mInstrumentation == null ||
                            !mInstrumentation.onException(mReceiver, e)) {
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                        throw new RuntimeException(
                            "Error receiving broadcast " + intent
                            + " in " + mReceiver, e);
                    } 
                }
                
                if (receiver.getPendingResult() != null) {
                    finish();
                }
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }               
        }
```

LoadedApk.ReceiverDispatcher 的 mReceiver 就是注册广播接收器的时候 new LoadedApk.ReceiverDispatcher 传进来那个 BroadcastReceiver 对象（忘记的回去注册篇看看）。这里在执行的线程中跑 run 函数，最终调用到了注册者实现的 BroadcastReceiver 的 onReceive 回调。到这里一个动态注册的广播接收器就算处理完成了。

这里回到前面 performReceive 的 mActivityThread.post 那里。前面说了现在是处理并行广播，怎么体现出来是并行的咧。再回去远一点，从接收器进程回到 AMS 的 processNextBroadcast 的那个 while 循环那里：

```java
    final void processNextBroadcast(boolean fromMsg) {
        synchronized(mService) {
            while (mParallelBroadcasts.size() > 0) { 
                for (int i=0; i<N; i++) {
                    deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
                }    
            }
... ...
    }
```

因为 performReceive 那里是 post 了一个 Runnable（LoadedApk.ReceiverDispatcher.Args）到一个 Handler 中，所以会马上返回（不会等待接收器的执行），所以 IPC 也会马上从接收器进程返回到 AMS 这边，所以 while（for）循环就会继续往下执行下一个接收器的处理。从精确的角度说虽然还是会有先后，但是广播这的东西本身不是什么太精准的东西，所以约等于是并行处理了。

这里我们从 processNextBroadcast 分段那里直接返回，因为后面是串行广播的处理（静态注册的接收器），然后我们就可以回到 AMS 的 broadcastIntentLocked 继续往下走了。

## c. 分发广播给静态注册接收器

我们接着往下看 broadcastIntentLocked，下面是这么一段：

```java
        // 这里是将前面还没处理的动态接收器和静态接收器合并为一个接收器列表
        // 这里是合并到 receivers（之前是收集了静态的接收器）
        // 这里知道为什么 receivers 要声明为 List 了，因为里面可能存2种类型的数据
        // Merge into one list.
        int ir = 0;
        if (receivers != null) {
            // 这段是说，如果一个应该刚安装上，还没被用户运行，那么将它静态注册的接收器排除掉
            // 这么做的目的是防止应用一刚装上就能够接收广播，然后被偷偷摸摸的启动起来
            // 可以预防一些恶意软件偷偷摸摸干坏事
            // A special case for PACKAGE_ADDED: do not allow the package
            // being added to see this broadcast.  This prevents them from
            // using this as a back door to get run as soon as they are
            // installed.  Maybe in the future we want to have a special install
            // broadcast or such for apps, but we'd like to deliberately make
            // this decision.
            String skipPackages[] = null;
            if (Intent.ACTION_PACKAGE_ADDED.equals(intent.getAction())
                    || Intent.ACTION_PACKAGE_RESTARTED.equals(intent.getAction())
                    || Intent.ACTION_PACKAGE_DATA_CLEARED.equals(intent.getAction())) {
                Uri data = intent.getData();
                if (data != null) {
                    String pkgName = data.getSchemeSpecificPart();
                    if (pkgName != null) {
                        skipPackages = new String[] { pkgName };
                    }
                }
            } else if (Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE.equals(intent.getAction())) {
                skipPackages = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
            }
            if (skipPackages != null && (skipPackages.length > 0)) {
                for (String skipPackage : skipPackages) {
                    if (skipPackage != null) {
                        int NT = receivers.size();
                        for (int it=0; it<NT; it++) {
                            ResolveInfo curt = (ResolveInfo)receivers.get(it);
                            if (curt.activityInfo.packageName.equals(skipPackage)) {
                                receivers.remove(it);
                                it--;
                                NT--;
                            }
                        }
                    }
                }
            }

            // 下面这段是取前面剩下的动态注册的接收器。为什么说剩下，因为前面如果动态注册的接收器处理了，
            // NR 是 0，registeredReceivers 也会变成 null。
            // 但是如果发送的是串行广播，那前面处理动态注册接收器的并行处理就不会执行，
            // 动态注册的接收器就只能留到下面和静态注册的接收一起串行处理进行。
            int NT = receivers != null ? receivers.size() : 0;
            int it = 0;
            ResolveInfo curt = null;
            BroadcastFilter curr = null;
            while (it < NT && ir < NR) {
                if (curt == null) {
                    curt = (ResolveInfo)receivers.get(it);
                }
                if (curr == null) {
                    curr = registeredReceivers.get(ir);
                }
                // 这里判断一下剩下的动态注册的接收器的优先级是否比静态注册的要高
                // 如果高的话，就插到 list 的前面， it 计数是从 0 开始的。
                if (curr.getPriority() >= curt.priority) {
                    // Insert this broadcast record into the final list.
                    receivers.add(it, curr);
                    ir++;
                    curr = null;
                    it++;
                    NT++;
                } else {
                    // Skip to the next ResolveInfo in the final list.
                    it++;
                    curt = null;
                }
            }
        }
        // 经过上面的筛选，剩下的动态注册的接收器优先级都是没静态注册的优先级高的，
        // 全部插到 list 的最后。
        while (ir < NR) {
            if (receivers == null) {
                receivers = new ArrayList();
            }
            receivers.add(registeredReceivers.get(ir));
            ir++;
        }
```

这段就是将剩下还没处理的动态注册的接收器合并到静态注册接收器的 list 里面去了，然后下面是 broadcastIntentLocked 的最后一段，串行处理静态注册接收器（包括强制发串行广播的动态注册的接收器）：

```java
    private final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle map, String requiredPermission,
            boolean ordered, boolean sticky, int callingPid, int callingUid,
            int userId) {
... ...
        
        // 这段代码和前面处理动态注册接收器那几乎是一样的 ... ...
        if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {
            BroadcastQueue queue = broadcastQueueForIntent(intent);
            // 区别一：这里传给 BroadcastRecord 是 receivers（合并过的接收器列表，可以说是静态接收器吧）
            // 前面是 registeredReceivers，动态注册接收器
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, requiredPermission,
                    receivers, resultTo, resultCode, resultData, map, ordered,
                    sticky, false, userId);
            if (DEBUG_BROADCAST) Slog.v(
                    TAG, "Enqueueing ordered broadcast " + r
                    + ": prev had " + queue.mOrderedBroadcasts.size());
            if (DEBUG_BROADCAST) {
                int seq = r.intent.getIntExtra("seq", -1);
                Slog.i(TAG, "Enqueueing broadcast " + r.intent.getAction() + " seq=" + seq);
            }
            boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
            if (!replaced) {
                // 区别二：前面是插入并行队列，这里是串行队列
                queue.enqueueOrderedBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
        }

        return ActivityManager.BROADCAST_SUCCESS;
    }
```

这里唯一和前面的区别就是处理的接收器的列表不一样，然后在 BroadcastQueue 中排队的队列也不一样。我们可以直接跳到前面的 processNextBroadcast 返回那里（前面那些流程都是一样的）：

```java
            // Now take care of the next serialized one...

            // 看到这个 Pending 命名的是不是有点眼熟，去看下 Binder 普通服务对象篇，
            // 目标进程没启动，就是使用了一个叫 mPendingService 的来保存需要等待目标
            // 进程启动来做一些事情的。这里也是一样的，不过这里留到后面去分析
            // If we are waiting for a process to come up to handle the next
            // broadcast, then do nothing at this point.  Just in case, we
            // check that the process we're waiting for still exists.
            if (mPendingBroadcast != null) {
                if (DEBUG_BROADCAST_LIGHT) {
                    Slog.v(TAG, "processNextBroadcast ["
                            + mQueueName + "]: waiting for "
                            + mPendingBroadcast.curApp);
                }

                boolean isDead;
                synchronized (mService.mPidsSelfLocked) {
                    isDead = (mService.mPidsSelfLocked.get(
                            mPendingBroadcast.curApp.pid) == null);
                }
                if (!isDead) {
                    // It's still alive, so keep waiting
                    return;
                } else {
                    Slog.w(TAG, "pending app  ["
                            + mQueueName + "]" + mPendingBroadcast.curApp
                            + " died before responding to broadcast");
                    mPendingBroadcast.state = BroadcastRecord.IDLE;
                    mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                    mPendingBroadcast = null;
                }
            }

            boolean looped = false;
            
            do { 
                // 前面处理并行的不说后面可以直接返回么，就是这个判断了，
                // 如果串行队列里没数据，就直接 return，不过现在有数据了。
                if (mOrderedBroadcasts.size() == 0) { 
                    // No more broadcasts pending, so all done!
                    mService.scheduleAppGcsLocked();
                    if (looped) {
                        // If we had finished the last ordered broadcast, then
                        // make sure all processes have correct oom and sched
                        // adjustments.
                        mService.updateOomAdjLocked();
                    }    
                    return;
                }    
                r = mOrderedBroadcasts.get(0);
                boolean forceReceive = false;

                // 下面这段我们留到后面讲串行广播的实现的时候再说
                ... ...
                
            } while (r == null);
```

上面的简单过过就行，一些东西我们这里不去深究，继续看下面：

```java
            // 注意这个 nextReceiver++，后面会知道这个的作用
            // Get the next receiver...    
            int recIdx = r.nextReceiver++; 

            // 这里设置接收器接收到广播的时间为当前的系统时间，
            // 然后设置一个超时等待（为什么要设置超时等待后面再说）
            // Keep track of when this receiver started, and make sure there
            // is a timeout message pending to kill it if need be.
            r.receiverTime = SystemClock.uptimeMillis();
            if (recIdx == 0) {
                r.dispatchTime = r.receiverTime;
                r.dispatchClockTime = System.currentTimeMillis();
                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG, "Processing ordered broadcast ["
                        + mQueueName + "] " + r);           
            }
            // 如果没有设置超时的话，设置一个超时（这个超时的作用后面再说）
            if (! mPendingBroadcastTimeoutMessage) {
                long timeoutTime = r.receiverTime + mTimeoutPeriod;
                if (DEBUG_BROADCAST) Slog.v(TAG,
                        "Submitting BROADCAST_TIMEOUT_MSG ["
                        + mQueueName + "] for " + r + " at " + timeoutTime);
                setBroadcastTimeoutLocked(timeoutTime);
            }

            // 下面这种情况很简单，其实就是前面的动态注册接收器推迟到这里处理而已。
            // (BroadcastFilter 是动态注册的类型 AMS 里面的，ResolveInfo 是静态的， PMS 里面的)
            // 如果发送串行广播，动态注册接收器就无法并行执行，所以放到这里串行执行。
            Object nextReceiver = r.receivers.get(recIdx);
            if (nextReceiver instanceof BroadcastFilter) {
                // Simple case: this is a registered receiver who gets
                // a direct call.              
                BroadcastFilter filter = (BroadcastFilter)nextReceiver; 
                if (DEBUG_BROADCAST)  Slog.v(TAG, 
                        "Delivering ordered ["         
                        + mQueueName + "] to registered "
                        + filter + ": " + r);                 
                deliverToRegisteredReceiverLocked(r, filter, r.ordered);
                if (r.receiver == null || !r.ordered) {
                    // The receiver has already finished, so schedule to
                    // process the next one.       
                    if (DEBUG_BROADCAST) Slog.v(TAG, "Quick finishing ["
                            + mQueueName + "]: ordered="   
                            + r.ordered + " receiver=" + r.receiver);
                    r.state = BroadcastRecord.IDLE;
                    scheduleBroadcastsLocked();
                }
                return;
            }
```

到这里其实都还不是静态注册接收器的处理，只是把前面动态注册的接收器串行处理了而已，流程前面说过了。现在我们继续往下：

```java
            // Hard case: need to instantiate the receiver, possibly
            // starting its application process to host it.

            // 下面的是静态注册的接收器了，ResolverInfo PMS 中的
            ResolveInfo info =
                (ResolveInfo)nextReceiver;
            ComponentName component = new ComponentName(
                    info.activityInfo.applicationInfo.packageName,
                    info.activityInfo.name);

            boolean skip = false;
            int perm = mService.checkComponentPermission(info.activityInfo.permission,
                    r.callingPid, r.callingUid, info.activityInfo.applicationInfo.uid,
                    info.activityInfo.exported);
            if (perm != PackageManager.PERMISSION_GRANTED) {
                if (!info.activityInfo.exported) {
                    Slog.w(TAG, "Permission Denial: broadcasting "
                            + r.intent.toString()
                            + " from " + r.callerPackage + " (pid=" + r.callingPid
                            + ", uid=" + r.callingUid + ")"
                            + " is not exported from uid " + info.activityInfo.applicationInfo.uid
                            + " due to receiver " + component.flattenToShortString());
                } else {
                    Slog.w(TAG, "Permission Denial: broadcasting "
                            + r.intent.toString()
                            + " from " + r.callerPackage + " (pid=" + r.callingPid
                            + ", uid=" + r.callingUid + ")"
                            + " requires " + info.activityInfo.permission
                            + " due to receiver " + component.flattenToShortString());
                }
                skip = true;
            }
            if (info.activityInfo.applicationInfo.uid != Process.SYSTEM_UID &&
                r.requiredPermission != null) {
                try {
                    perm = AppGlobals.getPackageManager().
                            checkPermission(r.requiredPermission,
                                    info.activityInfo.applicationInfo.packageName);
                } catch (RemoteException e) {
                    perm = PackageManager.PERMISSION_DENIED;
                }
                if (perm != PackageManager.PERMISSION_GRANTED) {
                    Slog.w(TAG, "Permission Denial: receiving "
                            + r.intent + " to "
                            + component.flattenToShortString()
                            + " requires " + r.requiredPermission
                            + " due to sender " + r.callerPackage
                            + " (uid " + r.callingUid + ")");
                    skip = true;
                }
            }
            boolean isSingleton = false;
            try {
                isSingleton = mService.isSingleton(info.activityInfo.processName,
                        info.activityInfo.applicationInfo,
                        info.activityInfo.name, info.activityInfo.flags);
            } catch (SecurityException e) {
                Slog.w(TAG, e.getMessage());
                skip = true;
            }
            if ((info.activityInfo.flags&ActivityInfo.FLAG_SINGLE_USER) != 0) {
                if (ActivityManager.checkUidPermission(
                        android.Manifest.permission.INTERACT_ACROSS_USERS,
                        info.activityInfo.applicationInfo.uid)
                                != PackageManager.PERMISSION_GRANTED) {
                    Slog.w(TAG, "Permission Denial: Receiver " + component.flattenToShortString()
                            + " requests FLAG_SINGLE_USER, but app does not hold "
                            + android.Manifest.permission.INTERACT_ACROSS_USERS);
                    skip = true;
                }
            }
            if (r.curApp != null && r.curApp.crashing) {
                // If the target process is crashing, just skip it.
                if (DEBUG_BROADCAST)  Slog.v(TAG,
                        "Skipping deliver ordered ["
                        + mQueueName + "] " + r + " to " + r.curApp
                        + ": process crashing");
                skip = true;
            }

            if (skip) {
                if (DEBUG_BROADCAST)  Slog.v(TAG,
                        "Skipping delivery of ordered ["
                        + mQueueName + "] " + r + " for whatever reason");
                r.receiver = null;
                r.curFilter = null;
                r.state = BroadcastRecord.IDLE;
                scheduleBroadcastsLocked();
                return;
            }
```

上面这一段我们也可以简单略过，就是权限检测，检测当前这个接收器有没有权限处理这次的广播，如果没有就忽略这个接收器；还有判断了接收器所在的进程是不是挂掉了，挂掉了也忽略这个接收器。我们接下来继续：

```java
            // 注意一下，这里处理的时候把状态设置为了 APP_RECEIVE
            r.state = BroadcastRecord.APP_RECEIVE;
            // 获取接收器进程信息
            String targetProcess = info.activityInfo.processName;
            r.curComponent = component;
            if (r.callingUid != Process.SYSTEM_UID && isSingleton) {
                info.activityInfo = mService.getActivityInfoForUser(info.activityInfo, 0);
            }
            // 这个 info 就是 AMS 从 PMS 那查询到匹配的静态注册的接收器的数据结构 ResolveInfo
            // 把这个关键信息保存到 BroadcastRecord 的 curReceiver 中
            r.curReceiver = info.activityInfo;
            if (DEBUG_MU && r.callingUid > UserHandle.PER_USER_RANGE) {
                Slog.v(TAG_MU, "Updated broadcast record activity info for secondary user, "
                        + info.activityInfo + ", callingUid = " + r.callingUid + ", uid = "
                        + info.activityInfo.applicationInfo.uid);
            }

            // 这里估计是禁止去把接收器的 apk 设置为 stop 状态吧
            // Broadcast is being executed, its package can't be stopped.
            try {
                AppGlobals.getPackageManager().setPackageStoppedState(
                        r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid));
            } catch (RemoteException e) {
            } catch (IllegalArgumentException e) {
                Slog.w(TAG, "Failed trying to unstop package "
                        + r.curComponent.getPackageName() + ": " + e);
            }
```

下面就要分2种情况来讨论。还记得 [Android Binder 分析——普通服务 Binder 对象的传递](http://light3moon.com/2015/01/28/Android Binder 分析——普通服务 Binder 对象的传递 "Android Binder 分析——普通服务 Binder 对象的传递")  中分了3种情况来讨论启动普通应用的服务，这里也是同样的道理。要分为接收器的进程是否已经启动在讨论：

### 接收器进程已经启动

我们先来看简单的情况，接收器的进程已经启动。这种情况不需要等待接收器进程启动，可以直接发起 IPC 调用，在接收器进程中跑处理广播的回调：

```java
            // 取接收器进程的记录（这个 ProcessRecord 经过 Binder 篇应该不陌生了）
            // Is this receiver's application already running?
            ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
                    info.activityInfo.applicationInfo.uid);
            // 
            if (app != null && app.thread != null) {
                try {
                    // 这个 addPackage 有啥用我们先不管
                    app.addPackage(info.activityInfo.packageName);
                    // 到下面这个函数了，然后后面直接返回
                    processCurBroadcastLocked(r, app);
                    return;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when sending broadcast to "
                          + r.curComponent, e);
                } catch (RuntimeException e) {
                    // 注意下这里的出错处理也是比较关键的，我们后面再说
                    Log.wtf(TAG, "Failed sending broadcast to "
                            + r.curComponent + " with " + r.intent, e);
                    // If some unexpected exception happened, just skip
                    // this broadcast.  At this point we are not in the call
                    // from a client, so throwing an exception out from here
                    // will crash the entire system instead of just whoever
                    // sent the broadcast.
                    logBroadcastReceiverDiscardLocked(r);
                    finishReceiverLocked(r, r.resultCode, r.resultData,
                            r.resultExtras, r.resultAbort, true);
                    scheduleBroadcastsLocked();
                    // We need to reset the state if we failed to start the receiver.
                    r.state = BroadcastRecord.IDLE;
                    return;
                }

                // If a dead object exception was thrown -- fall through to
                // restart the application.
            }
```

我们接着去看 processCurBroadcastLocked：

```java
    private final void processCurBroadcastLocked(BroadcastRecord r,
            ProcessRecord app) throws RemoteException {
        if (DEBUG_BROADCAST)  Slog.v(TAG, 
                "Process cur broadcast " + r + " for app " + app);
        if (app.thread == null) {      
            throw new RemoteException();   
        }
        // 取得接收器进程 IBinder 对象
        r.receiver = app.thread.asBinder();
        r.curApp = app;
        app.curReceiver = r;
        mService.updateLruProcessLocked(app, true);

        // Tell the application to launch this receiver.
        r.intent.setComponent(r.curComponent);

        boolean started = false;       
        try {
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG,
                    "Delivering to component " + r.curComponent
                    + ": " + r);                                    
            mService.ensurePackageDexOpt(r.intent.getComponent().getPackageName());
            // 这个和前面动态注册的调用的 IApplicationThread 接口有点像，但是不是同一个
            // （动态注册调用的是 scheduleRegisteredReceiver）
            app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
                    mService.compatibilityInfoForPackageLocked(r.curReceiver.applicationInfo),
                    r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId);
            if (DEBUG_BROADCAST)  Slog.v(TAG, 
                    "Process cur broadcast " + r + " DELIVERED for app " + app);
            started = true;
        } finally {
            if (!started) {
                if (DEBUG_BROADCAST)  Slog.v(TAG, 
                        "Process cur broadcast " + r + ": NOT STARTED!");
                r.receiver = null;             
                r.curApp = null;               
                app.curReceiver = null;        
            }
        }
    }
```

和前面动态注册的一样，经过 IPC 调用，我们得跑到接收器的进程中去了，还是 ActivityThread.java：

```java
public final class ActivityThread {

... ...
    
    private class ApplicationThread extends ApplicationThreadNative {

... ...

        public final void scheduleReceiver(Intent intent, ActivityInfo info,
                CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
                boolean sync, int sendingUser) {
            // 这里的 ActivityInfo 是 AMS 那里传过来的，是 AMS 从 PMS 那里查询到的
            // （忘记了的去看看前面的代码）
            // 然后 new 了一个 ReceiverData 出来，把这个 info 赋值给了 ReceiverData
            ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
                    sync, false, mAppThread.asBinder(), sendingUser);
            r.info = info;
            r.compatInfo = compatInfo;     
            queueOrSendMessage(H.RECEIVER, r);
        }
... ... 
                
    }
... ...
}
```

这里其实就是用了一个 Handler 又绕了半天，最终的处理函数是 handleReceiver： 

```java
public final class ActivityThread {

... ...

    final H mH = new H();
    
... ...

    // if the thread hasn't started yet, we don't have the handler, so just
    // save the messages until we're ready.
    private void queueOrSendMessage(int what, Object obj) {
        queueOrSendMessage(what, obj, 0, 0);
    }
            
    private void queueOrSendMessage(int what, Object obj, int arg1, int arg2) {
        synchronized (this) {
            if (DEBUG_MESSAGES) Slog.v(
                TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                + ": " + arg1 + " / " + obj);
            Message msg = Message.obtain();
            msg.what = what;
            msg.obj = obj;
            msg.arg1 = arg1;
            msg.arg2 = arg2;
            mH.sendMessage(msg);
        }   
    }

... ...

    private class H extends Handler {
... ...
        public static final int RECEIVER                = 113; 
        public static final int CREATE_SERVICE          = 114; 
... ...

        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
... ...
                case RECEIVER:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveComp");
                    handleReceiver((ReceiverData)msg.obj);
                    maybeSnapshot();
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
... ...
            }
            if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
        }
... ...
    }

... ...

    private void handleReceiver(ReceiverData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        String component = data.intent.getComponent().getClassName();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        IActivityManager mgr = ActivityManagerNative.getDefault();

        BroadcastReceiver receiver;    
        try {
            // 这里的 ClassLoader 是用来 new BroadcastReceiver 用的
            // 为什么前面动态注册的接收器通过 IPC 传递过来，然后直接调用，这里的需要 new
            // 那是因为动态注册的是把 BroadcastReceiver 保存在了 ReceiverDispatcher 中，
            // 然后 AMS IPC 调到接收器进程这边直接从 ReceiverDispatcher 中取的。
            // 这里静态注册的，注册进程这边就没有 BroadcastRecevier，所以现在要 new 出来
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            data.intent.setExtrasClassLoader(cl);
            data.setExtrasClassLoader(cl); 
            receiver = (BroadcastReceiver)cl.loadClass(component).newInstance();
        } catch (Exception e) {        
            if (DEBUG_BROADCAST) Slog.i(TAG,
                    "Finishing failed broadcast to " + data.intent.getComponent()); 
            // 注意出错了，又发送 finish 消息
            data.sendFinished(mgr);        
            throw new RuntimeException(    
                "Unable to instantiate receiver " + component
                + ": " + e.toString(), e);     
        }

        try {
            Application app = packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(         
                TAG, "Performing receive of " + data.intent
                + ": app=" + app               
                + ", appName=" + app.getPackageName()
                + ", pkg=" + packageInfo.getPackageName() 
                + ", comp=" + data.intent.getComponent().toShortString()
                + ", dir=" + packageInfo.getAppDir());

            ContextImpl context = (ContextImpl)app.getBaseContext();
            sCurrentBroadcastIntent.set(data.intent);
            receiver.setPendingResult(data);
            // 这里也调用到 BroadcastReceiver 的 onReceive 回调了
            receiver.onReceive(context.getReceiverRestrictedContext(),
                    data.intent);                  
        } catch (Exception e) {        
            if (DEBUG_BROADCAST) Slog.i(TAG,
                    "Finishing failed broadcast to " + data.intent.getComponent());
            // 出错还是会发送 finish 消息
            data.sendFinished(mgr);
            if (!mInstrumentation.onException(receiver, e)) {
                throw new RuntimeException(
                    "Unable to start receiver " + component
                    + ": " + e.toString(), e);
            }
        } finally {
            sCurrentBroadcastIntent.set(null);
        }

        if (receiver.getPendingResult() != null) {
            // 这里很关键，有一个 finish 的调用
            data.finish();
        }
    }

... ...
}
```

上面的处理最终调到 onReceiver 了。但是想想看，还是有一个地方没搞明白。前面说了静态注册的接收器是串行执行广播处理的，就是等到当前的那个执行完，才能执行下一个。好了，现在我们来解释下这个串行执行是怎么实现的。回到 AMS 那边的 processNextBroadcast 调用 processCurBroadcastLocked 那里：

```java
                try {
                    app.addPackage(info.activityInfo.packageName);
                    processCurBroadcastLocked(r, app);
                    return;
                } catch
```

这里就是刚刚我们上面分析的流程，IPC 调用到接收器进程的 onReceive 回调。然后这里 processNextBroadcast 就返回了。AMS 执行 BroadcastQueue.scheduleBroadcastsLocked() 早就返回了，因为 BroadcastQueue 里面是发到 Handler 里面处理 processNextBroadcast 的，然后 processNextBroadcast 结束后， AMS 好像这次广播处理就结束了。可是这才执行了第一个静态注册的接收器而已啊。这个时候去看接收器 handleReceiver 最后有一个：

<pre>
data.finish()
</pre>

这个 finish 是要等到 onReceive 执行完才会调用的（也就是说这个接收器处理完了）。前面说了这个 finish 很关键的，我们来看看 finish 里面做了什么事情。在这里之前得先看看 ReceiverData 这个对象：

```java
    static final class ReceiverData extends BroadcastReceiver.PendingResult {
        public ReceiverData(Intent intent, int resultCode, String resultData, Bundle resultExtras,
                boolean ordered, boolean sticky, IBinder token, int sendingUser) {
            // 注意这里传给父类的是 TYPE_COMPONENT
            super(resultCode, resultData, resultExtras, TYPE_COMPONENT, ordered, sticky,
                    token, sendingUser);           
            this.intent = intent;          
        }

        Intent intent;
        ActivityInfo info;
        CompatibilityInfo compatInfo;  
        public String toString() {     
            return "ReceiverData{intent=" + intent + " packageName=" +
                    info.packageName + " resultCode=" + getResultCode()
                    + " resultData=" + getResultData() + " resultExtras="
                    + getResultExtras(false) + "}";
        }
    }
```

这个 ReceiverData 是前面 scheduleReceiver 那里 new 出来的，它是继承自 BroadcastReceiver.PendingResult，这得去父类里面去看看：

```java
        /*
         * Finish the broadcast.  The current result will be sent and the
         * next broadcast will proceed.
         */
        public final void finish() {
            if (mType == TYPE_COMPONENT) {
                // 这里走的是这里
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                // 我们先不管下面这个 PendingWork
                if (QueuedWork.hasPendingWork()) {
                    // If this is a broadcast component, we need to make sure any
                    // queued work is complete before telling AM we are done, so
                    // we don't have our process killed before that.  We now know
                    // there is pending work; put another piece of work at the end
                    // of the list to finish the broadcast, so we don't block this
                    // thread (which may be the main thread) to have it finished.
                    //  
                    // Note that we don't need to use QueuedWork.add() with the
                    // runnable, since we know the AM is waiting for us until the
                    // executor gets to it.
                    QueuedWork.singleThreadExecutor().execute( new Runnable() {
                        @Override public void run() {
                            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                    "Finishing broadcast after work to component " + mToken);
                            Slog.i(ActivityThread.TAG,
                                    "Finishing broadcast after work to component " + mToken);
                            sendFinished(mgr);
                        }   
                    }); 
                } else {
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing broadcast to component " + mToken);
                    Slog.i(ActivityThread.TAG,
                            "Finishing broadcast to component " + mToken);
                    // 这里最后调用这个函数
                    sendFinished(mgr);
                }
            } else if (mOrderedHint && mType != TYPE_UNREGISTERED) {
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                        "Finishing broadcast to " + mToken);
                Slog.i(ActivityThread.TAG,
                        "Finishing broadcast to " + mToken);
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                sendFinished(mgr);
            }
        }
```

继续去看 sendFinished：

```java
        /* @hide */
        public void sendFinished(IActivityManager am) {
            synchronized (this) {
                if (mFinished) {
                    throw new IllegalStateException("Broadcast already finished");
                }
                mFinished = true;

                try {
                    if (mResultExtras != null) {
                        mResultExtras.setAllowFds(false);
                    }
                    // 这个 mOrderedHint 如果是串行广播就是 true，这里是 true
                    if (mOrderedHint) {
                        // 最后这里果然调用 AMS 里面去了
                        am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras,
                                mAbortBroadcast);
                    } else {
                        // This broadcast was sent to a component; it is not ordered,
                        // but we still need to tell the activity manager we are done.
                        am.finishReceiver(mToken, 0, null, null, false);
                    }
                } catch (RemoteException ex) {
                }
            }
        }
```

好，这里接收器进程处理完广播之后，finish 最后有调回 AMS 里面，看到这里是不是猜到了什么呢。对的，没错，串行广播一个接一个的执行是**靠前一个接收器执行完，通知 AMS 把广播发给下一个接收器接着处理来实现的**。虽然猜到了，但是我们还是继续去 AMS 里面看完流程：

```java
    public void finishReceiver(IBinder who, int resultCode, String resultData,
            Bundle resultExtras, boolean resultAbort) {
        if (DEBUG_BROADCAST) Slog.v(TAG, "Finish receiver: " + who);
    
        // Refuse possible leaked file descriptors
        if (resultExtras != null && resultExtras.hasFileDescriptors()) {  
            throw new IllegalArgumentException("File descriptors passed in Bundle");
        }   
        
        final long origId = Binder.clearCallingIdentity();
        try {
            boolean doNext = false;
            BroadcastRecord r = null;

            synchronized(this) {
                // 这里先去取上次处理完的 BroadcastRecord
                r = broadcastRecordForReceiverLocked(who);
                if (r != null) {
                    // 这里先调用 BroadcastQueue 的 finishReceiverLocked
                    // 来判断是否需要继续往下处理
                    doNext = r.queue.finishReceiverLocked(r, resultCode,
                        resultData, resultExtras, resultAbort, true);
                }
            }

            // 如果需要继续处理的话，就调用 BroadcastQueue 的 processNextBroadcast
            if (doNext) {
                // 注意这里参数是 false
                r.queue.processNextBroadcast(false);
            }
            trimApplications();
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

我们先来看看怎么取上次处理完成的 BroadcastRecord 的：

```java
    BroadcastRecord broadcastRecordForReceiverLocked(IBinder receiver) {
        for (BroadcastQueue queue : mBroadcastQueues) {
            BroadcastRecord r = queue.getMatchingOrderedReceiver(receiver);
            if (r != null) {
                return r;
            }
        }
        return null;
    } 

// ========== BroadcastQueue.java ===============

    public BroadcastRecord getMatchingOrderedReceiver(IBinder receiver) {
        // 这里只认串行广播的（并行的并不需要这种处理）
        if (mOrderedBroadcasts.size() > 0) {
            // 这里只认第一个（上一个处理的就是第一个）
            final BroadcastRecord r = mOrderedBroadcasts.get(0);
            // 还是要判断下 IBinder 对象是不是同一个
            if (r != null && r.receiver == receiver) {
                return r;
            }
        }
        return null;
    }
```

这里取到了上一个接收器的 BroadcastRecord，接着去 BroadcastQueue 中看看 finishReceiverLocked：

```java
    public boolean finishReceiverLocked(BroadcastRecord r, int resultCode,
            String resultData, Bundle resultExtras, boolean resultAbort,
            boolean explicit) {   
        // 这里取上一个接收器的 state         
        int state = r.state;
        // 然后把 state 设置为 IDLE 状态
        r.state = BroadcastRecord.IDLE;
        // 正常来说上一个的 state 应该是 APP_RECEIVE 的
        // 如果不是的话，说明有异常，打印一下警告
        if (state == BroadcastRecord.IDLE) {
            if (explicit) {
                Slog.w(TAG, "finishReceiver [" + mQueueName + "] called but state is IDLE"); 
            }
        }
        // 然后下面基本上都设置为 null
        r.receiver = null;
        r.intent.setComponent(null);   
        if (r.curApp != null) {        
            r.curApp.curReceiver = null;   
        }
        if (r.curFilter != null) {     
            r.curFilter.receiverList.curBroadcast = null;
        }
        r.curFilter = null;
        r.curApp = null;
        r.curComponent = null;
        r.curReceiver = null;
        mPendingBroadcast = null;      

        r.resultCode = resultCode;
        r.resultData = resultData;
        r.resultExtras = resultExtras;
        r.resultAbort = resultAbort;

        // 最后这里正常来说上一个 state 是 APP_RECEIVE 的，所以一般是返回 true 的
        // We will process the next receiver right now if this is finishing
        // an app receiver (which is always asynchronous) or after we have
        // come back from calling a receiver.
        return state == BroadcastRecord.APP_RECEIVE
                || state == BroadcastRecord.CALL_DONE_RECEIVE;
    }
```

这个 BroadcastRecord 的 state，待会我们再讨论。这里正常 finishReceiverLocked 返回 true，就意味着 AMS 会继续调用 BroadcastQueue 的 processNextBroadcast 继续处理。这个 processNextBroadcast 我们前面分析了好久了，不过这里和前面不太一样，首先传过去的参数为 false：

```java
    final void processNextBroadcast(boolean fromMsg) {
        synchronized(mService) {       
            BroadcastRecord r;

... ...

            // false 说明不会强制把这个标志设置为 false
            // 也就是 BroadcastQueue 会继续保持正在执行广播的状态
            if (fromMsg) {
                mBroadcastsScheduled = false;  
            }

... ...

            // Now take care of the next serialized one...

... ...

            // 现在我们可以来说说前面略过的这一段了
            do {
                // 串行广播列表 size 为0，说明已经没有匹配的接收器了
                // 可以结束本次广播处理了， return 返回。 
                if (mOrderedBroadcasts.size() == 0) { 
                    // No more broadcasts pending, so all done!
                    mService.scheduleAppGcsLocked();
                    if (looped) {
                        // If we had finished the last ordered broadcast, then
                        // make sure all processes have correct oom and sched
                        // adjustments.
                        mService.updateOomAdjLocked();
                    }    
                    return;
                }
                // 这里取到的还是上一个处理完的接收器的 BroadcastRecord
                r = mOrderedBroadcasts.get(0);
                boolean forceReceive = false;

                // 下面这里是判断下这个接收器是不是超时了。
                // dispatchTime 前面有一个地方记录了广播发到这个接收器的时间。
                // 然后拿之前记录的时候和现在对比，看是不是超过了特定的时间，
                // 是的话就执行广播超时处理
                // Ensure that even if something goes awry with the timeout
                // detection, we catch "hung" broadcasts here, discard them,
                // and continue to make progress.
                //   
                // This is only done if the system is ready so that PRE_BOOT_COMPLETED
                // receivers don't get executed with timeouts. They're intended for
                // one time heavy lifting after system upgrades and can take
                // significant amounts of time.
                int numReceivers = (r.receivers != null) ? r.receivers.size() : 0; 
                if (mService.mProcessesReady && r.dispatchTime > 0) { 
                    long now = SystemClock.uptimeMillis();
                    if ((numReceivers > 0) &&
                            (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                        Slog.w(TAG, "Hung broadcast ["
                                + mQueueName + "] discarded after timeout failure:"
                                + " now=" + now
                                + " dispatchTime=" + r.dispatchTime
                                + " startTime=" + r.receiverTime
                                + " intent=" + r.intent
                                + " numReceivers=" + numReceivers
                                + " nextReceiver=" + r.nextReceiver
                                + " state=" + r.state);

                        broadcastTimeoutLocked(false); // forcibly finish this broadcast
                        forceReceive = true;
                        r.state = BroadcastRecord.IDLE;
                    }
                }
                
                // BroadcastRecord 初始 state 是 IDLE，后面接到广播的时候会变成 APP_RECEIVE
                // 最后完成的时候（finishReceiver）又会变成 IDLE。
                // 这里如果不是 IDLE 的话，说明状态异常
                if (r.state != BroadcastRecord.IDLE) {
                    if (DEBUG_BROADCAST) Slog.d(TAG,
                            "processNextBroadcast("
                            + mQueueName + ") called when not idle (state="
                            + r.state + ")");
                    return;
                }

                // 这里是判断的关键，numReceivers 上面取的这个 BroadcastRecord 的 receiver list 有多少个 receiver。
                // BroadcastRecord 的 nextReceiver 初始值是 0，下面处理一个 receiver 会加 1
                // 然后我们这里以 receiver list 1 为例子，那么这个 BroadcastRecord 处理完一次后，
                // r.nextReceiver >= numReceivers 这个判断就会成立，就会跑下面的代码
                if (r.receivers == null || r.nextReceiver >= numReceivers
                        || r.resultAbort || forceReceive) {
                    // No more receivers for this broadcast!  Send the final
                    // result if requested...
                    // 我们先不管带返回信息的广播
                    if (r.resultTo != null) {
                        try {
                            if (DEBUG_BROADCAST) {
                                int seq = r.intent.getIntExtra("seq", -1);
                                Slog.i(TAG, "Finishing broadcast ["
                                        + mQueueName + "] " + r.intent.getAction()
                                        + " seq=" + seq + " app=" + r.callerApp);
                            }
                            performReceiveLocked(r.callerApp, r.resultTo,
                                new Intent(r.intent), r.resultCode,
                                r.resultData, r.resultExtras, false, false, r.userId);
                            // Set this to null so that the reference
                            // (local and remote) isnt kept in the mBroadcastHistory.
                            r.resultTo = null;
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Failure ["
                                    + mQueueName + "] sending broadcast result of "
                                    + r.intent, e);
                        }
                    }

                    // 这里上一个接收器算是正式处理完了，所以把上次设置的超时取消掉（到下一个接收器重新设一个）
                    if (DEBUG_BROADCAST) Slog.v(TAG, "Cancelling BROADCAST_TIMEOUT_MSG");
                    cancelBroadcastTimeoutLocked();
                    
                    if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG, "Finished with ordered broadcast "
                            + r);

                    // ... and on to the next...
                    addBroadcastToHistoryLocked(r);
                    // 上一个接收器处理完了，所以就把它从串行列表中删掉
                    mOrderedBroadcasts.remove(0);
                    r = null;
                    looped = true;
                    // 结束本循环，去前面取串行队列的下一个 BroadcastRecord
                    continue;
                }
            } while (r == null);

            // 这里其实应该是上一轮的代码， nextReceiver++（顺带贴这里了）
            // Get the next receiver...
            int recIdx = r.nextReceiver++;
```

解释基本上代码的注释中，然后贴下 BroadcastRecord 初始化代码：

```java
    BroadcastRecord(BroadcastQueue _queue,
            Intent _intent, ProcessRecord _callerApp, String _callerPackage,
            int _callingPid, int _callingUid, String _requiredPermission,
            List _receivers, IIntentReceiver _resultTo, int _resultCode,
            String _resultData, Bundle _resultExtras, boolean _serialized,
            boolean _sticky, boolean _initialSticky,
            int _userId) {
        queue = _queue;
        intent = _intent;
        callerApp = _callerApp;
        callerPackage = _callerPackage;
        callingPid = _callingPid;
        callingUid = _callingUid;
        requiredPermission = _requiredPermission;
        receivers = _receivers;
        resultTo = _resultTo;
        resultCode = _resultCode;
        resultData = _resultData;
        resultExtras = _resultExtras;
        ordered = _serialized;
        sticky = _sticky;
        initialSticky = _initialSticky;
        userId = _userId;
        // 注意下下面这个初始值
        nextReceiver = 0;
        state = IDLE;
    }
```

APP_RECEIVE 的 state 在静态注册接收器分2种情况讨论那里设置的（文章有点长，有搜索倒回去看看吧）。这个 state 的变化流程是：

<pre>
IDLE(init) --> APP_RECEIVE(handle) --> IDLE(finish)
</pre>

然后后面处理下一个接收器就是和前面一样的了，然后直到 mOrderedBroadcasts 中没有待处理的接收器为止。这样就形成了静态注册接收器一个接一个的处理。不过细心的你应该发现了一点，这种实现有个很不靠谱的地方：那就是它要假设上一个接收器正常完成处理。那么如果上一个接收器在处理的过程中挂掉了，或是在处理的时候耗费了大量时间还没处理，是不是串行广播就没法发送到下一个接收器了呢。这个问题我们留在后面再讨论，我们得继续回去把静态注册接收器的第二种情况讨论完，再说这个问题（不然越扯越远，前面说的什么都忘记了）。

PS：头有点晕的回最开始看一下图。

### 接收器进程还没启动 

好现在是比较复杂的情况了，接收器进程还没启动，经过之前 Binder 普通服务篇大致能猜到这里也是发请求给 AMS 去启动指定的进程，然后等待接收器进程启动，再做广播处理。我们先接着看 processNextBroadcast 最后的处理：

```java
            // Not running -- get it started, to be executed when the app comes up.
            if (DEBUG_BROADCAST)  Slog.v(TAG, 
                    "Need to start app ["          
                    + mQueueName + "] " + targetProcess + " for broadcast " + r);
            // 果然是调用 AMS 去启动目标进程
            if ((r.curApp=mService.startProcessLocked(targetProcess,
                    info.activityInfo.applicationInfo, true,
                    r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND, 
                    "broadcast", r.curComponent,   
                    (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false))
                            == null) {                     
                // Ah, this recipient is unavailable.  Finish it if necessary,
                // and mark the broadcast record as ready for the next.
                Slog.w(TAG, "Unable to launch app "
                        + info.activityInfo.applicationInfo.packageName + "/"
                        + info.activityInfo.applicationInfo.uid + " for broadcast "
                        + r.intent + ": process is bad");
                logBroadcastReceiverDiscardLocked(r);
                // 经过上面的分析，下面这2个函数的调用是结束当前这个接收器的处理接着处理下一个，
                // 因为如果启动接收器进程失败了，就忽略这个，接着要处理下一个
                finishReceiverLocked(r, r.resultCode, r.resultData,
                        r.resultExtras, r.resultAbort, true);
                scheduleBroadcastsLocked();    
                r.state = BroadcastRecord.IDLE;
                return;
            }

            // 这里保存一下等待接收器进程启动的广播对象和索引
            mPendingBroadcast = r;         
            mPendingBroadcastRecvIndex = recIdx;
```

AMS 启动进程的 startProcessLocked 接口这里不再多说，可以去 [Binder 普通服务篇的相关章节](http://light3moon.com/2015/01/28/Android Binder 分析——普通服务 Binder 对象的传递/#服务进程没有启动，服务代码也还没执行 "Binder 普通服务篇的相关章节") 看一下。然后后面把当前这一次的 BroadcastRecord 保存到了 mPendingBroadcast 中。这个变量前面有看到过，但是没细说。不过经过 Binder 普通服务篇应该不陌生了，这就是要等候接收器进程启动起来，然后 AMS 接着处理的时候能找回之前等待进程启动的 BroadcastRecord（进程启动最后是会通知 AMS 做一些事情的）。虽然说基本流程我们已经猜得差不多了，但是还是继续把代码看完吧。

上面这里就是 processNextBroadcast 最后的部分了。发送启动接收器进程请求给 AMS 之后，这次的广播处理暂时就完了。然后如果接收器进程正常启动的话那么它的 ActivityThread 会调用 AMS 的 attachApplication：

```java
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity(); 
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    } 

... ...

    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
        
        // Find the application record that is being attached...  either via
        // the pid if we are running in multiple processes, or just pull the
        // next app record if we are emulating process with anonymous threads.
        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }   
        } else {    
            app = null; 
        }  

... ...

        // 这里检测下是不是有广播在等待新启动的进程
        // Check if a next-broadcast receiver is in this process...
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {
                // 如果有的话就要继续执行广播处理
                didSomething = sendPendingBroadcastsLocked(app);
            } catch (Exception e) {
                // If the app died trying to launch the receiver we declare it 'bad'
                badApp = true; 
            }     
        }     

        // Check whether the next backup agent is in this process...
        if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
            if (DEBUG_BACKUP) Slog.v(TAG, "New app is backup target, launching agent for " + app); 
            ensurePackageDexOpt(mBackupTarget.appInfo.packageName);
            try { 
                thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                        compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                        mBackupTarget.backupMode);
            } catch (Exception e) {
                Slog.w(TAG, "Exception scheduling backup agent creation: ");
                e.printStackTrace();
            }     
        }     

        if (badApp) {
            // todo: Also need to kill application to deal with all
            // kinds of exceptions.
            handleAppDiedLocked(app, false, true);
            return false;
        }     

        if (!didSomething) {
            updateOomAdjLocked();
        }     

        return true; 
    }
```

我们来看是怎么检测的：

```java
    boolean isPendingBroadcastProcessLocked(int pid) {
        return mFgBroadcastQueue.isPendingBroadcastProcessLocked(pid)
                || mBgBroadcastQueue.isPendingBroadcastProcessLocked(pid);
    }

// ============== BroadcastQueue.java ====================

    public boolean isPendingBroadcastProcessLocked(int pid) {
        return mPendingBroadcast != null && mPendingBroadcast.curApp.pid == pid;
    }
```

果然前面保存的 mPendingBroadcast 是要等着后面用的。然后我们继续看 AMS 的 sendPendingBroadcastsLocked：

```java
    // 这注释已经说得很清楚了
    // The app just attached; send any pending broadcasts that it should receive
    boolean sendPendingBroadcastsLocked(ProcessRecord app) { 
        boolean didSomething = false;
        for (BroadcastQueue queue : mBroadcastQueues) {
            didSomething |= queue.sendPendingBroadcastsLocked(app);
        }   
        return didSomething;
    }

// ============== BroadcastQueue.java ====================

    public boolean sendPendingBroadcastsLocked(ProcessRecord app) {
        boolean didSomething = false;
        // 取之前等待接收器进程启动的 BroadcastRecord
        final BroadcastRecord br = mPendingBroadcast; 
        if (br != null && br.curApp.pid == app.pid) {
            try {
                // 这里接上处理之后，就把 mPendingBroadcast 设置成 null
                mPendingBroadcast = null;
                // 下面这个函数就接上接收器进程已经启动的流程了
                processCurBroadcastLocked(br, app);
                didSomething = true;
            } catch (Exception e) {
                Slog.w(TAG, "Exception in new application when starting receiver "
                        + br.curComponent.flattenToShortString(), e);
                logBroadcastReceiverDiscardLocked(br);
                // 出错了继续要把广播发给一下个接收器
                finishReceiverLocked(br, br.resultCode, br.resultData,
                        br.resultExtras, br.resultAbort, true);
                scheduleBroadcastsLocked();
                // We need to reset the state if we failed to start the receiver.
                br.state = BroadcastRecord.IDLE;
                throw new RuntimeException(e.getMessage());
            }       
        }                   
        return didSomething;
    }
```

看到 processCurBroadcastLocked 就松了一口气，后面的流程就和上面静态接收器进程已经启动的情况一样了。好像有了之前的知识（Binder 普通服务篇），这种复杂的情况这里说起来也没多复杂的样子，果然稍微明白 android 的一些设计手法之后很多地方都通用的说。

然后这里补充一个细节，在前面在串行广播处理中有多出错的地方（例如说启动接收器进程失败，或者是接收器为 null 的情况）都会调用

<pre>
finishReceiverLocked();
scheduleBroadcastsLocked();
</pre>

这是为了能然广播能够分发下去，当前的出错了就跳过去，然后面的继续执行。

PS：头有点晕的继续回最开始看一下图。

## 优先级问题

到这里广播的发送、处理流程就差不多说完了。不过前面有说到优先级的问题，这里详细讨论一下。注册篇说到静态注册系统级应用可以在 manifest 的 '<intent-filter>' 设置优先级。这样在收集静态注册接收器的时候优先级高的能排在串行广播列表的前面，就会优先收到广播。但是从上面的处理流程来看，如果广播不是串行的（默认并行），那么动态注册的接收器优先级永远比静态注册的要高（并行处理的我们认为它们同一个时间接到广播）。

这里可以稍微理解下 android 的设计： 虽然说并行广播的本意是让所有接收器一起响应。但是通过前面的分析知道，动态注册的进程都是已经启动起来了的；静态注册的进程基本上都还需要启动。动态切换换字体那篇工作小笔记分析 zygote 的时候知道，启动一个进程在 android 中是十分巨大的一个操作。而且如果静态注册的接收器的进程全都没有启动，那么就需要启动很多个进程。如果同时在后台启动这么多个进程，会造成系统响应严重顿卡。所以 android 广播的处理原则是（并行广播）：优先让动态注册的接收器处理广播，然后再让静态注册的接收器串行一个接着一个的处理（这样一次只会启动一个进程）。

然后回来讨论优先级问题。静态注册的接收器按照 manifest 里声明的优先级排序。注册篇说到动态注册的接收器 IntentFilter 有一个 setPriority 可以设置（这个没限制系统应用才能设置），然后动态注册的时候 IntentResolver 也会根据这个优先级排序。前面说在并行广播中（默认）动态注册的接收器优先级肯定排在静态注册的前面，所以动态注册接收器的优先级在并行广播中用处不大。但是在串行广播中，动态注册的接收会合并到静态注册接收器的列表中（前面合有分析合并代码的）。这个时候动态注册接收器的优先级就有用了，合并操作对比动态注册接收器和静态注册接收器的优先级，然后决定动态注册接收器的插入位置（排在静态注册的前面还是后面）。

最后总结一下，前面话太多不直观，来几点：

1. 并行广播下，动态注册优先级 > 静态注册，静态注册按自己的优先级排序
2. 串行广播下，动态注册和静态注册按各自的优先级一起排序

## 总结

经过注册篇和本篇的讲解能看得出，广播是集中由 AMS 来处理的：

1. 通过 AMS 接口可以动态注册接收器到 AMS（存储在 AMS 中）
2. 通过 manifest 声明可以静态注册，由 PMS 扫描（存储在 PMS 中）
3. 通过 AMS 发送广播时，AMS 会收集（匹配）自己和 PMS 中的保存的接收器，获取接收器列表（BroadcastRecord）
4. 将 BroadcastRecord 放入指定队列（前台 or 后台），执行广播队列（BroadcastQueue）处理
5. 广播队列按照优先级将依次（或者并行）广播分发到接收器进程 BroadcastReceiver 回调

前面还有一个讨论串行广播等待处理完成的问题（包括超时问题），鉴于本篇已经很长了，新开一篇来说吧。



