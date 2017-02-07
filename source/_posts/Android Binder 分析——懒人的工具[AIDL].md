title: Android Binder 分析——懒人的工具（AIDL）
date: 2015-01-28 21:53:16
updated: 2017-02-07 21:47:44
categories: [Android Framework]
tags: [android]
---

前面说了 binder 的原理，和 parcel，当时也说了实现 IBinder 接口（Bp 端、Bn 端）代码都非常的机械，人工写的话，不仅浪费时间，而且很有可能会出错（复制、粘贴后忘记改某个地方）。所以 android 弄了代码自动生成的东西。然后还美名其曰 XX 语言—— AIDL（Android Interface Definition Language）。这里不研究这个东西的实现代码，只是说说有这么个东西可以用，然后就是其实你不用也可以。

照例先把相关源码位置啰嗦一下（4.4）：

```bash
# AM 接口实现相关代码
frameworks/base/core/android/app/ActivityManager.java
frameworks/base/core/android/app/ActivityManagerNative.java
frameworks/base/core/android/app/IActivityManager.java

# AMS 代码
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java

# WM 接口实现相关代码
frameworks/base/core/android/view/WindowManager.java
frameworks/base/core/android/view/IWindowManager.aidl

# WM 接口自动生成的代码
out/target/common/obj/JAVA_LIBRARIES/framework-base_intermediates/src/com/android/view/IWindowManager.java

# WMS 代码
services/java/com/android/server/wm/WindowManagerService.java

# Context 相关代码
frameworks/base/core/java/android/app/ContextImpl.java
```

## java 层
其实 aidl 这个东西只能在 java 成使用，native 层是用另外一个类似的东西。但是前面也好说了，这个东西就是一个代码自动生成器，你自己手动写，不用也是可以的。我们就来看下 SystemServer（SS）中的典型吧。

### 纯手工打造
SS 中手动写 binder 接口代码的典型是 ActivityManagerService（AMS）。binder 接口需要实现什么东西，前面原理篇讲得很清楚了，忘记了的回去看下。

ActivityManager 是 sdk 提供给上层 app 使用的接口。ActivityManagerNative （AMN）接口这边的实现（接口端也是要写代码的，调用 binder 接口向服务端发送请求）。IActiviyManager 就是 AMS 继承 binder 定义的接口。最后服务端实现不止 ActivityManagerService.java 这一个文件，有一个专门的 am 包咧，这里不是分析 AMS ，所以列个代表。

ActivitManager.java 不说啥了，就是 sdk 公开的那些接口，它里面的接口实现全都是通过 ActivityManagerNative.getDefault() 转向 AMN 了。所以 interface 端的主要实现在 AMN 中：

```java
// ActivityManager.java ===============================

    // 挂马甲，真正的实现在 AMN 里面
    public void moveTaskToFront(int taskId, int flags, Bundle options) {
        try {
            ActivityManagerNative.getDefault().moveTaskToFront(taskId, flags, options);
        } catch (RemoteException e) { 
            // System dead, we will be dead too soon!
        }    
    }

// ActivityManagerNative.java ===========================

    /*
     * Retrieve the system's default/global activity manager.
     */
    static public IActivityManager getDefault() {
        return gDefault.get();
    }

    // 单例的实现方式，节省内存，速度也更快
    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            // 向 SM 取 AM 的 binder 对象
            // AM向SM注册的名字就是 "activity"，不过不用 static 变量不好吧
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }    
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am); 
            }    
            return am;
        }    
    };
```

ActivityManagerNative 采用单列设计模式，保证一个进程中只有一份 Bp。这个做应该是提高效率，因为 app 里面先不说应用自己，framework 里面经常随手 Context.getSystemService("activity")，如果每次都创建新实例，很浪费内存的，而且慢。

来看下 ActivityManagerNative 的继承关系：

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager {}
public class Binder implements IBinder {}
public interface IActivityManager extends IInterface {}
```

AMN 继承自 Binder （实现 IBinder 接口，所有的远程接口都是这个），实现 IActivityManager， IActivityManager 继承自 IInterface 这个东西就是 binder 提供给客户端的接口，使用者通过继承这个类定义自己的接口。

来看下 AMS 定义的接口：

```java
public interface IActivityManager extends IInterface {

