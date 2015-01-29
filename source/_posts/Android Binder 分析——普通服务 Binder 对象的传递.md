title: Android Binder 分析——普通服务 Binder 对象的传递
date: 2015-01-28 21:33:16
categories: [Android Framework]
tags: [android]
---

上一篇把 SS 传递 binder 对象说了， SS 传递是要依靠 SM 的。但是普通应用的服务是没权向 SM 注册的，就是说普通应用获取普通服务的接口不能像 SS 那样，通过 SM 的接口去取。这篇就来讲讲普通应用中 binder 对象的传递。

那这篇主要就是分析这些，照例先把源代码位置啰嗦一下（4.4）：

```bash
# java 层 Context 相关接口
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/LoadedApk.java
frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/core/java/android/content/ServiceConnection

# java 层 Service 相关接口
frameworks/base/core/java/android/app/IActivityManager.java
frameworks/base/core/java/android/app/ActivityManager.java
frameworks/base/core/java/android/app/ActivityManagerNative.java
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/java/com/android/server/am/ActiveServices.java
frameworks/base/services/java/com/android/server/am/ServiceRecord.java
```

## Client 相关接口
还是和上一篇的一样，先从相关接口说。普通服务和 SS 不一样，好像没有 native 层相关的接口，虽然说 native 层有个 BinderService 的东西，但是没有相关绑定的接口，所以说应该没办法写 native 的普通服务。所以这里只说 java 层的。

要在别的进程使用普通服务提供的 IPC 接口，需要绑定服务，我们来看下绑定服务的接口：

```java
    public abstract boolean bindService(Intent service, ServiceConnection conn,
            int flags);
``` 

这个是 Context 里面的接口，就是说一般在 Activity 里面可以调用这个来绑定服务。Intent 是要绑定的服务。ServiceConnection 是个接口，我们待会再说。flags 是一个标志，可以设置好几个，这里我们讨论最常见的一个： `BIND_AUTO_CREATE` 这个表示如果要绑定的服务的进程没有运行就自动启动。

然后我们来说说 ServiceConnection 这个接口：

```java
public interface ServiceConnection {
    /* 
     * Called when a connection to the Service has been established, with
     * the {@link android.os.IBinder} of the communication channel to the
     * Service.
     *
     * @param name The concrete component name of the service that has
     * been connected.
     *
     * @param service The IBinder of the Service's communication channel,
     * which you can now make calls on.
     */
    public void onServiceConnected(ComponentName name, IBinder service);

    /*
     * Called when a connection to the Service has been lost.  This typically
     * happens when the process hosting the service has crashed or been killed.
     * This does <em>not</em> remove the ServiceConnection itself -- this
     * binding to the service will remain active, and you will receive a call
     * to {@link #onServiceConnected} when the Service is next running.
     *
     * @param name The concrete component name of the service whose
     * connection has been lost.
     */
    public void onServiceDisconnected(ComponentName name);
}
```

注释已经说得比较清楚了。在 onServiceConnected 中有 IBinder 对象，前面原理篇说了，这个就是 java 层的 binder 对象，有了这个就是拿到了 Bp 端对象了，就可以通过这个发起 IPC 调用了。一般来说在某个 Activity 中要 bindService，就会让这个 Activity 实现 ServiceConnection 接口，然后在 onServiceConnected 回调取到要使用的 Service 的 binder 对象，然后就完成了普通应用的 binder 对象的传递。

onServiceDisconnected 是告诉你服务连接中断了，在 onServiceDisconnected 中取到的 IBinder 对象无效了。这个时候可能是服务所在的进程被杀掉，或是异常退出了。这个问题后面再说，这里只讨论连接的情况，也就是 IBinder 对象的传递。

## Service 相关接口
前面说了 Client 端的接口，在 Client 需要 bindService，那么 Service 这边也有相应的接口。framework 里面有一个抽象类 Service.java ，普通服务都要继承这个类，然后实现 IBinder 接口。这个 Service 是一个抽象类，它有一个比较重要的需要重载的函数：

```java
public abstract IBinder onBind(Intent intent);
```

这个函数的注释太长了，意思就是说需要返回自己服务的 IBinder 对象。没错，这个返回的 IBinder 对象就是前面 Client ServiceConnection 接口 onServiceConnected 的那个参数。只不过在 Service 的 onBind 里返回的是 Bn，到了 Client 的 onServiceConnected 就变成 Bp 了（具体的可以回去看 SS 传递篇，binder 的传递过程是一样的）。

## bindService 的实现
前面把接口、用法说完了。我们来看看系统的实现吧。首先我们假设有个一个普通的服务叫 SA（Service A，它在 Proc A 中），另一个普通应用 AppB（它在 Porc B 中）。现在 SA 要提供一些 IPC 接口，它首先得继承 Service ，然后实现 IBinder 接口，并且在 onBind 函数中返回自己实现的 IBinder 对象。 AppB 要调用 SA 的某个接口，那么它就得调用 Context 的 bindService，并且自己实现 ServiceConnection，在 onServiceConnected 中取得 SA 的 IBiner 对象，然后就可以通过取得的 IBinder 对象调用 SA 的 IPC 接口了。

Context 的 bindService 实现在 ContextImpl 中：

```java
    @Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, Process.myUserHandle());
    }

    // 真正干活的是这个函数
    private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            UserHandle user) {
        IServiceConnection sd;
        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }    
        if (mPackageInfo != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
                    mMainThread.getHandler(), flags);
        } else {
            throw new RuntimeException("Not supported in system context");
        }    
        validateServiceIntent(service);
        try {
            IBinder token = getActivityToken();
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null 
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                flags |= BIND_WAIVE_PRIORITY;
            }    
            service.prepareToLeaveProcess();
            int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(),
                service, service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            return false;
        }
    }
```

下面的代码讲解，我会忽略很多不太重要的东西，因为 ActivityManager（AM）实在是太庞大了，如果追根究底的话，就太多了，而且容易丢掉主要的东西。首先来看看 mPackageInfo 这个东西。它是一个 LoadedApk 的变量。这个类好像是会保存一些加载了 apk 的信息，例如应用 data 目录，lib 目录等等，在这里最重要的是它保存了 service 绑定的一些信息。我们来看看下它的 getServiceDispatcher 函数：

```java
    public final IServiceConnection getServiceDispatcher(ServiceConnection c,
            Context context, Handler handler, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            // 这个 mServices 就是连接的信息的 map 表
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            if (map != null) {
                sd = map.get(c);
            }    
            if (sd == null) {
                // 没有的话就创建一个新的
                sd = new ServiceDispatcher(c, context, handler, flags);
                // 然后保存起来，方便下次使用
                if (map == null) {
                    map = new ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>();
                    mServices.put(context, map);
                }    
                map.put(c, sd); 
            } else {
                // 如果之前保存就直接取
                sd.validate(context, handler);
            }    
            return sd.getIServiceConnection();
        }    
    }
```

