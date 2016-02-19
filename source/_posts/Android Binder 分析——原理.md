title: Android Binder 分析——原理
date: 2015-01-28 20:17:16
updated: 2016-03-31 10:31:16
categories: [Android Framework]
tags: [android]
---

分析之前说一下原理。为要 android 要搞这么复杂的一个东西。那是因为 android 是个多进程的系统，进程间的数据交换、相互调用（某几个程序配合完成某些业务）就涉及跨进程通信。2个进程不能直接访问数据的原因：

1. 每个进程的地址空间的独立的，所以进程A中某个数据的地址在进程B中不确定是什么东西。
2. 安全性，如果能随便访问其它进程空间的数据，那么是非常危险的事情（想想看你再用支付宝输支付密码的时候，其它随便一个程序就能轻轻松松读取你输入的密码是多么恐怖）。

所以 android 就整一套进程通信框架——binder

## 原理
首先 binder 在最底层有 kernel 的驱动支持。/dev/binder 是 binder 的设备文件。然后 android 通过这个驱动在 native 层整了一套 C/S 架构的框架出来，最后在 java 对应也封装了一层（可以理解为 native 的马甲）。这些东西后面再慢慢分析。

## 应用
基于 binder android 弄了很多 manager services，不过我觉得倒是因为需要存在这些 maanger services 才需要 binder 进程间通信。这里说说为什么需要这些 manager services（我后面把这些称为：那一票 services）。因为设备的上的有些硬件（例如相机、传感器）一般一次只能一个访问，有些需要把一些数据混合在一起输出（SurfaceFlinger、AudioFlinger），这就需要一些管理，但是应用是不同的程序，它们并不知道其它人的情况，所以就需要一个 manager，而且这个 manager 是要能接受不同进程的。这就引出了 android binder 的最经典的场景—— android 那一票 services。

同时由于有这一票 services 的 存在，那么又要有人来管它们，所以就有一个东西叫： ServiceManager 。这个东西本身也是基于 binder 通信的。

## 框架设计——native
binder 主要的实现在 native 层。先来张图，整体对框架有个概括（图中我省略了 binder 进程 death 之后的通知机制，这个可以后面单独分析，这里只是画出了刚开始比较关键的通信的那部分）：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-base/1.png)

这里先啰嗦下 binder native 层的代码位置（这个是个模块，相关的都里面，没几个文件，自己找吧）： 

```bash
# 头文件：
frameworks/native/include/binder
# 实现：
frameworks/native/libs/binder
```

图中分为2个部分：一个是实现部分（BBinder、Bpinder 那部分），一个是接口部分（IInterface 那部分）。先来看下实体部分：

前面说了 binder 是 C/S 架构的，那当然得有 server 和 client。BBinder 代表 server，Bpinder 代表 client，然后在这个基础上抽象出 IBinder 这个基类。IBinder 中比较重要的抽象方法有4个：

```cpp
    /**
     * Check if this IBinder implements the interface named by
     * @a descriptor.  If it does, the base pointer to it is returned,
     * which you can safely static_cast<> to the concrete C++ interface.
     */ 
    virtual sp<IInterface>  queryLocalInterface(const String16& descriptor);

    virtual status_t        transact(   uint32_t code,
                                        const Parcel& data,
                                        Parcel* reply,
                                        uint32_t flags = 0) = 0;

    virtual BBinder*        localBinder();
    virtual BpBinder*       remoteBinder();
```

然后在基类有默认实现的是3个：

```cpp
sp<IInterface>  IBinder::queryLocalInterface(const String16& descriptor)
{
    return NULL;
}

BBinder* IBinder::localBinder()
{
    return NULL;
}

BpBinder* IBinder::remoteBinder()
{
    return NULL;
}
```

基类的实现还真全是马甲，但是这样也有个好出，就是之类可以只覆盖自己感兴趣的方法就可以了（否则子类就必须全部实现，不然编译会报错的）。

先来看看 localBinder 和 remoteBinder 这2个，这2个看定义就十分明显了，一个是返回 BBinder 的指针（服务器的），一个是返回 BpBinder 的指针（客户端的），而且在 BBinder 和 BpBinder 分别只实现了一个：

```cpp
# 简单干脆，都直接返回自己
BBinder* BBinder::localBinder()
{
    return this;
}

BpBinder* BpBinder::remoteBinder()
{
    return this;
}
```

然后是 queryLocalInterface 这个，这个返回的是 IInterface 的指针（sp android 搞的啥智能指针，可以参看我前面一篇相关的备忘，挺烦的）。BBinder 和 BpBinder 都没有实现，这个放到后面暴露的接口去实现了（后面再说）。