    // 定义的接口
    public int startActivity(IApplicationThread caller,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options) throws RemoteException;
    public int startActivityAsUser(IApplicationThread caller,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options, int userId) throws RemoteException;
    public WaitResult startActivityAndWait(IApplicationThread caller,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int flags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options, int userId) throws RemoteException;
    public int startActivityWithConfig(IApplicationThread caller,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho,
            int requestCode, int startFlags, Configuration newConfig,
            Bundle options, int userId) throws RemoteException;
    public int startActivityIntentSender(IApplicationThread caller,
            IntentSender intent, Intent fillInIntent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode,
            int flagsMask, int flagsValues, Bundle options) throws RemoteException;
    public boolean startNextMatchingActivity(IBinder callingActivity,
            Intent intent, Bundle options) throws RemoteException;

... ....

    String descriptor = "android.app.IActivityManager";

    // 这个其实和上面的接口对应，一般一个接口对应一个 transaction code 。
    // 在 interface 的实现 onTransact 中就能看到这些东西的用处了。 
    // Please keep these transaction codes the same -- they are also
    // sent by C++ code.
    int START_RUNNING_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION;
    int HANDLE_APPLICATION_CRASH_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+1;
    int START_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+2;
    int UNHANDLED_BACK_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+3;
    int OPEN_CONTENT_URI_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+4;

    // Remaining non-native transaction codes.
    int FINISH_ACTIVITY_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+10;
    int REGISTER_RECEIVER_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+11;
    int UNREGISTER_RECEIVER_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+12;
    int BROADCAST_INTENT_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+13;
    int UNBROADCAST_INTENT_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+14;

... ...

}
```

AMN 实现上面的接口定义，里面还又个 ActivityManagerProxy（AMP），Proxy，Binder Proxy 啊，Bp 端的实现。android 喜欢把 Bp 端和 Bn 端的接口放在一起实现：

```java
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{
    /*
     * Cast a Binder object into an activity manager interface, generating
     * a proxy if needed.
     */
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        // 返回的是内部类 Proxy，这才能叫 Bp 么 
        return new ActivityManagerProxy(obj); 
    }

    // 这部分代码是 service 实现，上面 IActivityManager 定义那一堆
    // transcation code 就是用在这里了， switch case 用的 -_-||
    // 注意这里的代码是运行在 service 端的，所以里面的调用的函数是 service
    // 里面真正实现命令的函数（看后面的分析）。
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case START_ACTIVITY_TRANSACTION:
        {
            // 读的顺序要和下面写的一样，parcel 篇有说到的
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            IBinder resultTo = data.readStrongBinder();
            String resultWho = data.readString();
            int requestCode = data.readInt();
            int startFlags = data.readInt();
            String profileFile = data.readString();
            ParcelFileDescriptor profileFd = data.readInt() != 0
                    ? data.readFileDescriptor() : null;
            Bundle options = data.readInt() != 0
                    ? Bundle.CREATOR.createFromParcel(data) : null;
            int result = startActivity(app, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags,
                    profileFile, profileFd, options);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
        }
... ....
        }   
                    
        return super.onTransact(code, data, reply, flags);
    }

... ...

class ActivityManagerProxy implements IActivityManager
{