接下去看下 ServiceDispatcher（它是 LoadedApk 中的内部类） 这个东西：

```java
   static final class ServiceDispatcher {
        private final ServiceDispatcher.InnerConnection mIServiceConnection;
        private final ServiceConnection mConnection;
        private final Context mContext;
        private final Handler mActivityThread;
        private final ServiceConnectionLeaked mLocation;
        private final int mFlags;

        private RuntimeException mUnbindLocation;

        private boolean mDied;
        private boolean mForgotten;

        private static class ConnectionInfo {
            IBinder binder;
            IBinder.DeathRecipient deathMonitor;
        }

        // IserviceConnection 接口的实现
        private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service) throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {              
                    sd.connected(name, service);   
                }
            }
        }

        private final ArrayMap<ComponentName, ServiceDispatcher.ConnectionInfo> mActiveConnections
            = new ArrayMap<ComponentName, ServiceDispatcher.ConnectionInfo>();

        ServiceDispatcher(ServiceConnection conn,
                Context context, Handler activityThread, int flags) {
            mIServiceConnection = new InnerConnection(this);
            mConnection = conn;            
            mContext = context;            
            mActivityThread = activityThread;
            mLocation = new ServiceConnectionLeaked(null);
            mLocation.fillInStackTrace();
            mFlags = flags;
        }

        public void connected(ComponentName name, IBinder service) {
            if (mActivityThread != null) { 
                mActivityThread.post(new RunConnection(name, service, 0));
            } else {
                doConnected(name, service);    
            }
        }

        public void doConnected(ComponentName name, IBinder service) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {          
                if (mForgotten) {              
                    // We unbound before receiving the connection; ignore
                    // any connection received.    
                    return;
                }
                old = mActiveConnections.get(name);
                if (old != null && old.binder == service) {
                    // Huh, already have this one.  Oh well!
                    return;
                }

                if (service != null) {         
                    // A new service is being connected... set it all up.
                    mDied = false;                 
                    info = new ConnectionInfo();   
                    info.binder = service;         
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                        service.linkToDeath(info.deathMonitor, 0);
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {  
                        // This service was dead before we got it...  just
                        // don't do anything with it.  
                        mActiveConnections.remove(name);
                        return;                        
                    }

                } else {
                    // The named service is being disconnected... clean up.
                    mActiveConnections.remove(name);
                }

                if (old != null) {             
                    old.binder.unlinkToDeath(old.deathMonitor, 0);
                }
            }

            // If there was an old service, it is not disconnected.
            if (old != null) {
                mConnection.onServiceDisconnected(name);
            }
            // If there is a new service, it is now connected.
            if (service != null) {
                // 终于调到这个回调了
                mConnection.onServiceConnected(name, service);
            }
        }
... ... 
    } 
``` 

其实 ServiceDispatcher 就是封装了下 ServiceConnection，因为 ServiceConnection 是没有实现 IBinder 接口，但是最后 ServiceConnection 的接口是在 AM 里面调用的，所以它必须要实现 IBinder 接口才能在 AM 调用。所以这里弄了一个 InnerConnection 的实现出来，其实就是包装了下 ServiceConnection 的回调而已。这里还多弄了一堆东西出来，层层调用，这里不管别的，只要知道 InnerConnection 实现的 IServiceConnection 接口能够在 AM 被调用，调用后就会调用 Proc B 实现的 onServiceConnected，然后 Proc B 就能得到 SA 的 IBinder 对像了。

好这里现在就要看看是哪里调用了 IServiceConnection 的接口（其实在 AM 里面）。然后我们回到 ContextImpl 的 bindServiceCommon，取到 IServiceConnection 接口后（这里是 Bn 端），然后就调用 AM 的 bindService 的 IPC 接口了，并且把 IServiceConnection 传了过去。这里 IServiceConnection 也是一个 IBinder 对象来的，它的传递参见 SS 那篇。其实 IBinder 对象的传递，只要有最原始的 Bn 端就能够传递，SS 是把自己的 Bn 传给了 SM，其它进程再从 SM 那取。因为其它进程没 SS 原始 Bn 端。但是在 binder 设计上 SM 是特殊的（handle 固定是 0），在任何进程可以取得到，所以可以通过 SM 取指定的 SS 的 IBinder 对象。这里有 IServiceConnection 的原始 Bn 对象所以是可以传递给 AM 的。


接下来我们就要去看 AM 的 bindService 接口了（其实很多一部分工作是在 AM 里面完成的，普通应用程序通过 SM 取 AM -_-||）。

```java
    public int bindService(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType,
            IServiceConnection connection, int flags, int userId) {
        enforceNotIsolatedCaller("bindService");
        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service, resolvedType,
                    connection, flags, userId);    
        }
    }
```

AM 中 Service 这一块的相关工作是在 ActiveServices.java 中实现的：

```java
    int bindServiceLocked(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType,
            IServiceConnection connection, int flags, int userId) {
        if (DEBUG_SERVICE) Slog.v(TAG, "bindService: " + service
                + " type=" + resolvedType + " conn=" + connection.asBinder()
                + " flags=0x" + Integer.toHexString(flags));

... ...

        // 这里取到 ServiceRecord
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType,
                    Binder.getCallingPid(), Binder.getCallingUid(), userId, true, callerFg);
        if (res == null) {
            return 0;
        }
        if (res.record == null) {
            return -1;
        }
        ServiceRecord s = res.record;

... ...

        return 1;
    }
```

这个函数比较长，我们只看关键的地方。首先通过 retrieveServiceLocked 获取 ServiceRecord（这个 ServiceRecord 是个比较重要的东西，在 AM 中记录了 Service 的相关数据）：