最后来看下： transact，这个看名字，和参数就知道这个就是通信用的方法。这个在基类中也没实现，但是在 BBinder 和 BpBinder 有实现，并且不一样（当然得不一样，服务器能和客户端一样么）。BBinder 的：

```cpp
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) 
{
    data.setDataPosition(0);
    
    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            reply->writeInt32(pingBinder());
            break;
        default:
            err = onTransact(code, data, reply, flags);
            break;
    }
   
    if (reply != NULL) {
        reply->setDataPosition(0);
    }
    
    return err; 
}
```

那个 `PING_TRANSACTION` 估计是测试用的，先不理它，它主要就是调用了 onTransact 这个回调。这个回调是在 BBinder 中定义的，是个虚函数，主要留给它的子类来实现。可以想象得到服务器端在等待客户端的请求，当有请求来的时候，就会出发 onTransact 然后由具体的服务（子类）来实现这个回调，处理不同的逻辑。 

而在 BpBinder 中是这样的：

```cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
   
    return DEAD_OBJECT;
}
```

又是马甲，这个IPCThreadState 是线程剧本变量，就是一个线程存一个，不同线程不一样，binder 的 C/S 架构采用了多线程来处理请求，这个也是后面再分析，先来看看实现再说：

```cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;

    IF_LOG_TRANSACTIONS() {
        TextOutput::Bundle _b(alog);
        alog << "BC_TRANSACTION thr " << (void*)pthread_self() << " / hand "
            << handle << " / code " << TypeCode(code) << ": " 
            << indent << data << dedent << endl;
    }
    
    if (err == NO_ERROR) {
        LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
            (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    
    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }
    
    if ((flags & TF_ONE_WAY) == 0) { 
        #if 0
        if (code == 4) { // relayout
            ALOGI(">>>>>> CALLING transaction 4");
        } else {
            ALOGI(">>>>>> CALLING transaction %d", code);
        }
        #endif
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }    
        #if 0
        if (code == 4) { // relayout
            ALOGI("<<<<<< RETURNING transaction 4");
        } else {
            ALOGI("<<<<<< RETURNING transaction %d", code);
        }
        #endif

        IF_LOG_TRANSACTIONS() {
            TextOutput::Bundle _b(alog);
            alog << "BR_REPLY thr " << (void*)pthread_self() << " / hand "
                << handle << ": ";
            if (reply) alog << indent << *reply << dedent << endl;
            else alog << "(none requested)" << endl;
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }

    return err;
}
```

这个函数中关键点是 writeTransactionData 和  waitForResponse，这2个函数分别是对 binder 驱动写请求（数据已经通过 Parcel 打包好，这个也是后面再分析），然后等待 binder 驱动的返回的数据结果（服务器那端写的）。驱动相关的也是后面再说，先在继续往下走。这里可以看得出客户端是将请求写入驱动，发送给服务器，然后等待服务器返回的结果。

然后就是接口了，接口基类是 IInterface，这个基类很简单，就定义2个有用的函数：

```cpp
class IInterface : public virtual RefBase
{
public:
            IInterface();
            sp<IBinder>         asBinder();
            sp<const IBinder>   asBinder() const;
            
protected:
    virtual                     ~IInterface();
    virtual IBinder*            onAsBinder() = 0;
};

sp<IBinder> IInterface::asBinder()
{
    return this ? onAsBinder() : NULL;
}

sp<const IBinder> IInterface::asBinder() const
{
    return this ? const_cast<IInterface*>(this)->onAsBinder() : NULL;
}
```

其实它留给子类的就 onAsBinder 这个回调，用来获取 IBinder 的指针。IInterface 的子类是 BnInterface<INTERFACE> 和 BpInterface<INTERFACE> 分别对应 BBinder（服务器） 和 BpBinder（客户端）的接口。在这之前得先看看 INTERFACE 这个东西，android 在这里弄了一个模版类，BnXx 和 BpXx 都是。

在 IInterface.h 中有这2个宏：

```cpp
// ----------------------------------------------------------------------

#define DECLARE_META_INTERFACE(INTERFACE)                               \
    static const android::String16 descriptor;                          \
    static android::sp<I##INTERFACE> asInterface(                       \
            const android::sp<android::IBinder>& obj);                  \
    virtual const android::String16& getInterfaceDescriptor() const;    \
    I##INTERFACE();                                                     \
    virtual ~I##INTERFACE();                                            \


#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    const android::String16 I##INTERFACE::descriptor(NAME);             \
    const android::String16&                                            \
            I##INTERFACE::getInterfaceDescriptor() const {              \
        return I##INTERFACE::descriptor;                                \
    }                                                                   \
    android::sp<I##INTERFACE> I##INTERFACE::asInterface(                \
            const android::sp<android::IBinder>& obj)                   \
    {                                                                   \
        android::sp<I##INTERFACE> intr;                                 \
        if (obj != NULL) {                                              \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == NULL) {                                         \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    I##INTERFACE::I##INTERFACE() { }                                    \
    I##INTERFACE::~I##INTERFACE() { }                                   \


#define CHECK_INTERFACE(interface, data, reply)                         \
    if (!data.checkInterface(this)) { return PERMISSION_DENIED; }       \


// ----------------------------------------------------------------------
```