    // 这部分的代码是 interface 的现实，通过 binder 向 service 发送请求
    public int startActivity(IApplicationThread caller, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, String profileFile,
            ParcelFileDescriptor profileFd, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        data.writeString(profileFile);
        if (profileFd != null) {
            data.writeInt(1);
            profileFd.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }

... ...

}

}
```

最后来看看 Bn 端这边的实现：

```java
// 这里直接就继承 AMN 了，后面对比下 WMS 就发现继承的不一样
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException { 
        // 针对特殊请求要做下处理
        if (code == SYSPROPS_TRANSACTION) {
            // We need to tell all apps about the system property change.
            ArrayList<IBinder> procs = new ArrayList<IBinder>();
            synchronized(this) {
                for (SparseArray<ProcessRecord> apps : mProcessNames.getMap().values()) {
                    final int NA = apps.size();
                    for (int ia=0; ia<NA; ia++) {
                        ProcessRecord app = apps.valueAt(ia);
                        if (app.thread != null) {
                            procs.add(app.thread.asBinder());
                        }
                    }
                }
            }
    
            int N = procs.size();
            for (int i=0; i<N; i++) {
                Parcel data2 = Parcel.obtain();
                try {
                    procs.get(i).transact(IBinder.SYSPROPS_TRANSACTION, data2, null, 0);
                } catch (RemoteException e) { 
                }
                data2.recycle();
            }
        }
        try {
            // 其余的直接调用父类的处理，就是上面 AMN 里的实现
            return super.onTransact(code, data, reply, flags);
        } catch (RuntimeException e) {
            // The activity manager only throws security exceptions, so let's
            // log all others.
            if (!(e instanceof SecurityException)) {
                Slog.e(TAG, "Activity Manager Crash", e);
            }
            throw e;
        }
    }

}
```

上面只是列举了 AM 中的一个接口的实现，剩下那一堆一个一个手动写吧。很麻烦是不是，所以就可以使用代码自动生成工具了（aidl）：

### 代码自动生成
看了上面的 binder 接口的实现方法，感觉对很麻烦。仔细观察上面的 AMS 的代码，会发现其实有不少东西可以由机器来完成的。aidl 就是干这个事情的，下面来看看 SS （其实 SS 中大部分 Servier 是用 aidl 来写接口的）中使用 aidl 的典型： WindowManagerServices （WMS）。

WindowManager.java 这个 sdk 对上层应用提供的接口。然后这里有个 IWindowManager.aidl 的文件。虽然说是代码自动生成，但是开发者还是要写点东西的，人家好歹叫一个语言咧，不写点代码怎么行（傻瓜相机还要按快门咧）。只要在这个 aidl 文件中把要公开的接口按照一定的格式写就行了，那个格式感觉和 java 代码基本一样：

```java
/*
 * System private interface to the window manager.
 *
 * {@hide}
 */
interface IWindowManager
{
    /*
     * ===== NOTICE =====
     * The first three methods must remain the first three methods. Scripts
     * and tools rely on their transaction number to work properly.
     */
    // This is used for debugging
    boolean startViewServer(int port);   // Transaction #1
    boolean stopViewServer();            // Transaction #2
    boolean isViewServerRunning();       // Transaction #3

    IWindowSession openSession(in IInputMethodClient client,
            in IInputContext inputContext);
    boolean inputMethodClientHasFocus(IInputMethodClient client);

    void setForcedDisplaySize(int displayId, int width, int height);
    void clearForcedDisplaySize(int displayId);
    void setForcedDisplayDensity(int displayId, int density);
    void clearForcedDisplayDensity(int displayId);

    void setOverscan(int displayId, int left, int top, int right, int bottom);