```java
    private ServiceLookupResult retrieveServiceLocked(Intent service,
            String resolvedType, int callingPid, int callingUid, int userId,
            boolean createIfNeeded, boolean callingFromFg) {
        ServiceRecord r = null;
        if (DEBUG_SERVICE) Slog.v(TAG, "retrieveServiceLocked: " + service
                + " type=" + resolvedType + " callingUid=" + callingUid);

        userId = mAm.handleIncomingUser(callingPid, callingUid, userId,
                false, true, "service", null);

        ServiceMap smap = getServiceMap(userId);
        final ComponentName comp = service.getComponent();
        // 先通过 ComponentName 在 ServiceMap 中取 ServiceMap
        if (comp != null) {
            r = smap.mServicesByName.get(comp);
        }
        // ComponentName 没取到的话，再通过 Intent filer 取    
        if (r == null) {
            Intent.FilterComparison filter = new Intent.FilterComparison(service);
            r = smap.mServicesByIntent.get(filter);
        }
        if (r == null) {
            try {       
                ResolveInfo rInfo =
                    AppGlobals.getPackageManager().resolveService(
                                service, resolvedType,
                                ActivityManagerService.STOCK_PM_FLAGS, userId);
                ServiceInfo sInfo =
                    rInfo != null ? rInfo.serviceInfo : null;
                if (sInfo == null) {
                    Slog.w(TAG, "Unable to start service " + service + " U=" + userId +
                          ": not found");
                    return null;
                }
                ComponentName name = new ComponentName(
                        sInfo.applicationInfo.packageName, sInfo.name);
                if (userId > 0) {
                    if (mAm.isSingleton(sInfo.processName, sInfo.applicationInfo,
                            sInfo.name, sInfo.flags)) {
                        userId = 0;
                        smap = getServiceMap(0);
                    }
                    sInfo = new ServiceInfo(sInfo);
                    sInfo.applicationInfo = mAm.getAppInfoForUser(sInfo.applicationInfo, userId);
                }
                // 最后再取一次
                r = smap.mServicesByName.get(name);
                if (r == null && createIfNeeded) {
                    Intent.FilterComparison filter
                            = new Intent.FilterComparison(service.cloneFilter());
                    ServiceRestarter res = new ServiceRestarter();
                    BatteryStatsImpl.Uid.Pkg.Serv ss = null;
                    BatteryStatsImpl stats = mAm.mBatteryStatsService.getActiveStatistics();
                    synchronized (stats) {
                        ss = stats.getServiceStatsLocked(
                                sInfo.applicationInfo.uid, sInfo.packageName,
                                sInfo.name);
                    }
                    // 再取不到就 new 一个了
                    r = new ServiceRecord(mAm, ss, name, filter, sInfo, callingFromFg, res);
                    res.setService(r);
                    // 然后添加到 ServiceMap 中去
                    smap.mServicesByName.put(name, r);
                    smap.mServicesByIntent.put(filter, r);

                    // Make sure this component isn't in the pending list.
                    for (int i=mPendingServices.size()-1; i>=0; i--) {
                        ServiceRecord pr = mPendingServices.get(i);
                        if (pr.serviceInfo.applicationInfo.uid == sInfo.applicationInfo.uid
                                && pr.name.equals(name)) {
                            mPendingServices.remove(i);
                        }
                    }
                }
            } catch (RemoteException ex) {
                // pm is in same process, this will never happen.
            }
        }
        // 后面 ServiceLookupResult 其实可以不怎么管
        if (r != null) {
            if (mAm.checkComponentPermission(r.permission,
                    callingPid, callingUid, r.appInfo.uid, r.exported)
                    != PackageManager.PERMISSION_GRANTED) {
                if (!r.exported) {
                    Slog.w(TAG, "Permission Denial: Accessing service " + r.name
                            + " from pid=" + callingPid
                            + ", uid=" + callingUid
                            + " that is not exported from uid " + r.appInfo.uid);
                    return new ServiceLookupResult(null, "not exported from uid "
                            + r.appInfo.uid);
                }
                Slog.w(TAG, "Permission Denial: Accessing service " + r.name
                        + " from pid=" + callingPid
                        + ", uid=" + callingUid
                        + " requires " + r.permission);
                return new ServiceLookupResult(null, r.permission);
            }
            if (!mAm.mIntentFirewall.checkService(r.name, service, callingUid, callingPid,
                    resolvedType, r.appInfo)) {
                return null;
            }
            return new ServiceLookupResult(r, null);
        }
        return null;
    }
```

AM 中有一个 ServiceMap 保存了 ComponentName、Intent filter 相关的 ServiceRecord （反正通过 Intent 能找得到的）。然后我们来看下 ServiceRecord 的结构：

```java
// ServiceRecord.java ======================================

/*
 * A running application service.
 */
final class ServiceRecord extends Binder {
    // Maximum number of delivery attempts before giving up.
    static final int MAX_DELIVERY_COUNT = 3;

    // Maximum number of times it can fail during execution before giving up.
    static final int MAX_DONE_EXECUTING_COUNT = 6;

... ...

    final ArrayMap<Intent.FilterComparison, IntentBindRecord> bindings
            = new ArrayMap<Intent.FilterComparison, IntentBindRecord>();
                            // All active bindings to the service.
    final ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections
            = new ArrayMap<IBinder, ArrayList<ConnectionRecord>>();
                            // IBinder -> ConnectionRecord of all bound clients

... ...

}

// ConnectionRecord.java =====================================

/**
 * Description of a single binding to a service.
 */
final class ConnectionRecord {
    final AppBindRecord binding;    // The application/service binding.
    final ActivityRecord activity;  // If non-null, the owning activity.
    final IServiceConnection conn;  // The client connection.
    final int flags;                // Binding options.
    final int clientLabel;          // String resource labeling this client.
    final PendingIntent clientIntent; // How to launch the client.
    String stringName;              // Caching of toString.
    boolean serviceDead;            // Well is it?

... ...

// IntentBindRecord.java ===================================== 

/**
 * A particular Intent that has been bound to a Service.
 */
final class IntentBindRecord {
    /** The running service. */
    final ServiceRecord service;
    /** The intent that is bound.*/
    final Intent.FilterComparison intent; //
    /** All apps that have bound to this Intent. */
    final ArrayMap<ProcessRecord, AppBindRecord> apps
            = new ArrayMap<ProcessRecord, AppBindRecord>();
    /** Binder published from service. */
    IBinder binder;
    /** Set when we have initiated a request for this binder. */
    boolean requested;
    /** Set when we have received the requested binder. */
    boolean received;
    /** Set when we still need to tell the service all clients are unbound. */
    boolean hasBound;
    /** Set when the service's onUnbind() has asked to be told about new clients. */
    boolean doRebind;

}
```

ServiceRecord 中有一个 ConnectionRecord 的数组，ConnectionRecord 中有 IServiceConnection 这个 IBinder 接口。这个就是上面封装 Client 中实现的 ServiceConnection 接口。然后 ServiceRcord 里面还有一个 IntentBindRecord 这个记录服务被绑定的信息。不过 new ServiceRecord 那里并没设置 connections，我们接着往下看:

```java
// ActiveServices--bindServiceLocked  ====================

        // s 为之前取到的 ServiceRecord
        try {
            if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {
                if (DEBUG_SERVICE) Slog.v(TAG, "BIND SERVICE WHILE RESTART PENDING: "
                        + s);
            }

            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();
                if (!s.hasAutoCreateConnections()) {
                    // This is the first binding, let the tracker know.
                    ProcessStats.ServiceState stracker = s.getTracker();
                    if (stracker != null) {
                        stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(),
                                s.lastActivity);
                    }
                }
            }

            // 在 ServiceRecord 里取 AppBindRecord，最关键的是这个函数
            // 会触发ServiceRecord创建IntentBindRecord（如果还没创建的话）
            // IntentBindRecord 前面说了，保存了绑定的信息的
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
            // 这里 new ConnectionRecord 了，注意参数 connection 是
            // client bindService 传过来的 IServiceConnection
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent);

            // 之前没 ServiceRecord 的 connections 没设置，这里设置了哦
            IBinder binder = connection.asBinder();
            ArrayList<ConnectionRecord> clist = s.connections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                s.connections.put(binder, clist);
            }
            clist.add(c);
            b.connections.add(c);
            if (activity != null) {
                if (activity.connections == null) {
                    activity.connections = new HashSet<ConnectionRecord>();
                }
                activity.connections.add(c);
            }
            b.client.connections.add(c);
            if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
                b.client.hasAboveClient = true;
            }
            if (s.app != null) {
                updateServiceClientActivitiesLocked(s.app, c);
            }
            clist = mServiceConnections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                mServiceConnections.put(binder, clist);
            }
            clist.add(c);

            ... ...

        } finally {
            Binder.restoreCallingIdentity(origId);
        }
```