如果在 .h 中的 class 定义中调用 `DECLARE_META_INTERFACE("Xx")` 在 .cpp 中的实现中调用 `IMPLEMENT_META_INTERFACE("Xx", "Xx")` 那么就想当于声明和实现了：

1. 默认构造函数
2. 析构函数
3. 定义了并以 Xx 初始化 String16 descriptor 这个变量
4. asInterface： 返回 IXx 的指针
5. getInterfaceDescriptor： 返回 descriptor 字符串

实际上不是如果，后面的具体的 native 的 service 的接口就是这么写的（后面会有具体的实例分析）。

接下来就看真正的接口基类： BnInterface<Xx>:

```cpp
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
{
public:
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
    virtual const String16&     getInterfaceDescriptor() const;

protected:
    virtual IBinder*            onAsBinder();
};
```

哎呦咧，C++ 的多重继承，分别继续了 BBinder（binder 的服务器） 和 INTERFACE 这个其实就是上面的 IInterface，在具体的 services 中会通过 IInterface 中的那2个宏弄一个 IXx 出来，然后 BnInterface 这里的 INTERFACE 就是 IXx（这个后面到了实例分析，就会很清楚了）。这里实现上面 BBinder 那没实现的几个接口：

```cpp
template<typename INTERFACE>
inline sp<IInterface> BnInterface<INTERFACE>::queryLocalInterface(
        const String16& _descriptor)   
{
    // 对比下是不是自己的标示，是的话直接返回自己
    if (_descriptor == INTERFACE::descriptor) return this;
    return NULL;
}

template<typename INTERFACE>
inline const String16& BnInterface<INTERFACE>::getInterfaceDescriptor() const
{
    // 这 ... 有必要重写一下么？？
    return INTERFACE::getInterfaceDescriptor();
}

template<typename INTERFACE>
IBinder* BnInterface<INTERFACE>::onAsBinder()
{
    // 这里抛弃 BBinder 的 localBinder 了，直接返回自己了
    return this;
}
```

queryLocalInterface 这个是只有 BBinder 才有的（对比 IBinder 的接口），然后根据在这个函数的实现：对比 descriptor 是不是自己定义的（通过 `DECLARE_META_INTERFACE` 这个宏定义的），然后是否返回自己，可以猜得到：1、这个函数是用来判断请求是不是处于本进程内，如果是的话，应该就不需要跨进程调用，直接可以调用本进程的方法。这样对于上层应用来说，暴露的是 IBinder 接口，上层应用不需要关心调用是本地的还是远程的。2：descriptor 这个是用来区别 binder 的服务器的，binder 通过这个来判断请求是不是发给自己的，如果 descriptor 不匹配，则拒绝处理。（这些猜测后面再慢慢说）

然后是 BpInterface 了：

```cpp
template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
                                BpInterface(const sp<IBinder>& remote);

protected:
    virtual IBinder*            onAsBinder();
};
```

比 Bn 相比，没了判断是不是本地的接口了，客户端当然不需要有啦。然后应该是继续 BpBinder 的，变成了 BpRefBase ，看到这个名字我又想到了 android 那蛋疼的智能指针和引用计数，我真的很烦这个东西，就不能好好的自己管好内存么？？

```cpp
class BpRefBase : public virtual RefBase
{
protected:
                            BpRefBase(const sp<IBinder>& o); 
    virtual                 ~BpRefBase();
    virtual void            onFirstRef();
    virtual void            onLastStrongRef(const void* id);
    virtual bool            onIncStrongAttempted(uint32_t flags, const void* id);

    // 最重要的一个函数，返回 mRemote IBinder 指针
    inline  IBinder*        remote()                { return mRemote; }
    inline  IBinder*        remote() const          { return mRemote; }

private:
                            BpRefBase(const BpRefBase& o); 
    BpRefBase&              operator=(const BpRefBase& o); 

    IBinder* const          mRemote;
    RefBase::weakref_type*  mRefs;
    volatile int32_t        mState;
};
```

上面这个东西，其它的不看，就看一个函数 remote() 返回 mRemote 这个 IBinder 指针。然后来看下 BpInterface 的构造函数：