    // These can only be called when holding the MANAGE_APP_TOKENS permission.
    void pauseKeyDispatching(IBinder token);
    void resumeKeyDispatching(IBinder token);
    void setEventDispatching(boolean enabled);
    void addWindowToken(IBinder token, int type);
    void removeWindowToken(IBinder token);

... ...

}
```

和上面 AM 的代码对比下，这简直太简单了。然后 Bn 端（WindowManagerService.java）实现真正的业务就可以了（AM 除了要实现 binder 接口，这一步也是不能少的）：

```java
// 注意继承的类 IWindowManager.Stub
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs,
                DisplayManagerService.WindowManagerFuncs, DisplayManager.DisplayListener {

... ...

    // 实现接口业务
    /*
     * Starts the view server on the specified port.
     *
     * @param port The port to listener to.
     *
     * @return True if the server was successfully started, false otherwise.
     *
     * @see com.android.server.wm.ViewServer
     * @see com.android.server.wm.ViewServer#VIEW_SERVER_DEFAULT_PORT
     */
    public boolean startViewServer(int port) {
        if (isSystemSecure()) {
            return false;
        }

        if (!checkCallingPermission(Manifest.permission.DUMP, "startViewServer")) {
            return false;
        }

        if (port < 1024) {
            return false;
        }

        if (mViewServer != null) {
            if (!mViewServer.isRunning()) {
                try {
                    return mViewServer.start();
                } catch (IOException e) {
                    Slog.w(TAG, "View server did not start");
                }
            }
            return false;
        }

        try {
            mViewServer = new ViewServer(this, port);
            return mViewServer.start();
        } catch (IOException e) {
            Slog.w(TAG, "View server did not start");
        }
        return false;
    }

... ... 

    @Override
    public void removeWindowToken(IBinder token) {
        if (!checkCallingPermission(android.Manifest.permission.MANAGE_APP_TOKENS,
                "removeWindowToken()")) {
            throw new SecurityException("Requires MANAGE_APP_TOKENS permission");
        }  

... ...
    }

... ...

}
```

自动生成的 java 代码在 framework 的源码是找不到的，是编译的时候 aidl 工具动态生成的（这里不分析这个工具），在 `out/target/common/obj/JAVA_LIBRARIES/framework-base_intermediates/src` 下面，和放 aidl 文件位置对应（WM 的是在 src/core/java/android/view/IWindowManager.java）。然后 IWindowManager.Stub 就在这个文件里面：

```java
public static abstract class Stub extends android.os.Binder implements android.view.IWindowManager
{
private static final java.lang.String DESCRIPTOR = "android.view.IWindowManager";
/** Construct the stub at attach it to the interface. */
public Stub()
{
this.attachInterface(this, DESCRIPTOR);
}
/*
 * Cast an IBinder object into an android.view.IWindowManager interface,
 * generating a proxy if needed.
 */
public static android.view.IWindowManager asInterface(android.os.IBinder obj) 
{
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin!=null)&&(iin instanceof android.view.IWindowManager))) {
return ((android.view.IWindowManager)iin);
}
return new android.view.IWindowManager.Stub.Proxy(obj);
}
@Override public android.os.IBinder asBinder()
{
return this;
}
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
switch (code)
{
case INTERFACE_TRANSACTION:
{
reply.writeString(DESCRIPTOR);
return true;
}
case TRANSACTION_startViewServer:
{
// 机器生成的代码顺序肯定不会出错
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
boolean _result = this.startViewServer(_arg0);
reply.writeNoException();
reply.writeInt(((_result)?(1):(0)));
return true;
}

... ... 

case TRANSACTION_removeWindowToken:
{
data.enforceInterface(DESCRIPTOR);
android.os.IBinder _arg0;
// 会根据不同的参数类型调用 parcel 对应的打包函数
_arg0 = data.readStrongBinder();
this.removeWindowToken(_arg0);
reply.writeNoException();
return true;
}

... ...

}
return super.onTransact(code, data, reply, flags);
}

private static class Proxy implements android.view.IWindowManager
{
private android.os.IBinder mRemote;
Proxy(android.os.IBinder remote)
{
mRemote = remote;
}
@Override public android.os.IBinder asBinder()
{
return mRemote;
}
public java.lang.String getInterfaceDescriptor()
{
return DESCRIPTOR;
}
/*
     * ===== NOTICE =====
     * The first three methods must remain the first three methods. Scripts
     * and tools rely on their transaction number to work properly.
     */// This is used for debugging

@Override public boolean startViewServer(int port) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
boolean _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(port);
mRemote.transact(Stub.TRANSACTION_startViewServer, _data, _reply, 0);
_reply.readException();
_result = (0!=_reply.readInt());
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
// Transaction #1

... ...

@Override public void addWindowToken(android.os.IBinder token, int type) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeStrongBinder(token);
_data.writeInt(type);
mRemote.transact(Stub.TRANSACTION_addWindowToken, _data, _reply, 0);
_reply.readException();
}
finally {
_reply.recycle();
_data.recycle();
}
}

... ...

}