将 Client 的 IServiceConneciton 保存到 AM 中了。然后后面的处理要分为3种情况来说：

1. 要绑定的服务所在的进程已经在运行，并且服务代码也已经执行了，这个时候只要请求绑定服务就行了。
2. 要绑定的服务所在的进程已经在运行，但是服务代码没有执行，这个时候需要执行服务代码，然后再绑定服务。
3. 要绑定的服务的进程还没运行，要先启动服务所在的进程，然后执行服务代码，最后再绑定服务。

### 服务进程和服务都已经启动
第一种情况比较简单，进程和服务都在运行，直接绑定就可以了。

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-service/1.png)

我们继续看 bindServiceLocked 的代码：

```java
            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false) != null) {
                    return 0;
                }
            }
```

`BIND_AUTO_CREATE` 前面说了，如果要绑定的服务没启动的话，会自动帮你启动服务。主要是 bringUpServiceLocked 这个函数处理，看名字还挺形象的，把服务拉起来 -_-||。这里既然是第一种情况，我们也当设置了这个标志，进去看看:

```java
    private final String bringUpServiceLocked(ServiceRecord r,
            int intentFlags, boolean execInFg, boolean whileRestarting) {
        //Slog.i(TAG, "Bring up service:");
        //r.dump("  ");
        
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }      

... ...
        
        return null;
    }
```

如果是服务所在进程已经在运行（r.app != null），服务代码也已经被执行了（r.app.thread != null），这里啥也没做，马上就返回 null 了。我们回到 bindServiceLocked 继续往下：

```java
            // 更新下最近运行的程序和 OOM
            if (s.app != null) {           
                // This could have made the service more important.
                mAm.updateLruProcessLocked(s.app, s.app.hasClientActivities, b.client);
                mAm.updateOomAdjLocked(s.app); 
            }

            if (DEBUG_SERVICE) Slog.v(TAG, "Bind " + s + " with " + b
                    + ": received=" + b.intent.received
                    + " apps=" + b.intent.apps.size()
                    + " doRebind=" + b.intent.doRebind);

            // 这个判断代表服务已经跑起来了
            if (s.app != null && b.intent.received) {
                // Service is already running, so we can immediately
                // publish the connection.     
                try {
                    // 调用 Client 的 IServiceConnection 的接口了
                    c.conn.connected(s.name, b.intent.binder);
                } catch (Exception e) {        
                    Slog.w(TAG, "Failure sending service " + s.shortName
                            + " to connection " + c.conn.asBinder()
                            + " (in " + c.binding.client.processName + ")", e);
                }

                // 我们先不管重新绑定的情况
                // If this is the first app connected back to this binding,
                // and the service had previously asked to be told when
                // rebound, then do so.
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }
            } else if (!b.intent.requested) {
                // 正常应该走这里的
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            }

            getServiceMap(s.userId).ensureNotStartingBackground(s);
```

这里这个 s.app != null && b.intent.received 判断应该是比较特殊的， app != null 好说。那个 IntentBindRecord 的 received == true 的话，后面会看到，请求绑定的话，会被设置为 true 的，就是说这个已经绑定过了，所以可以直接调用 IServiceConnection 接口了。调用这个接口相关的，后面再说。我们主要看正常路线，就是下面那个 !b.intent.requested。这个会调用 requestServiceBindingLocked：

```java
    private final boolean requestServiceBindingLocked(ServiceRecord r, 
            IntentBindRecord i, boolean execInFg, boolean rebind) {
        // 看这个判断，前面那个实际上只有 b.intent.received 有用而已
        if (r.app == null || r.app.thread == null) {
            // If service is not currently running, can't yet bind.
            return false;
        }
        // 注意这个 requested 的判断，如果已经 requested 过了就不处理了       
        if ((!i.requested || rebind) && i.apps.size() > 0) {
            try {   
                // 远程调用到服务的线程执行绑定操作
                bumpServiceExecutingLocked(r, execInFg, "bind");
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.repProcState);
                // 成功后，会把 IntentBindRecord的 requested 设置为 true
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            } catch (RemoteException e) {
                if (DEBUG_SERVICE) Slog.v(TAG, "Crashed while binding " + r);
                return false;
            }
        }
        return true;
    }
```

最开始 r.app == null || r.app.thread == null 这个判断说明这条路线需要绑定的服务，进程、执行线程都要在运行才行。然后 r.app.thread 这个其实是一个 IActivityThread 的 IBinder 接口。ActivityThread（AT） 是 android 应用进程的主线程，java 的 main 函数在这个线程里面跑的。AT 管理了进程的主线程，负责执行各种 Activity、Service 和 Broadcast。简单的来说就这里这个 IPC 远程调用，就跑到服务运行的进程中去了。注意传递的一个参数，把 ServiceRecord 传过去了，ServiceRecord 在 java 层继承自 Binder，能够在 IPC 之间传递的。我们来具体看下吧

```java
// ActivityThread.java ==========================

        public final void scheduleBindService(IBinder token, Intent intent,
                boolean rebind, int processState) {
            updateProcessState(processState, false);
            BindServiceData s = new BindServiceData();
            s.token = token;
            s.intent = intent;
            s.rebind = rebind;

            if (DEBUG_SERVICE)
                Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                        + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
            sendMessage(H.BIND_SERVICE, s);
        }
```

这里是发送消息到 Handle 里面去了，然后就返回了。我们看到 AT 的这个 Handle（mH）其实还是在 AT 这个线程里面（代码我不贴了，直接在 AT 中 new 出来的，Looper 还是 AT 的线程来的）。为什么还要费劲转到 Handle 里处理呢。这里说一下，主要是为了能够马上返回，因为这个调用是在 AM 的 bindService 里面的，这个函数后面的调用是有 AM 的锁的。然后后面会看到 handleBindService 里面又要调到 AM 里面去的，然后那个里面也有 AM 的锁，如果这里不返回的话，后面会调用 AM 里面就会死锁的（至于为什么 AM 里面有那么多锁，后面多线程篇会讲到的）。这里返回后后面就为什么处理了，我们去看 AT Handle 里面的处理：