```cpp
template<typename INTERFACE>
inline BpInterface<INTERFACE>::BpInterface(const sp<IBinder>& remote)
    : BpRefBase(remote)
{
}

template<typename INTERFACE>
inline IBinder* BpInterface<INTERFACE>::onAsBinder()
{
    // 调用 BpRefBase 的 remote() 返回 BpBinder 的远程 binder 指针
    return remote();
}
```

BpInterface 的 sp<IBinder> 参数的构造函数，把 sp<IBinder> 传给 BpRefBase 了，这个 IBinder 其实就是 BpBinder （通过后面的分析可以看得出的）。结合上面 Bn 的 onAsBinder 是返回 this 自己（BBinder），Bp 的是返回 BpBinder 。

再下面，就是具体业务相关的了，IXx 继承自 IInterface，主要是使用 IInterface 提供的那个 `DECLARE_META_INTERFACE(Xx)` 这个宏声明一些接口（上面有分析的），Xx 就是接口的名字了，例如 ServerManager、SurfaceComposer 之类的（这个后面会有具体的实例分析的）。然后这个 IXx 还得定义这个 service 对外（客户端）提供的接口，例如 CaptureScreen 之类的，这个就和具体的 service 相关了。 

BnXx 继承 BnInterface（同时继承 IXx），使用 IInterface 提供的 `IMPLEMENT_META_INTERFACE(Xx, Xx)` 来实现上面说的那些接口。注意这里的 Xx 要和 `DECLARE_META_INTERFACE` 那里的一样，例如都叫 SurfaceComposer， 然后后面那个是标示（descriptor），例如 "android.ui.ISurfaceComposer"（用包名来标示一般不会重复）。

然后剩下的主要是实现 onTransact 就是响应客户端的请求。其实通过后面的分析 BnXx 中也并不是真正实现请求的地方，这个只是一个中转站而已，真正的实现在 service 模块里面，一个一个业务函数实现的，这里的 onTransact 只是区分客户端发过来的请求命令，然后去调用 service 里面的函数，实现这些的文件都叫 I
Xx.h, IXx.cpp 看上去也不像实现的样子。然后真正的 service 就要继承 BnXx 去实现 onTransact。但是你也发现了，onTransact 已经在 BnXx 里实现了，所以你在 services 看到的 onTransact 都是马甲（有些做了一些拦截，例如权限检测，没权限的请求直接拦截下来），基本上都是调用： super.onTransact 的 -_-|| 。

BpXx 继承 BpInterface。 BpInterface 并没强制要求 BpXx 实现啥东西，但是作用客户端（Java 层那一堆 XxManager 给其它 apk 调用的），暴露给第三方引用使用的，必须要实现服务器提供的方法对应的接口：例如说 service 那边有一个方法是： captureScreen 用来实现截屏用的，那么客户端也必须有响应的方法： captureScreen，然后客户端调用 remote()（IBinder） 的 transact 发送请求到服务器。当然其实函数名字也可以不一一对应，你只要在服务器 onTransact 里调用正确就行了。但是后面你会发现如果这些东西一一对应的话，代码是很机械的，后面 android 就搞了个代码自动生成的工具出来（aidl）。
 
这里 native 层的框架就说完了。接下来看下 java 层的。

## 框架设计——java
照例来先张图先：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-base/2.png)

图中我把相关的 jni 里面类也画了出来，这样就能更加明显的看到 java 层和 native 层的对应关系。

代码位置也先说下：

```bash
# java：
frameworks/base/core/java/android/os/Binder.java
frameworks/base/core/java/android/os/Parcel.java
frameworks/base/core/java/com/android/internal/os/BinderInternal.java

# jni：
frameworks/base/core/jni/android_util_Binder.h
frameworks/base/core/jni/android_util_Binder.cpp
frameworks/base/core/jni/android_os_Parcel.h
frameworks/base/core/jni/android_os_Parcel.cpp
```

从图可得到，java 层的架构和 native 的很像。也是分为2个部分：实现部分和接口部分（几乎连名字都和 native 的一样）。

实现部分的话，也是一个 IBinder 的基类，额，不对 java 的叫 interface ，这个比 c++ 更加注重抽象。然后实现这个接口的分别是 Binder（Binder.java） 和 BinderProxy（Binder.java），分别对应 native 的 BBinder 和 BpBinder ，同时也分别代表 java 层的服务端和客户端。

Binder 这边的话：