// 自动计数
static final int TRANSACTION_startViewServer = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
static final int TRANSACTION_stopViewServer = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
static final int TRANSACTION_isViewServerRunning = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
static final int TRANSACTION_openSession = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
static final int TRANSACTION_inputMethodClientHasFocus = (android.os.IBinder.FIRST_CALL_TRANSACTION + 4);
static final int TRANSACTION_setForcedDisplaySize = (android.os.IBinder.FIRST_CALL_TRANSACTION + 5);
static final int TRANSACTION_clearForcedDisplaySize = (android.os.IBinder.FIRST_CALL_TRANSACTION + 6);
static final int TRANSACTION_setForcedDisplayDensity = (android.os.IBinder.FIRST_CALL_TRANSACTION + 7);
static final int TRANSACTION_clearForcedDisplayDensity = (android.os.IBinder.FIRST_CALL_TRANSACTION + 8);

... ...

}
```

这个东西除了机器生成的排版比较抱歉以外，其它和上面的 AMS 的 AMN 基本上是一个模子印出来的。aidl 自动生成的代码能够根据参数的类型自动调用 parcel 对应的数据打包函数。基本类型都支持，IBinder 类型的也有特殊的函数。如果是自定义类型的话，需要自己实现 Parcelable 接口。

aidl 的参数定义有2个比较有意思的关键字 in 和 out。如果参数前面加了 in 则表示这是个输入型参数，aidl 生成的代码会先通过 Parcelable 接口读取 Bp 端传过来的对象数据，然后调用对应类型的 Parcelable 接口的 CREATOR.createFromParcel 重新示例化对象，达到跨进程传递自定义类型数据的目的。

如果是 out 则表示这个是输出型参数，aidl 生成的代码会调用该类的默认构造参数创建一个对象，然后传递给 Bn 端的真正函数， Bn 端的这个函数的实现，要负责填充对应的数据，然后调用完成后， aidl 的代码会调用 Parcelable 的接口，写入 binder，返回给 Bp 端调用者。来看下 WMS 里面的例子就比较好理解了：

```java
interface IWindowManager
{

... ...
   
    // 第一个 Bitmap 是 in 类型的
    void overridePendingAppTransitionThumb(in Bitmap srcThumb, int startX, int startY,
            IRemoteCallback startedCallback, boolean scaleUp);

    // 第二 List 数组是 out 类型的
    /*
     * Gets the infos for all visible windows.
     */
    void getVisibleWindowsForDisplay(int displayId, out List<WindowInfo> outInfos);

... ...

}

... ...

public interface IWindowManager extends android.os.IInterface
{
/** Local-side IPC implementation stub class. */
public static abstract class Stub extends android.os.Binder implements android.view.IWindowManager
{

@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
switch (code)

... ...

case TRANSACTION_overridePendingAppTransitionThumb:
{
data.enforceInterface(DESCRIPTOR);
android.graphics.Bitmap _arg0;
// 通过 Parcel 先读取 Bp 端传过来的参数
if ((0!=data.readInt())) {
// 然后再根据对象的 Parcel 接口，用刚刚的数据重新实例化对象，达到跨进程传递 Object 的目的
_arg0 = android.graphics.Bitmap.CREATOR.createFromParcel(data);
}
else {
_arg0 = null;
}
int _arg1;
_arg1 = data.readInt();
int _arg2;
_arg2 = data.readInt();
android.os.IRemoteCallback _arg3;
_arg3 = android.os.IRemoteCallback.Stub.asInterface(data.readStrongBinder());
boolean _arg4;
_arg4 = (0!=data.readInt());
this.overridePendingAppTransitionThumb(_arg0, _arg1, _arg2, _arg3, _arg4);
reply.writeNoException();
return true;
}

... ...

case TRANSACTION_getVisibleWindowsForDisplay:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
// 创建新对象以便 Bn 端实现函数能填充数据
java.util.List<android.view.WindowInfo> _arg1;
_arg1 = new java.util.ArrayList<android.view.WindowInfo>();
this.getVisibleWindowsForDisplay(_arg0, _arg1);
reply.writeNoException();
// 通过 Parcel 接口，把填充好的输入返回给 Bp 端
reply.writeTypedList(_arg1);
return true;
}

... ...

return super.onTransact(code, data, reply, flags);
}