```java
    private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (DEBUG_SERVICE)
            Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
        if (s != null) {
            try {   
                data.intent.setExtrasClassLoader(s.getClassLoader());
                try {
                    if (!data.rebind) {
                        // 这里调用 Service 实现的 onBind 获取 Service 
                        // 的 IBinder 对象了。
                        IBinder binder = s.onBind(data.intent);
                        ActivityManagerNative.getDefault().publishService(
                                data.token, data.intent, binder);
                    } else {
                        s.onRebind(data.intent);
                        ActivityManagerNative.getDefault().serviceDoneExecuting(
                                data.token, 0, 0, 0);
                    }
                    ensureJitEnabled();
                } catch (RemoteException ex) {
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to bind to service " + s
                            + " with " + data.intent + ": " + e.toString(), e);
                }
            }
        }
    }
```

开始那个 mServices 是一个 map，用来保存 Service 对象的，key 是 scheduleBindService 传递的 ServiceRecord 的 IBinder 对象。后面启动服务的时候，会看到，在本线程内每启动一个 Service 会把这个 Service 对象保存到 mServices 里去的。这么看其实可以一个线程里面跑多个服务，在应用层来说应该是一个进程里面可以跑多个服务的（这个是主线程）。

取到 Service 对象后，在 Service 本进程里可以调用 Service 的 onBind 函数取得 Service 的 IBinder 对象。然后把这个 IBinder 对象又传递给 AM 那调用 publishService 了。转了半天又回去了（不过想想其实也是对的，后面要调用 Client 的 IServiceConnection 的回调，这个应该由 AM 处理比较好）：

```java
// ActivityManagerService.java ==========================

    public void publishService(IBinder token, Intent intent, IBinder service) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.publishServiceLocked((ServiceRecord)token, intent, service);
        }
    }
```

果然又是要到 ActiveServices.java 里面：

```java
    // ServiceRecord 被 AM 传递给 ActivityThread，又传回来了 -_-||
    // 第二 service（IBinder） 是 ActivityThread 取的 Service 的，
    // Client 就是要这个 IBinder 对象
    void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity();
        try {
            if (DEBUG_SERVICE) Slog.v(TAG, "PUBLISHING " + r
                    + " " + intent + ": " + service);
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {
                    b.binder = service;
                    // 这里 requested 和 received 都设置 true 了
                    b.requested = true;
                    b.received = true;
                    // 前面 bindService 保存的 ConnectionRecord 这里要派上用场了
                    for (int conni=r.connections.size()-1; conni>=0; conni--) {
                        ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);
                            if (!filter.equals(c.binding.intent.intent)) {
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG, "Not publishing to: " + c);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG, "Bound intent: " + c.binding.intent.intent);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG, "Published intent: " + intent);
                                continue;
                            }
                            if (DEBUG_SERVICE) Slog.v(TAG, "Publishing to: " + c);
                            try {
                                // 终于调 IServiceConnection 回调了
                                c.conn.connected(r.name, service);
                            } catch (Exception e) {
                                Slog.w(TAG, "Failure sending service " + r.name +
                                      " to connection " + c.conn.asBinder() +
                                      " (in " + c.binding.client.processName + ")", e);
                            }
                        }
                    }
                }

                // 其他的都先忽略吧
                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

这里就算是最后一步了。 ActivityThread 把 Service 的 IBinder 对象传递过来了，然后 AM 把 bindService 中保存的 ConnectionRecord 遍历了一次，依次调用 IServiceConneciton 接口，这样 Client 那里就能得到 Service 的 IBinder 对象了（IBinder 对象的具体传递细节可以看上一篇）。注意下前面把 IntentBindRecord 的 requested 和 received 设置为 true 了，这样能防止重复调用 IServiceConneciton 接口（设置了特殊标志位的除外）。

### 服务进程已经启动，但是服务还没运行
接下来我们来看第二种情况，这种情况服务所在进程依然已经启动，但是服务可能还没运行。我们需要启动服务后，再绑定。

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-service/2.png)
（handleBindService 后面的我省略了，第一种情况画过了）

这条分支的话，我们在 bringUpServiceLocked 开头不会返回：

```java
    private final String bringUpServiceLocked(ServiceRecord r,
            int intentFlags, boolean execInFg, boolean whileRestarting) {
        //Slog.i(TAG, "Bring up service:");
        //r.dump("  ");

        // 这里不会返回
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        if (!whileRestarting && r.restartDelay > 0) {
            // If waiting for a restart, then do nothing.
            return null;
        }

        if (DEBUG_SERVICE) Slog.v(TAG, "Bringing up " + r + " " + r.intent);

        // We are now bringing the service up, so no longer in the
        // restarting state.
        if (mRestartingServices.remove(r)) {
            clearRestartingIfNeededLocked(r);
        }

        // Make sure this service is no longer considered delayed, we are starting it now.
        if (r.delayed) {
            if (DEBUG_DELAYED_STATS) Slog.v(TAG, "REM FR DELAY LIST (bring up): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        // Make sure that the user who owns this service is started.  If not,
        // we don't want to allow it to run.
        if (mAm.mStartedUsers.get(r.userId) == null) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }

        // Service is now being launched, its package can't be stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;

        // 前面就打打酱油，主要看下面的，这里正常使用不是 isolated 的
        if (!isolated) {
            // 通过 Service 的 procName 取 ProcessRecord
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                        + " app=" + app);
            // 这里如果第二种分支，这里就会走这条分支，去启动服务。
            // 然后返回，后面和第一种分支一样绑定服务了。
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, mAm.mProcessStats);
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }

                // If a dead object exception was thrown -- fall through to
                // restart the application.
            }
        } else {
            // If this service runs in an isolated process, then each time
            // we call startProcessLocked() we will get a new isolated
            // process, starting another process if we are currently waiting
            // for a previous process to come up.  To deal with this, we store
            // in the service any current isolated process it is running in or
            // waiting to have come up.
            app = r.isolatedProc;
        }

... ...
       
        return null;
    }