```java
    /*
     * Convenience method for associating a specific interface with the Binder.
     * After calling, queryLocalInterface() will be implemented for you
     * to return the given owner IInterface when the corresponding
     * descriptor is requested.
     */
    public void attachInterface(IInterface owner, String descriptor) {
        mOwner = owner;
        mDescriptor = descriptor;      
    }
       
    /*
     * Default implementation returns an empty interface name.
     */
    public String getInterfaceDescriptor() {
        return mDescriptor;   
    }

    /*
     * Use information supplied to attachInterface() to return the
     * associated IInterface if it matches the requested
     * descriptor.
     */
    public IInterface queryLocalInterface(String descriptor) {
        if (mDescriptor.equals(descriptor)) {
            return mOwner;
        }
        return null;
    }

    /*
     * Default implementation is a stub that returns false.  You will want
     * to override this to do the appropriate unmarshalling of transactions.
     * 
     * <p>If you want to call this, call transact().
     */
    protected boolean onTransact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        if (code == INTERFACE_TRANSACTION) {
            reply.writeString(getInterfaceDescriptor());
            return true;
        } else if (code == DUMP_TRANSACTION) {
            ParcelFileDescriptor fd = data.readFileDescriptor();
            String[] args = data.readStringArray(); 
            if (fd != null) {
                try {
                    dump(fd.getFileDescriptor(), args);
                } finally {   
                    try {
                        fd.close();                    
                    } catch (IOException e) {      
                        // swallowed, not propagated back to the caller
                    }
                }
            }
            // Write the StrictMode header.
            if (reply != null) {           
                reply.writeNoException();      
            } else {
                StrictMode.clearGatheredViolations();
            }
            return true;
        }
        return false;
    }

    /*
     * Default implementation rewinds the parcels and calls onTransact.  On
     * the remote side, transact calls into the binder to do the IPC.
     */
    public final boolean transact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        if (false) Log.v("Binder", "Transact: " + code + " to " + this);
        if (data != null) {
            data.setDataPosition(0);       
        }
        boolean r = onTransact(code, data, reply, flags);
        if (reply != null) {  
            reply.setDataPosition(0);      
        }
        return r;
    }
```

前面几个函数都挺简单的，就是返回 descriptor ，要不返回 IInterface，要不设置下 IInterface 和 desriptor 。transact 这个函数看注释说是啥默认实现，别看它调用了 onTransact （native Bn 端的实现主要在这里）其实就是个摆设，没用的，因为要实现 IBinder 的接口，才必须要有一个实现而已（到后面就就知道了）。然后 onTransact 这个和 native 对应了，不过这里其实也没干声明正事，功能主要是留在service 的接口里实现的。

java 的 Binder 最主要要看的是它有个叫 mObject 的变量：

```java
    /* mObject is used by native code, do not remove or rename */
    private int mObject;
```

看注释说是 native 层要使用，其实是 jni 要使用。来看看这个 mObject 是怎么使用的：

```cpp
    private native final void init();

    /**
     * Default constructor initializes the object.
     */
    public Binder() {
        init();

        if (FIND_POTENTIAL_LEAKS) {    
            final Class<? extends Binder> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Binder class should be static or leaks might occur: " +
                    klass.getCanonicalName());     
            }
        }
    } 
```

在 Binder 的构造函数中会调用一个 init 的 native 方法，这个方法在 jni `android_util_Binder.cpp` 里面：

```cpp
static void android_os_Binder_init(JNIEnv* env, jobject obj) 
{
    JavaBBinderHolder* jbh = new JavaBBinderHolder();
    if (jbh == NULL) {
        jniThrowException(env, "java/lang/OutOfMemoryError", NULL);
        return;
    }
    ALOGV("Java Binder %p: acquiring first ref on holder %p", obj, jbh);
    jbh->incStrong((void*)android_os_Binder_init);
    env->SetIntField(obj, gBinderOffsets.mObject, (int)jbh);
}
```

这里创建了一个 JavaBBinderHolder （这个是 c++ 的类）的类，然后把这个指针保存在了 mObject 这个变量里面。int 类型是 32bit 的，指针（地址）在 32bit 系统上也是 32bit 的。这里再先看看 gBinderOffsets 这个东西，其实还有一个叫 gBinderProxyOffsets 的。这2个是 jni 里面 java 的类信息变量，弄成全局变量，如果访问频繁的话，能提高速度，因为每次都要去查 java 的类表很慢的，加载的代码在：