.... ....

}
```

上面那些东西用机械化的代码自动生成比手写第一快、第二不会出错。SS 中大部分是用 aidl 来实现接口的，除了少数的几个（有可能就是手工写那个几个太麻烦了，才诞生了 aidl 吧）。上层第三应用的服务应该也是可用手工写的，但是估计比 framework 还要麻烦，因为好像有一些接口是 hide 的。


---

这里说个小插曲，在 core/java/com/android/internal/statusbar/IStatusBar.aidl 中有 **oneway** 这一个声明：

```java
/* @hide */
oneway interface IStatusBar
{
    void setIcon(int index, in StatusBarIcon icon);
    void removeIcon(int index);
    void addNotification(IBinder key, in StatusBarNotification notification);
    void updateNotification(IBinder key, in StatusBarNotification notification);
    void removeNotification(IBinder key);
    void disable(int state);
    void animateExpandNotificationsPanel();
    void animateExpandSettingsPanel();
    void animateCollapsePanels();
    void setSystemUiVisibility(int vis, int mask);
    void topAppWindowChanged(boolean menuVisible);
    void setImeWindowStatus(in IBinder token, int vis, int backDisposition);
    void setHardKeyboardStatus(boolean available, boolean enabled);
    void toggleRecentApps();
    void preloadRecentApps();
    void cancelPreloadRecentApps();
}
```

还记得前面说 binder 传送过程中有一种 no reply 的情况么。如果声明了这个关键字，那么 aidl 什么的代码是这样的：

```java
@Override public void setIcon(int index, com.android.internal.statusbar.StatusBarIcon icon) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(index);
if ((icon!=null)) {
_data.writeInt(1);
icon.writeToParcel(_data, 0); 
}
else {
_data.writeInt(0);
}
// 发现和上面没加 oneway 的区别了没， reply 传过去是 null 
mRemote.transact(Stub.TRANSACTION_setIcon, _data, null, android.os.IBinder.FLAG_ONEWAY);
}
finally {
_data.recycle();
}
}
```

如果你的 binder 接口不需要返回值，oneway 声明会快一点，**当然如果你的 binder 接口需要返回值，记得把这个声明去掉，否则会编译报错的（生成的代码 reply 这个变量会没初始化就当返回值用了）**。目前 framework 中我就看到 IStatusBar 加这个声明。

---


这里 aidl 自动生成代码还忽略了一点。上面手工写的，不是搞了个单例模式么，但是 aidl 生成没用这种模式，是不是自动生成的代码，效率就低了些呢。这里从我们经常用的 Context 接口 getSystemService 说起：

```java
// ContextImpl.java ==============================

    @Override
    public Object getSystemService(String name) {
        ServiceFetcher fetcher = SYSTEM_SERVICE_MAP.get(name);
        return fetcher == null ? null : fetcher.getService(this);
    }

``` 

我们先来看下 ServiceFetcher 是什么东西：

```java

    // The system service cache for the system services that are
    // cached per-ContextImpl.  Package-scoped to avoid accessor
    // methods.
    final ArrayList<Object> mServiceCache = new ArrayList<Object>();

    /*
     * Override this class when the system service constructor needs a
     * ContextImpl.  Else, use StaticServiceFetcher below.
     */
    /*package*/ static class ServiceFetcher {
        int mContextCacheIndex = -1;   

        /*
         * Main entrypoint; only override if you don't need caching.
         */
        public Object getService(ContextImpl ctx) {
            // 原来是缓存啊
            ArrayList<Object> cache = ctx.mServiceCache;
            Object service;
            synchronized (cache) {
                // 刚开始拿 null 的来占位         
                if (cache.size() == 0) {       
                    // Initialize the cache vector on first access.
                    // At this point sNextPerContextServiceCacheIndex
                    // is the number of potential services that are
                    // cached per-Context.         
                    for (int i = 0; i < sNextPerContextServiceCacheIndex; i++) {
                        cache.add(null);               
                    }
                } else {
                    // 在缓存中取 SS 的 Bp
                    service = cache.get(mContextCacheIndex);
                    if (service != null) {
                        return service;
                    }
                }
                // 取不到的话，new 一个出来，然后保存在缓存中
                service = createService(ctx);
                cache.set(mContextCacheIndex, service);
                return service;
            }
        }

        /*
         * Override this to create a new per-Context instance of the
         * service.  getService() will handle locking and caching.
         */
        public Object createService(ContextImpl ctx) {
            throw new RuntimeException("Not implemented");
        }
    }