```

app != null && app.thread != null 这个判断说明服务所在的进程已经启动了（主线程也在跑了），就调用 realStartServiceLocked 去启动服务。如果不出错的话，启动完成后，bringUpServiceLocked 就会返回 null，然后继续第一种情况的分支，后面绑定服务。我们来看看 realStartServiceLocked：

```java
    // ServiceRerord 和 ProcessRecord 传递过来了
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        if (DEBUG_MU)
            Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                    + ", ProcessRecord.uid = " + app.uid);
        // 这里把 ProcessRecord 保存到 ServiceRecrod 里面去了
        // 以前再跑 bindService 就会走第一种情况的分支了 
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
 
        app.services.add(r);
        bumpServiceExecutingLocked(r, execInFg, "create");
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();

        boolean created = false;
        try {
            String nameTerm;
            int lastPeriod = r.shortName.lastIndexOf('.');
            nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
            EventLogTags.writeAmCreateService(
                    r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }    
            mAm.ensurePackageDexOpt(r.serviceInfo.packageName);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            // 和第一种情况差不多，远程跑到服务的进程中调用相关函数处理
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } finally {
            if (!created) {
                app.services.remove(r);
                r.app = null;
                scheduleServiceRestartLocked(r, false);
            }    
        }    

        // 这里怎么调用请求绑定的函数，后面返回之后同样还会调用的，
        // 不过有设置变量，倒不会重复调用，不过感觉有点怪
        requestServiceBindingsLocked(r, execInFg);

        // 后面的忽略了 ... ...
        // If the service is in the started state, and there are no
        // pending arguments, then fake up one so its onStartCommand() will
        // be called.
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }

        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            if (DEBUG_DELAYED_STATS) Slog.v(TAG, "REM FR DELAY LIST (new proc): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STATS) Slog.v(TAG, "Applying delayed stop (from start): " + r);
                stopServiceLocked(r);
            }
        }
    }
```

只要 realStartServiceLocked 之后，ServiceRecord 的 app 就不是 null 了，以后绑定，就可以走第一种情况的分支（这个快）。然后主要是远程调用到 AT 的 scheduleCreateService 去让 Service 的进程的主线程执行 Service 的代码：

```java
        public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            // token 是 ServiceRecord 的 IBinder 对象
            s.token = token;
            // info 里面有 Service 类名
            s.info = info;
            s.compatInfo = compatInfo;     

            sendMessage(H.CREATE_SERVICE, s);
        }
```

这里和前面一样也是发到 Handle 里面的，然后能够直接返回，我们先看看返回后，还有什么处理。其实返回后然后会调用 requestServiceBindsLocked（里面调用 requestServiceBindingsLocked） 去请求绑定服务，这个第一种情况的分支分析过了。不过 bringUpServiceLocked 返回 null 后，还有调用这个的，这里有设置标志，不会重复调用的，至于什么会有2个地方有请求绑定服务的函数，后面到第三种情况就知道了。

现在回到 AT 这边：

```java
    private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;        
        try {
            // 这里用反射 new 出 Service 的对象出来了                 
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {        
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(    
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);     
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

            // Context 也 new 了一个出来
            ContextImpl context = new ContextImpl();
            context.init(packageInfo, null, this);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            context.setOuterContext(service);
            // 这个 attach 设置下Service的运行上下文环境（刚刚new出来那个） 
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            // 调用了 Service 的生命周期函数 onCreate 了
            service.onCreate();            
            // 把这个 new 出来的 Service 对象保存到 mServices 里了。
            // key 是 ServiceRecord 的 IBinder 对象。
            // 前面 handleBindService 有用到这个东西。
            mServices.put(data.token, service);
            // 后面的忽略 ... ...
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, 0, 0, 0);          
            } catch (RemoteException e) {  
                // nothing to do.              
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
```

这个 scheduleCreateService 名字太形象的，还真是 new 了一个 Service 对象出来 -_-||。这样 Service 的代码就在主线程中加载起来了（执行了）。前面 AM 那边又会调用 requestServiceBindingsLocked 会让 AT 这边继续向 Handle 发一个 handleBindService 的消息，Handle 的 MessageQueue 会排队的，不用担心，能保证先 CreateService 再 BindService。

### 服务进程没有启动，服务代码也还没执行
前面2种情况说完了。第三种是最悲剧的，服务没运行，连所在的进程都还没跑起来。这种情况下需要启动服务所在进程、然后执行服务代码，最后才能绑定服务。这种情况是最慢的，所以 bindService 有些时候回调快、有些时候回调慢，就是这个原因（看我这篇文章里有关 Zygote 启动进程的就知道慢了： [工作小笔记——Android 动态切换系统字体](http://mingming-killer.diandian.com/post/2014-10-23/40063248496 "工作小笔记——Android 动态切换系统字体")）。

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-service/3.png)
（realStartServiceLocked 后面的我省略了，第二种情况画过了）

我们回到 bringUpServiceLocked 那个分支那：

```java
    private final String bringUpServiceLocked(ServiceRecord r,
            int intentFlags, boolean execInFg, boolean whileRestarting) {

... ...

        // 这里接着前面那里就可以接着往下走了
        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        if (app == null) {
            // 调用 AM 的接口去启动服务的进程
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }

        // 把 ServiceRecord 添加到 mPendingServices
        // 这个玩意后面会有用的
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        // 后面的忽略了 ... ...
        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STATS) Slog.v(TAG, "Applying delayed stop (in bring up): " + r);
                stopServiceLocked(r);
            }
        }

        return null;
    }
```

这是第三种情况的分支，前面获取 ProcessRecord 是 null，就会往下走了。然后就要调用 AM 的 startProcessLocked 去启动服务所在的进程：

```java
    final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            String hostingType, ComponentName hostingName, boolean allowWhileBooting,
            boolean isolated, boolean keepIfLarge) {
        ProcessRecord app;
        // 先已经存在的进程记录集中取一下，看是不是进程已经存在了
        // 已经存在的话用已经存在的进程记录
        if (!isolated) {
            app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
        } else {
            // If this is an isolated process, it can't re-use an existing process.
            app = null;
        }
        // We don't have to do anything more if:
        // (1) There is an existing application record; and
        // (2) The caller doesn't think it is dead, OR there is no thread
        //     object attached to it so we know it couldn't have crashed; and
        // (3) There is a pid assigned to it, so it is either starting or
        //     already running.
        if (DEBUG_PROCESSES) Slog.v(TAG, "startProcess: name=" + processName
                + " app=" + app + " knownToBeDead=" + knownToBeDead
                + " thread=" + (app != null ? app.thread : null)
                + " pid=" + (app != null ? app.pid : -1));
        if (app != null && app.pid > 0) {
            if (!knownToBeDead || app.thread == null) {
                // We already have the app running, or are waiting for it to
                // come up (we have a pid but not yet its thread), so keep it.
                if (DEBUG_PROCESSES) Slog.v(TAG, "App already running: " + app);
                // If this is a new package in the process, add the package to the list
                app.addPackage(info.packageName, mProcessStats);
                return app;
            }

            // An application record is attached to a previous process,
            // clean it up now.
            if (DEBUG_PROCESSES || DEBUG_CLEANUP) Slog.v(TAG, "App died: " + app);
            handleAppDiedLocked(app, true, true);
        }

        String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;

        if (!isolated) {
            if ((intentFlags&Intent.FLAG_FROM_BACKGROUND) != 0) {
                // If we are in the background, then check to see if this process
                // is bad.  If so, we will just silently fail.
                if (mBadProcesses.get(info.processName, info.uid) != null) {
                    if (DEBUG_PROCESSES) Slog.v(TAG, "Bad process: " + info.uid
                            + "/" + info.processName);
                    return null;
                }
            } else {
                // When the user is explicitly starting a process, then clear its
                // crash count so that we won't make it bad until they see at
                // least one crash dialog again, and make the process good again
                // if it had been bad.
                if (DEBUG_PROCESSES) Slog.v(TAG, "Clearing bad process: " + info.uid
                        + "/" + info.processName);
                mProcessCrashTimes.remove(info.processName, info.uid);
                if (mBadProcesses.get(info.processName, info.uid) != null) {
                    EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                            UserHandle.getUserId(info.uid), info.uid,
                            info.processName);
                    mBadProcesses.remove(info.processName, info.uid);
                    if (app != null) {
                        app.bad = false;
                    }
                }
            }
        }

        // 不存在的话，new 一个 ProcessRecord 出来的，然后保存一下
        if (app == null) {
            app = newProcessRecordLocked(info, processName, isolated);
            if (app == null) {
                Slog.w(TAG, "Failed making new process record for "
                        + processName + "/" + info.uid + " isolated=" + isolated);
                return null;
            }
            mProcessNames.put(processName, app.uid, app);
            if (isolated) {
                mIsolatedProcesses.put(app.uid, app);
            }
        } else {
            // If this is a new package in the process, add the package to the list
            app.addPackage(info.packageName, mProcessStats);
        }

        // If the system is not ready yet, then hold off on starting this
        // process until it is.
        if (!mProcessesReady
                && !isAllowedWhileBooting(info)
                && !allowWhileBooting) {
            if (!mProcessesOnHold.contains(app)) {
                mProcessesOnHold.add(app);
            }
            if (DEBUG_PROCESSES) Slog.v(TAG, "System not ready, putting on hold: " + app);
            return app;
        }

        // 又是马甲调用
        startProcessLocked(app, hostingType, hostingNameStr);
        return (app.pid != 0) ? app : null;
    }