```cpp
const char* const kBinderPathName = "android/os/Binder";

static int int_register_android_os_Binder(JNIEnv* env)
{
    jclass clazz;

    clazz = env->FindClass(kBinderPathName);
    LOG_FATAL_IF(clazz == NULL, "Unable to find class android.os.Binder");

    gBinderOffsets.mClass = (jclass) env->NewGlobalRef(clazz);
    gBinderOffsets.mExecTransact
        = env->GetMethodID(clazz, "execTransact", "(IIII)Z");
    assert(gBinderOffsets.mExecTransact);

    gBinderOffsets.mObject
        = env->GetFieldID(clazz, "mObject", "I");
    assert(gBinderOffsets.mObject);

    return AndroidRuntime::registerNativeMethods(
        env, kBinderPathName,
        gBinderMethods, NELEM(gBinderMethods));
}

const char* const kBinderProxyPathName = "android/os/BinderProxy";

static int int_register_android_os_BinderProxy(JNIEnv* env)
{
    jclass clazz;

    clazz = env->FindClass("java/lang/ref/WeakReference");
    LOG_FATAL_IF(clazz == NULL, "Unable to find class java.lang.ref.WeakReference");
    gWeakReferenceOffsets.mClass = (jclass) env->NewGlobalRef(clazz);
    gWeakReferenceOffsets.mGet
        = env->GetMethodID(clazz, "get", "()Ljava/lang/Object;");
    assert(gWeakReferenceOffsets.mGet);

    clazz = env->FindClass("java/lang/Error");
    LOG_FATAL_IF(clazz == NULL, "Unable to find class java.lang.Error");
    gErrorOffsets.mClass = (jclass) env->NewGlobalRef(clazz);

    clazz = env->FindClass(kBinderProxyPathName);
    LOG_FATAL_IF(clazz == NULL, "Unable to find class android.os.BinderProxy");

    gBinderProxyOffsets.mClass = (jclass) env->NewGlobalRef(clazz);
    gBinderProxyOffsets.mConstructor
        = env->GetMethodID(clazz, "<init>", "()V");
    assert(gBinderProxyOffsets.mConstructor);
    gBinderProxyOffsets.mSendDeathNotice
        = env->GetStaticMethodID(clazz, "sendDeathNotice", "(Landroid/os/IBinder$DeathRecipient;)V");
    assert(gBinderProxyOffsets.mSendDeathNotice);

    gBinderProxyOffsets.mObject
        = env->GetFieldID(clazz, "mObject", "I");
    assert(gBinderProxyOffsets.mObject);
    gBinderProxyOffsets.mSelf
        = env->GetFieldID(clazz, "mSelf", "Ljava/lang/ref/WeakReference;");
    assert(gBinderProxyOffsets.mSelf);
    gBinderProxyOffsets.mOrgue
        = env->GetFieldID(clazz, "mOrgue", "I");
    assert(gBinderProxyOffsets.mOrgue);

    clazz = env->FindClass("java/lang/Class");
    LOG_FATAL_IF(clazz == NULL, "Unable to find java.lang.Class");
    gClassOffsets.mGetName = env->GetMethodID(clazz, "getName", "()Ljava/lang/String;");
    assert(gClassOffsets.mGetName);

    return AndroidRuntime::registerNativeMethods(
        env, kBinderProxyPathName,
        gBinderProxyMethods, NELEM(gBinderProxyMethods));
}

int register_android_os_Binder(JNIEnv* env)
{
    if (int_register_android_os_Binder(env) < 0)
        return -1;
    if (int_register_android_os_BinderInternal(env) < 0)
        return -1;
    if (int_register_android_os_BinderProxy(env) < 0)
        return -1;

    ... ...
}
```

这个 `register_android_os_Binder` 会在 android 的 java 初始化的时候调用，这个时候需要的类信息就加载好了。gBinderOffsets 对应的是 Binder， gBinderProxyOffsets 对应的是 BinderProxy 。gBinderOffsets 的 mObject 就是 Binder 的 mObject 变量，所以那个注释说不要乱改这个变量的名字，不然这里就找不到了。gBinderOffsets 还有一个保存了 execTransact 的变量，这个注意一下，后面会说到的。

回过来看下 JavaBBinderHolder 这个东西：

```cpp
class JavaBBinderHolder : public RefBase
{
public:
    sp<JavaBBinder> get(JNIEnv* env, jobject obj)
    {
        AutoMutex _l(mLock);  
        sp<JavaBBinder> b = mBinder.promote();
        if (b == NULL) {
            b = new JavaBBinder(env, obj); 
            mBinder = b;      
            ALOGV("Creating JavaBinder %p (refs %p) for Object %p, weakCount=%d\n",
                 b.get(), b->getWeakRefs(), obj, b->getWeakRefs()->getWeakCount());
        }

        return b;
    }

    sp<JavaBBinder> getExisting()
    {
        AutoMutex _l(mLock);  
        return mBinder.promote();      
    }

private:
    Mutex           mLock;
    wp<JavaBBinder> mBinder;
};
```