```

看到上就差不多能明白了，aidl 的 get Bp 的优化是在 Context 中做的。然后继续看那个 `SYSTEM_SERVICE_MAP`：

```java
    // static 变量来的，一个进程一个
    private static final HashMap<String, ServiceFetcher> SYSTEM_SERVICE_MAP =
            new HashMap<String, ServiceFetcher>();
        
    // 这个 registerService 就是往 SYSTEM_SERVICE_MAP 中加东西
    private static int sNextPerContextServiceCacheIndex = 0;
    private static void registerService(String serviceName, ServiceFetcher fetcher) {
        if (!(fetcher instanceof StaticServiceFetcher)) {
            fetcher.mContextCacheIndex = sNextPerContextServiceCacheIndex++;
        }   
        SYSTEM_SERVICE_MAP.put(serviceName, fetcher);
    }
                    
    // This one's defined separately and given a variable name so it
    // can be re-used by getWallpaperManager(), avoiding a HashMap
    // lookup.
    private static ServiceFetcher WALLPAPER_FETCHER = new ServiceFetcher() {
            public Object createService(ContextImpl ctx) {
                return new WallpaperManager(ctx.getOuterContext(),
                        ctx.mMainThread.getHandler());
            }};    

    // static 代码块，类只要加载就会跑这段。
    // 这里就把那一堆 SS 都添加到 SYSTEM_SERVICE_MAP 的 map 中去了。
    // 下面还有一堆，我只贴一部分。
    static {
        registerService(ACCESSIBILITY_SERVICE, new ServiceFetcher() {
                public Object getService(ContextImpl ctx) {
                    return AccessibilityManager.getInstance(ctx);
                }});
                
        registerService(CAPTIONING_SERVICE, new ServiceFetcher() {
                public Object getService(ContextImpl ctx) {
                    return new CaptioningManager(ctx);
                }});

        registerService(ACCOUNT_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    IBinder b = ServiceManager.getService(ACCOUNT_SERVICE);
                    IAccountManager service = IAccountManager.Stub.asInterface(b);
                    return new AccountManager(ctx, service);
                }});

        registerService(ACTIVITY_SERVICE, new ServiceFetcher() {
                public Object createService(ContextImpl ctx) {
                    return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
                }});
                
... ...

    }
```

看到这里就能放心在 Activity 中尽情的调用 getSystemService 了，因为不管你调用多少次，一个进程就只会有一个服务的 Bp 实例而已。

## native 层
前面说了 aidl 是 java 层才有的东西。那么 native 是不是就要手动写那一堆麻烦的 binder 接口实现啦。基本上是，但是有点改善，因为 C++ 有宏和模板函数这2个东西。回去看下原理篇中说的 native 层 binder 库中 IInterface.h 中的那几个宏：

<pre config="brush:bash;toolbar:false;">
DECLARE_META_INTERFACE
IMPLEMENT_META_INTERFACE
CHECK_INTERFACE
</pre>

以及衍生出来的 BpInterface 和 BnInterface 这2个模板类。至少提供了一些模板，让子类省了点事。这几个东西回去看原理篇吧，这里不重复说了。

native 层的 SS 本身就不多，4.4 目前就下面几个吧：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-aidl/1.png)

然后，普通应用直接不开放 native 层服务的接口。所以估计 android 就拿宏和模板函数凑活下了。

## 总结
aidl 这个东西还是不错的，原来要写一堆无聊重复的代码，现在只要列一个 aidl 的清单文件就行了。我又要啰嗦一句：android 干了很多事。