```

前面做了一些时候存在的处理，关键的还是最后一个，又调用到另一个同名不同参数的 startProcessLocked 里去了：

```java
    private final void startProcessLocked(ProcessRecord app,
            String hostingType, String hostingNameStr) {
        if (app.pid > 0 && app.pid != MY_PID) {
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.remove(app.pid);
                mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
            }
            app.setPid(0);
        }

        if (DEBUG_PROCESSES && mProcessesOnHold.contains(app)) Slog.v(TAG,
                "startProcessLocked removing on hold: " + app);
        mProcessesOnHold.remove(app);
        
        // 更新下 cpu 统计        
        updateCpuStats();
                
        try {
            int uid = app.uid;
            
            // 获取下这个应用的安装位置（对应 class 文件的位置）
            // 以及用户组等信息
            int[] gids = null;
            int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
            if (!app.isolated) {
                int[] permGids = null;
                try {
                    final PackageManager pm = mContext.getPackageManager();
                    permGids = pm.getPackageGids(app.info.packageName);
               
                    if (Environment.isExternalStorageEmulated()) {
                        if (pm.checkPermission(
                                android.Manifest.permission.ACCESS_ALL_EXTERNAL_STORAGE,
                                app.info.packageName) == PERMISSION_GRANTED) {
                            mountExternal = Zygote.MOUNT_EXTERNAL_MULTIUSER_ALL;
                        } else {
                            mountExternal = Zygote.MOUNT_EXTERNAL_MULTIUSER;
                        }
                    }
                } catch (PackageManager.NameNotFoundException e) {
                    Slog.w(TAG, "Unable to retrieve gids", e);
                }

                /*
                 * Add shared application GID so applications can share some
                 * resources like shared libraries
                 */
                if (permGids == null) {
                    gids = new int[1];
                } else {
                    gids = new int[permGids.length + 1];
                    System.arraycopy(permGids, 0, gids, 1, permGids.length);
                }
                gids[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
            }
            if (mFactoryTest != SystemServer.FACTORY_TEST_OFF) {
                if (mFactoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL
                        && mTopComponent != null
                        && app.processName.equals(mTopComponent.getPackageName())) {
                    uid = 0;
                }
                if (mFactoryTest == SystemServer.FACTORY_TEST_HIGH_LEVEL
                        && (app.info.flags&ApplicationInfo.FLAG_FACTORY_TEST) != 0) {
                    uid = 0;
                }
            }
            // 设置下 zygote 调试信息标志
            int debugFlags = 0;
            if ((app.info.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0) {
                debugFlags |= Zygote.DEBUG_ENABLE_DEBUGGER;
                // Also turn on CheckJNI for debuggable apps. It's quite
                // awkward to turn on otherwise.
                debugFlags |= Zygote.DEBUG_ENABLE_CHECKJNI;
            }
            // Run the app in safe mode if its manifest requests so or the
            // system is booted in safe mode.
            if ((app.info.flags & ApplicationInfo.FLAG_VM_SAFE_MODE) != 0 ||
                Zygote.systemInSafeMode == true) {
                debugFlags |= Zygote.DEBUG_ENABLE_SAFEMODE;
            }
            if ("1".equals(SystemProperties.get("debug.checkjni"))) {
                debugFlags |= Zygote.DEBUG_ENABLE_CHECKJNI;
            }
            if ("1".equals(SystemProperties.get("debug.jni.logging"))) {
                debugFlags |= Zygote.DEBUG_ENABLE_JNI_LOGGING;
            }
            if ("1".equals(SystemProperties.get("debug.assert"))) {
                debugFlags |= Zygote.DEBUG_ENABLE_ASSERT;
            }

            // 调用 Procss 去启动进程
            // Start the process.  It will either succeed and return a result containing
            // the PID of the new process, or else throw a RuntimeException.
            Process.ProcessStartResult startResult = Process.start("android.app.ActivityThread",
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, null);

// 后面不贴了，和这里关系不大的
... ...

            }
        } catch (RuntimeException e) {
            // XXX do better error recovery.
            app.setPid(0);
            Slog.e(TAG, "Failure starting process " + app.processName, e);
        }
    }