这个东西继承自 native 的 RefBase，然后在看它的 get 方法的返回值，可以理解为这个东西就是在 java 里面放了一个 natvie 的 sp<IBinder> 一样。它持有的是 JavaBBinder 对象，这个才是重点：

```java
class JavaBBinder : public BBinder
{
public:
    JavaBBinder(JNIEnv* env, jobject object)
        : mVM(jnienv_to_javavm(env)), mObject(env->NewGlobalRef(object))
    {
        ALOGV("Creating JavaBBinder %p\n", this);
        android_atomic_inc(&gNumLocalRefs);
        incRefsCreated(env);
    }

    bool    checkSubclass(const void* subclassID) const
    {
        return subclassID == &gBinderOffsets;
    }

    jobject object() const
    {
        return mObject;
    }

protected:
    virtual ~JavaBBinder()
    {
        ALOGV("Destroying JavaBBinder %p\n", this);
        android_atomic_dec(&gNumLocalRefs);
        JNIEnv* env = javavm_to_jnienv(mVM);
        env->DeleteGlobalRef(mObject);
    }

    virtual status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
    {
        JNIEnv* env = javavm_to_jnienv(mVM);

        ALOGV("onTransact() on %p calling object %p in env %p vm %p\n", this, mObject, env, mVM);

        IPCThreadState* thread_state = IPCThreadState::self();
        const int strict_policy_before = thread_state->getStrictModePolicy();
        thread_state->setLastTransactionBinderFlags(flags);

        //printf("Transact from %p to Java code sending: ", this);
        //data.print();
        //printf("\n");
        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, (int32_t)&data, (int32_t)reply, flags);
        jthrowable excep = env->ExceptionOccurred();

        if (excep) {
            report_exception(env, excep,
                "*** Uncaught remote exception!  "
                "(Exceptions are not yet supported across processes.)");
            res = JNI_FALSE;

            /* clean up JNI local ref -- we don't return to Java code */
            env->DeleteLocalRef(excep);
        }

        // Restore the Java binder thread's state if it changed while
        // processing a call (as it would if the Parcel's header had a
        // new policy mask and Parcel.enforceInterface() changed
        // it...)
        const int strict_policy_after = thread_state->getStrictModePolicy();
        if (strict_policy_after != strict_policy_before) {
            // Our thread-local...
            thread_state->setStrictModePolicy(strict_policy_before);
            // And the Java-level thread-local...
            set_dalvik_blockguard_policy(env, strict_policy_before);
        }

        jthrowable excep2 = env->ExceptionOccurred();
        if (excep2) {
            report_exception(env, excep2,
                "*** Uncaught exception in onBinderStrictModePolicyChange");
            /* clean up JNI local ref -- we don't return to Java code */
            env->DeleteLocalRef(excep2);
        }

        // Need to always call through the native implementation of
        // SYSPROPS_TRANSACTION.
        if (code == SYSPROPS_TRANSACTION) {
            BBinder::onTransact(code, data, reply, flags);
        }

        //aout << "onTransact to Java code; result=" << res << endl
        //    << "Transact from " << this << " to Java code returning "
        //    << reply << ": " << *reply << endl;
        return res != JNI_FALSE ? NO_ERROR : UNKNOWN_TRANSACTION;
    }

    virtual status_t dump(int fd, const Vector<String16>& args)
    {
        return 0;
    }

private:
    JavaVM* const   mVM;
    jobject const   mObject;
};
```

JavaBBinder 继承自 native 的 BBinder，这下清楚了，java 层最重要还是持有了 native 的 binder 对象。同时这个东西还保存了 java Binder 的对象（相互保存 -_-||），那个 jobject mObject 就是，构造函数传进来的，然后在 JavaBBinderHolder 的 get 那里第一次会触发 new JavaBBinder，参数就是 java 的 Binder，这里后面再说。然后这个类重载了 BBinder 的 onTransact ，前面 native 说过了， Bn 端的实现主要是要重写 onTransact。然后在 onTransact 通过 java 对象 mObject 调用了 java Binder 的 gBinderOffsets 的 mExecTransact 。还记得前面加载 Binder 类信息的时候，说要注意这个 mExecTransact 的么，就是这里用啦，对应 Binder 的 execTransact 函数：

```java
    // Entry point from android_util_Binder.cpp's onTransact
    private boolean execTransact(int code, int dataObj, int replyObj,
            int flags) {
        Parcel data = Parcel.obtain(dataObj); 
        Parcel reply = Parcel.obtain(replyObj);
        // theoretically, we should call transact, which will call onTransact,
        // but all that does is rewind it, and we just got these from an IPC,
        // so we'll just call it directly.
        boolean res;
        try {
            res = onTransact(code, data, reply, flags);
        } catch (RemoteException e) {  
            reply.setDataPosition(0);      
            reply.writeException(e);       
            res = true;
        } catch (RuntimeException e) { 
            reply.setDataPosition(0);      
            reply.writeException(e);       
            res = true;
        } catch (OutOfMemoryError e) { 
            RuntimeException re = new RuntimeException("Out of memory", e);
            reply.setDataPosition(0);      
            reply.writeException(re);      
            res = true;
        }
        reply.recycle();
        data.recycle();
        return res;
    }
```

最后是调用了 Binder 的 onTransact 函数，所以还是和 native 的一样，Bn 靠重载 onTransact。所以前面说 Binder 实现 IBinder 的 transact 接口是摆设，因为这个不像 native 层 BBinder 的 transact，java 层的压根就没调用到。总结下，java 的 Bn 是通过 native 的 BBinder 的 transact 被调用，然后 natvie BBinder 的 onTransact（JavaBBinder 重载） 的被调用，然后在 jni 中调用 java 的 Binder 的 onTransact 。

然后是 BinderProxy:

```java
final class BinderProxy implements IBinder {

    public native String getInterfaceDescriptor() throws RemoteException;
    public native boolean transact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException; 

    public IInterface queryLocalInterface(String descriptor) {
        return null;
    }

}
```

BinderProxy 很多都直接是 native 方法。queryLocalInterface 直接放回 null，和 native 层的一样， Bp 端不实现 Bn 端的方法。它和 Binder 一样有一个 int mObject 的变量，同样这个也是保存 native 对象指针的，这个其实是 sp<BpBinder>，这个后面再分析了。所以 java 的 BinderProxy transact 方法就直接调用 native BpBinder 的 transact 。

接下来就是接口部分了。IInterface :

```java
/*
 * Base class for Binder interfaces.  When defining a new interface,
 * you must derive it from IInterface.
 */
public interface IInterface
{
    /*
     * Retrieve the Binder object associated with this interface.
     * You must use this instead of a plain cast, so that proxy objects
     * can return the correct result.
     */
    public IBinder asBinder();
}
```

和 native 层的很像，但是这个和 Binder 和 BinderProxy 不一样，java 的 IInterface 和 native 层的没啥关系（Binder 和 BinderProxy 可是保存了 native 层对象的指针的），只是为单纯为了对应而已。

然后同样，java 层的 service 需要继续这个接口，自己写一个 IXxManager 定义自己的服务提供的业务逻辑接口（那个 Manager 的后缀不是必须的，但是 android 系统的 service 都很统一，接口都叫 XxManager，服务都叫 XxManagerServices）。然后 Bn 这边的话，实现接口一般都叫 XxManagerNative 去实现这个 IXxManager 的接口，相当于 native 的 BnXx 实现 BnInterface 一样。然后服务模块继承这个 XxManagerNative，真正实现业务逻辑功能，XxManagerNative 里面的 onTransact 负责接受客户端发送过来的请求，并且调用正确的 service 的业务函数完成功能（和 native 流程一样）。

Bp 这边呢，一般由一个叫 XxProxy 的类实现 IXxManager 的接口，然后它有一个 mRemote 的 IBinder 变量，其实就是 BinderProxy （这个后面实例慢慢分析）。然后接口是通过 Parcel 把请求打包好，通过的 mRemote（BinderProxy，BinderProxy 直接调用 native BpBinder 的 transact） 的 transact 发送给 binder 驱动，最后再转给 Bn 端接收。流程也和 native 的是一样的。最后暴露给应用使用的 XxManager 其实一般都保存了一个 XxProxy 对象，然后 XxManager 的接口，基本上都是马甲，直接调用 XxProxy 对应的方法的（同样后面对照实例慢慢分析）。

前面 uml 中我在 XxManagerNative 和 XxProxy 中还有个括号，我前面说了 android 在 java 搞了个代码自动生成的东西——aidl，那个括号里面的类名是工具生成出来的，括号外面是手动写的。android 那一票 services 绝大部分接口的代码是用工具生成的，但是有几个是手写的，原因么，估计刚开始还没这个工具吧。 


android 故意在 java 层上 binder 的框架结构和 native 层保持一致，这是个不错的设计，然后 java 层 binder 通信其实就是 native 的调用而已。上面简单把 binder 的框架梳理了一下，有很多地方后面再慢慢分析，因为 binder 涉及的东西太多了（横跨 java、native 和 kernel），而且进程间通信本来就是比较麻烦的东西，我认为多进程这个是现代智能操作系统的必不可少的基本功能之一。

把基本的东西先弄清楚，后面分析会方便很多。