```

启动进程前面要准备比较多参数。最后是调用 Process.java 中的 start 接口去启动进程的。进程通过 Zygote fork 出来的，细节可以看我前面说的我的那篇工作小笔记。这里发过去的要加载的 java 类是 ActivityThread，前面说了这个是 android 应用程序进程的主线程，里面有 java 的 main 函数。然后通过 Zygote fork 后，Service 的进程就跑起来了，然后会跑到 AT 的 main 函数里：

```java
    public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);  

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        Security.addProvider(new AndroidKeyStoreProvider());

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();    

        // 关键看这里， attch 这个函数
        ActivityThread thread = new ActivityThread(); 
        thread.attach(false); 

        if (sMainThreadHandler == null) { 
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

这个 main 函数，new 了一个 AT 对象出来，然后跑了 AT 的 attch 函数：

```java
    private void attach(boolean system) {
        sCurrentActivityThread = this; 
        mSystemThread = system;
        // 系统进程走的下面那个        
        if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {            
                    ensureJitEnabled();            
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());        
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                // 主要是这个，调用了 AM 的 attchApplication
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) { 
                // Ignore
            }
        } else {       
            // 系统进程可能在别的地方 attchApplication 了吧
            // 看注释不想给系统进程设置崩溃弹出的那个对话框       
            // Don't set application object here -- if the system crashes,
            // we can't display an alert, we just want to die die die.
            android.ddm.DdmHandleAppName.setAppName("system_process",
                                                    UserHandle.myUserId());        
            try {
                mInstrumentation = new Instrumentation();
                ContextImpl context = new ContextImpl();
                context.init(getSystemContext().mPackageInfo, null, this);
                Application app = Instrumentation.newApplication(Application.class, context);
                mAllApplications.add(app);     
                mInitialApplication = app;     
                app.onCreate();                
            } catch (Exception e) {        
                throw new RuntimeException(    
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }

        // 后面继续忽略 ... ...
        // add dropbox logging to libcore
        DropBox.setReporter(new DropBoxReporter());

        ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
            @Override
            public void onConfigurationChanged(Configuration newConfig) {
                synchronized (mResourcesManager) {
                    // We need to apply this change to the resources
                    // immediately, because upon returning the view
                    // hierarchy will be informed about it.
                    if (mResourcesManager.applyConfigurationToResourcesLocked(newConfig, null)) {
                        // This actually changed the resources!  Tell
                        // everyone about it.
                        if (mPendingConfiguration == null ||
                                mPendingConfiguration.isOtherSeqNewer(newConfig)) {
                            mPendingConfiguration = newConfig;

                            sendMessage(H.CONFIGURATION_CHANGED, newConfig);
                        }
                    }
                }
            }
            @Override
            public void onLowMemory() {
            }
            @Override
            public void onTrimMemory(int level) {
            }
        });
    }
```

这里主要是看调用 AM 的 attachApplication，然后又转到 AM 里面了：

```java
    @Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity(); 
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

是个马甲，但是注意锁。这里我们要回到前面 AM 调用 Process 去 start 服务进程那里，因为那里也是有锁的。所以 AT 这里的 atttach 应该是会被锁住的（binder 的多线程问题见我的另一篇线程篇）。我们先回去到 AM 调用完 Process.start 那里。回去看前面的代码，调用 Process.start 就会从 AM 返回到 bringUpServiceLocked，然后后面把 ServiceRecord 添加到 mPendingServices，然后返回 null 到 bindServiceLocked 继续往下执行，但是到 requestServiceBindingLocked 那里回由于 ServiceRecord 的 app 为 null 而失败，因为进程虽然跑起来了，但是服务还没执行。然后这个 Client 的 AM 的 bindService 应该就调用结束了。然后 AM 的锁就应该没了，现在可以继续回到 Service 的 AT 调用 AM 的 attachApplication 那里继续了。

这里是调用了另一个 attachApplicationLocked:

```java
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

// 这函数太长了，前面都是和这篇关系不大的，忽略先
... ...

        // 主要看这个，果然又调用到 ActiveServices 里面去了
        // Find any services that should be running in this process...
        if (!badApp) {
            try {
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                badApp = true;
            }
        }

... ...

        if (!didSomething) {
            updateOomAdjLocked();
        }

        return true;
    }
```

我们继续去 ActiveServices 里面去看：

```java
    boolean attachApplicationLocked(ProcessRecord proc, String processName) throws Exception {
        boolean didSomething = false;
        // 前面那个 mPendingServices 作用就是这里啦
        // Collect any services that are waiting for this process to come up.
        if (mPendingServices.size() > 0) {
            ServiceRecord sr = null;
            try {   
                for (int i=0; i<mPendingServices.size(); i++) {
                    sr = mPendingServices.get(i);
                    if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                            || !processName.equals(sr.processName))) {
                        continue;
                    }
            
                    // 调用 realStartServiceLocked 去启动服务了
                    mPendingServices.remove(i);
                    i--;
                    proc.addPackage(sr.appInfo.packageName, mAm.mProcessStats);
                    realStartServiceLocked(sr, proc, sr.createdFromFg);
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.w(TAG, "Exception in new application when starting service "
                        + sr.shortName, e);
                throw e;
            }   
        }       
        // Also, if there are any services that are waiting to restart and
        // would run in this process, now is a good time to start them.  It would
        // be weird to bring up the process but arbitrarily not let the services
        // run at this point just because their restart time hasn't come up.
        if (mRestartingServices.size() > 0) {
            ServiceRecord sr = null;
            for (int i=0; i<mRestartingServices.size(); i++) {
                sr = mRestartingServices.get(i);
                if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                        || !processName.equals(sr.processName))) {
                    continue;
                }
                mAm.mHandler.removeCallbacks(sr.restarter);
                mAm.mHandler.post(sr.restarter);
            }
        }
        return didSomething;
    }
```

还记得前面 bringUpServiceLocked 调用完 AM startProcessLocked 把 ServiceRecord 添加到 mPendingServices 中，我说这个东西后面会有用的。就是在这里用的。前面也看到了，bindService 继续完后走，但是由于服务还没执行，所以无法进行绑定操作。所以就拿了个东西把要绑定的服务信息保存起来，当服务所在的进程跑起来去 attach 上下文的时候去检测这个东西，如果有的话就去启动需要绑定的服务。怪不得叫 Pending XX ，其实就是延迟处理的意思。怪不得前面分析 realStartServiceLocked 为什么最后那里还有 requestServiceBindingsLocked 的操作，明明 bindServiceLocked 后面已经有了，原来是这里用的啊。

realStartServiceLocked 前面分析过了，回去看看吧，这里不重复说了。到这里 bindService 三种情况都说完了。可以看得出来第三种情况是最麻烦的，得拿个东西保存一下要绑定的信息，然后去启动服务所在的进程，等进程跑主线程的 main 函数再去执行服务代码，然后才能绑定。

## 总结
普通服务的 binder 对象的传递就分析完了。其实普通服务 binder 对象的获取原理上和 SS 是差不多的，SS 是向 SM 注册，然后 SM 保存了 SS binder 的 handle 值，然后通过向 SM 取 SS 的 binder 对象。而普通服务则是在启动的时候在 AM 保存了 ServiceRecord，然后普通应用通过 AM 的 bindService 去让 AM 向普通服务要 binder 对象，然后传给普通应用。感觉对于普通应用（服务）来说，AM 就牵了根线，有点像 SM 的感觉了。还有一点区别，SS 是一开机就启动的，正常情况下一直运行的，所以不存在说服务还没运行一说，所以 SM 一直返回之前保存的 handle 值就行了。但是普通服务没这待遇，开机不一定能启动不说，还可能由于系统资源不足而被杀死，所以 AM 在 bindService 的时候还得兼顾进程、服务还没启动的情况。

这就是铁饭碗和临时工的区别啊 -_-|| 。


