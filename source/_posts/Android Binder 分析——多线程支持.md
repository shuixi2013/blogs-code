title: Android Binder 分析——多线程支持
date: 2015-01-28 21:42:16
updated: 2015-03-31 14:29:16
categories: [Android Framework]
tags: [android]
---

前面普通服务篇那里说到 ActivityManager（AM） 里锁的问题，其实不光 AM，WindowManager（WM）、PackageMananger（PM）中基本上很多对外的业务函数里面都是加锁的，所以这些 SS 里面有会有带 Locked 结尾的函数（这些函数都是在锁里执行）。这里就提出一个疑问为什么要加锁。这篇就来解答这个问题，顺带扯出 binder 的多线程支持的问题。

照例先把相关源码位置啰嗦一下（4.4）：

```bash
# java 层 SS 相关代码
frameworks/base/services/java/com/android/server/SystemServer.java

# java 层 zygote 相关接口
frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
libcore/dalvik/src/main/java/dalvik/system/Zygote.java

# jni runtime 接口
frameworks/base/core/jni/AndroidRuntime.cpp

# native app process 程序
frameworks/base/cmds/app_process/app_main.cpp

# native binder 库
frameworks/native/libs/binder/ProcessState.cpp
frameworks/native/libs/binder/IPCThreadState.cpp
frameworks/native/libs/binder/Static.cpp

# native SS 主程序
frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp
frameworks/native/services/sensorservice/main_sensorservice.cpp

# libutils 库（线程支持）
system/core/include/utils/AndroidThreads.h
system/core/include/utils/Thread.h
system/core/libutils/Threads.cpp

# kernel binder 驱动
kernel/drivers/staging/android/binder.h
kernel/drivers/staging/android/binder.c
```

## SS 多线程的例子
前面说了 binder 是 C/S 模型。既然是 C/S 模型，很容易让人想到会服务器同时可能会被多个客户端连接、请求。传统的服务器，都会开一个线程池来应对这种情况，避免请求数一多，某些客户端长时间没响应的情况。

这个在 binder 中是一样的，只不过 binder 是本地的而已（这里说的本地是相对网络上的服务器来说的）。你想想你写 android 代码的时候，经常就 get AM ，然后 AM xx 的调用了。所以说 binder 中的 Bn 端经常会同一时候有多个 Bp 请求的，特别是 SS（谁让它们是公用的）。既然这样的话，我们在写服务的时候是不是也要像传统的服务器一样考虑使用线程池来提高服务的并发响应能力咧。先别急，android 早就帮我们整好一个现成的框架了。

我们先来看看一些实际的列子。我们以 sensorservice 为例（先说 native 的 SS，java 层的要绕好久，后面再说）。sensorservice 的 main 函数：

```cpp
// main_sensorservice.cpp =============================

// 这个 main 函数就一句话 ... ...
int main(int argc, char** argv) {
    SensorService::publishAndJoinThreadPool();
    return 0;
}
```

SensorService 是继承自 BinderService 的类，我们去看看 BinderService：

```cpp
// BinderService.h =============================

// 这是一个模板类
template<typename SERVICE>
class BinderService
{
public:
    static status_t publish(bool allowIsolated = false) {
        // 获取 SM
        sp<IServiceManager> sm(defaultServiceManager());
        // new Service 对象，然后 add 到 SM 中
        return sm->addService(
                String16(SERVICE::getServiceName()),
                new SERVICE(), allowIsolated);
    }
    
    static void publishAndJoinThreadPool(bool allowIsolated = false) {
        publish(allowIsolated);
        joinThreadPool();
    }

    static void instantiate() { publish(); }

    static status_t shutdown() { return NO_ERROR; }

private:
    static void joinThreadPool() {
        // 初始化 ProcessState 对象
        sp<ProcessState> ps(ProcessState::self());
        // 启动线程池
        ps->startThreadPool();
        ps->giveThreadPoolName();
        // 当前线程加入线程池
        IPCThreadState::self()->joinThreadPool();
    }
};
```

这个 BinderService.h 是在 natvie binder 的 include 头文件中，看样子 android 直接提供了一个模板，SS 直接继承，然后在 main 函数调用一句话就行了。模板就是先 new 一个 Service 对象，然后 add 到 SM 中，然后构造一个 ProcessState 对象（这个对象一个进程只有一个），调用 ProcessState->startThreadPool 开线程池（新开了一个线程），最后调用 IPCThreadState->joinThreadPool 将当前进程加入到线程池中（开始阻塞等待了）。

## 进程对象和线程对象
new Service 对象不说了，不同的服务逻辑不一样的。先来看 ProcessState。这个东西前面通信篇介绍过的，一个进程就一个对象，怎么保证的呢，来看它的典型调用方法：

```cpp
// Static.cpp =====================================

// 原来整了一个静态变量噻
Mutex gProcessMutex;
sp<ProcessState> gProcess;

// ProcessState.h =================================

class ProcessState : public virtual RefBase
{
public:
    // 这个也是一个静态函数
    static  sp<ProcessState>    self();

... ...

};

// ProcessState.cpp ================================

sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;        
    }
    gProcess = new ProcessState;
    return gProcess;
}
```

就是搞了一个静态变量，然后一个进程只会创建一次，之后就直接返回这个静态变量就行了，借此来保证一个进程就只有一个 ProcessState 对象，怪不得叫 ProcessState。然后 self() 是静态的，而且无参数，所以进程任何地方任何时候都能够使用。

既然说到了 ProcessState 的 self()，我们顺带也来看下 IPCThreadState 的 self()。看名字这个应该就是一个线程有一个这样对象的了。这个是怎么实现的咧：

```cpp
// IPCThreadState.h ===============================

class IPCThreadState
{
public:
    static  IPCThreadState*     self();

... ...

};

// IPCThreadState.cpp ==============================

static pthread_mutex_t gTLSMutex = PTHREAD_MUTEX_INITIALIZER;
static bool gHaveTLS = false;
static pthread_key_t gTLS = 0; 
static bool gShutdown = false;
static bool gDisableBackgroundScheduling = false;

IPCThreadState* IPCThreadState::self()
{
    // 首先得确保 pthread_key 创建了
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        // 如果之前设置了线程私有变量，取线程私有变量返回
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        // 否则 new 一个新对象
        return new IPCThreadState;
    }

    // 如果线程已经退出了，就直接返回 NULL 了
    if (gShutdown) return NULL;
    
    // pthread_key 没创建的话，先创建 pthread_key
    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) {
        if (pthread_key_create(&gTLS, threadDestructor) != 0) { 
            pthread_mutex_unlock(&gTLSMutex);
            return NULL;
        }    
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    // 回去取线程私有变量
    goto restart;
}
```

IPCThreadState 的 self() 稍微复杂点，因为一个进程有多个线程，要保证一个线程一个对象，这个时候就有利用 linux pthread 的线程私有变量，这个东西可以给每一个 pthread 线程（android 使用 linux 的 pthread 作为多线程的支持）创建一个线程独立的私有变量，这个变量是线程私有的，不用考虑多线程的互斥、锁问题。利用这个特性就很容易显示一个线程一个对象了，把 IPCThreadState 对象保存为 pthread 的线程私有变量就行了，用得的时候就去取线程私有变量，每个线程都是自己的。有了这些知识，上面的代码就不难理解了，就是开始看有没有设置线程私有变量（取之前得先创建 phtread_key，这方面的用法可以去看《UNIX环境高级编程》这本书），有的话直接去线程私有变量就行了，没有的话就 new 一个 IPCThreadState 出来。但是是在哪设置线程私有变量的咧，我们来看看 IPCThreadState 的构造函数：

```cpp
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mMyThreadId(androidGetTid()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0) 
{
    // 就是这里把自己的对象设置为线程私有变量啦
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
```

第一句就是设置。然后我们回去看下 `pthread_key_create` 第二参数 threadDestructor，这是一个函数指针，是说线程退出的话，就调用这个函数，我们看看里面做了什么：

```cpp
void IPCThreadState::threadDestructor(void *st)
{
        IPCThreadState* const self = static_cast<IPCThreadState*>(st);
        if (self) {
                self->flushCommands();
#if defined(HAVE_ANDROID_OS)
        if (self->mProcess->mDriverFD > 0) {
            // 向 binder 发送线程退出的命令
            ioctl(self->mProcess->mDriverFD, BINDER_THREAD_EXIT, 0);
        }   
#endif      
                delete self;
        }
}
```

这个退出函数调用 ioctl 对 binder 驱动发送了一条 `BINDER_THREAD_EXIT` 退出的消息。这个消息后面再说，不过到最后你会发现这个函数其实暂时没用的。

现在我们知道了在进程任何地方调用 ProcessState::self() 就能取到本进程的进程对象，并且可以调用一些 binder 进程相关的接口，在进程任何地方调用 IPCThreadState::self() 就能取得当前线程的对象，并且可以调用一些 binder 线程相关的接口。

## 多线程接口实现.上
其实前面调用的接口就2个：ProcessState 的 startThreadPool 和 IPCThreadState 的 joinThreadPool。我们先来看 startThreadPool：

```cpp
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    // 看样子这个判断，startThreadPool 的调用只会有效一次
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;     
        spawnPooledThread(true);       
    }
}
```

然后看下 spawnPooledThread：

```cpp
// 上面传递过来的 isMain 是 true
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName(); 
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}
```

这个函数很简单，主要是 new 了一个 PoolThread 对象，然后调用了它的 run 方法。这里就先要暂停一下，插说下 android 线程的东西。

## android 中的线程
android 中的线程到底是啥东西。这里先说下答案：就是 pthread，native 层的从代码可以看得出，java 层的虽然是虚拟机实现的，但是应该还是用 pthread 实现的（我猜的，没去看代码）。这里就看下 natvie 的就行了。

首先之前在 ProcessState 中 new 的 PoolThread：

```cpp
class PoolThread : public Thread
{
public:
    PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }
       
protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
    
    const bool mIsMain;
};
```

PoolThread 很简单，它继承自 libutils 里面的 Thread：

```cpp
class Thread : virtual public RefBase
{
public:
    // Create a Thread object, but doesn't create or start the associated
    // thread. See the run() method.
                        Thread(bool canCallJava = true);
    virtual             ~Thread();

    // Start the thread in threadLoop() which needs to be implemented.
    virtual status_t    run(    const char* name = 0,
                                int32_t priority = PRIORITY_DEFAULT,
                                size_t stack = 0); 

... ...

private:
    // Derived class must implement threadLoop(). The thread starts its life
    // here. There are two ways of using the Thread object:
    // 1) loop: if threadLoop() returns true, it will be called again if
    //          requestExit() wasn't called.
    // 2) once: if threadLoop() returns false, the thread will exit upon return.
    virtual bool        threadLoop() = 0;

... ...

};
``` 

这个就算是基类了（父类都是我讨厌的那个引用计数的玩意了）。这类就封装了 android 线程的实现。有2个重要的函数，看注释。一个是 run，看注释说 new 了 Thread 对象后，线程并没有执行，要调用 run 才会跑的。第二是 threadLoop，子类要重载这个函数，就是线程真正的执行函数（回调），如果返回 false，调用一次线程就会结束，如果返回 true，这个函数会循环调用执行。我们看到 PoolThread 重载了 threadLoop 这个函数，在里面调用了 IPCThread 的 joinThreadPool，然后返回 false。

我们一个一个看，首先看下 Thread 的构造函数：

```cpp
Thread::Thread(bool canCallJava)
    :   mCanCallJava(canCallJava),
        mThread(thread_id_t(-1)),
        mLock("Thread::mLock"),
        mStatus(NO_ERROR),
        mExitPending(false), mRunning(false)
#ifdef HAVE_ANDROID_OS
        , mTid(-1)
#endif
{
}

Thread::~Thread()
{
}
```

构造函数挺简单，我们这里关心的是 mCanCallJava 这个 bool 值，默认是 true，这个值表示在 native 的线程能不能调用虚拟机 java 的环境（例如利用个反射，调用一些 java 层的函数等等）。然后我们看 run：

```cpp
status_t Thread::run(const char* name, int32_t priority, size_t stack)
{
    Mutex::Autolock _l(mLock);
    
    // 正在运行的，不允许再执行
    if (mRunning) {
        // thread already started
        return INVALID_OPERATION;
    }
    
    // reset status and exitPending to their default value, so we can
    // try again after an error happened (either below, or in readyToRun())
    mStatus = NO_ERROR;
    mExitPending = false;
    mThread = thread_id_t(-1);
        
    // hold a strong reference on ourself
    mHoldSelf = this;
    
    mRunning = true;

    // 这里，前面说了默认是 true，就是走上面那个分支
    bool res;
    if (mCanCallJava) {
        res = createThreadEtc(_threadLoop,
                this, name, priority, stack, &mThread);
    } else {
        res = androidCreateRawThreadEtc(_threadLoop,
                this, name, priority, stack, &mThread);
    }

    if (res == false) {
        mStatus = UNKNOWN_ERROR;   // something happened!
        mRunning = false;
        mThread = thread_id_t(-1);
        mHoldSelf.clear();  // "this" may have gone away after this.

        return UNKNOWN_ERROR;
    }

    // Do not refer to mStatus here: The thread is already running (may, in fact
    // already have exited with a valid mStatus result). The NO_ERROR indication
    // here merely indicates successfully starting the thread and does not
    // imply successful termination/execution.
    return NO_ERROR;

    // Exiting scope of mLock is a memory barrier and allows new thread to run
}
```

这个函数其实最关键就是 mCanCallJava 这里的这个2个分支，这才是创建线程的地方。mCanCallJava 前面说了默认是 true，就是说一般是走前面那个分支的，但是这里我们先说下面那个分支（至于原因后面你就知道了）：

```cpp
// entryFunction 是线程执行函数指针
int androidCreateRawThreadEtc(android_thread_func_t entryFunction,
                               void *userData,                
                               const char* threadName,        
                               int32_t threadPriority,        
                               size_t threadStackSize,        
                               android_thread_id_t *threadId) 
{
    // 呵呵，是 pthread 的调用了吧
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

#ifdef HAVE_ANDROID_OS  /* valgrind is rejecting RT-priority create reqs */
    if (threadPriority != PRIORITY_DEFAULT || threadName != NULL) {
        // Now that the pthread_t has a method to find the associated
        // android_thread_id_t (pid) from pthread_t, it would be possible to avoid
        // this trampoline in some cases as the parent could set the properties
        // for the child.  However, there would be a race condition because the
        // child becomes ready immediately, and it doesn't work for the name.
        // prctl(PR_SET_NAME) only works for self; prctl(PR_SET_THREAD_NAME) was
        // proposed but not yet accepted.
        thread_data_t* t = new thread_data_t;
        t->priority = threadPriority;  
        t->threadName = threadName ? strdup(threadName) : NULL;
        t->entryFunction = entryFunction;
        t->userData = userData;        
        entryFunction = (android_thread_func_t)&thread_data_t::trampoline;
        userData = t;                  
    }
#endif 

    if (threadStackSize) {
        pthread_attr_setstacksize(&attr, threadStackSize);
    }
    
    errno = 0;
    pthread_t thread;
    int result = pthread_create(&thread, &attr, 
                    (android_pthread_entry)entryFunction, userData);
    pthread_attr_destroy(&attr);
    if (result != 0) {
        ALOGE("androidCreateRawThreadEtc failed (entry=%p, res=%d, errno=%d)\n"
             "(android threadPriority=%d)", 
            entryFunction, result, errno, threadPriority);
        return 0;
    }

    // Note that *threadID is directly available to the parent only, as it is
    // assigned after the child starts.  Use memory barrier / lock if the child
    // or other threads also need access.
    if (threadId != NULL) {
        *threadId = (android_thread_id_t)thread; // XXX: this is not portable
    }
    return 1;
}
```

前面也好说了对 phread 的接口不太清楚的去看《UNIX高级环境编程》。这里简单说下，传了线程执行函数、线程名字、优先级、线程堆栈、用户自定义数据进来，然后那个 threadId 是个输出参数，返回的是线程的 id 号（tid，类似 pid）。函数返回后，`pthread_create` 就会创建线程，并且执行 entryFunction。这里传过来的 entryFunction 是 `_threadLoop`：

```cpp
int Thread::_threadLoop(void* user)
{
    Thread* const self = static_cast<Thread*>(user);

    sp<Thread> strong(self->mHoldSelf);
    wp<Thread> weak(strong);
    self->mHoldSelf.clear();

#ifdef HAVE_ANDROID_OS
    // this is very useful for debugging with gdb
    self->mTid = gettid();
#endif 

    bool first = true;

    do {
        // 还是调用 threadLoop 的
        bool result;
        if (first) {
            first = false;
            self->mStatus = self->readyToRun();
            result = (self->mStatus == NO_ERROR); 

            if (result && !self->exitPending()) {
                // Binder threads (and maybe others) rely on threadLoop
                // running at least once after a successful ::readyToRun()
                // (unless, of course, the thread has already been asked to exit
                // at that point).             
                // This is because threads are essentially used like this:
                //   (new ThreadSubclass())->run();
                // The caller therefore does not retain a strong reference to
                // the thread and the thread would simply disappear after the
                // successful ::readyToRun() call instead of entering the
                // threadLoop at least once.
                result = self->threadLoop();
            }
        } else {
            result = self->threadLoop();
        }

        // 知道前面的注释为什么说返回 false 只会执行一次了
        // establish a scope for mLock
        {
        Mutex::Autolock _l(self->mLock);
        if (result == false || self->mExitPending) {
            self->mExitPending = true;
            self->mRunning = false;
            // clear thread ID so that requestExitAndWait() does not exit if
            // called by a new thread using the same thread ID as this one.
            self->mThread = thread_id_t(-1);
            // note that interested observers blocked in requestExitAndWait are
            // awoken by broadcast, but blocked on mLock until break exits scope
            self->mThreadExitedCondition.broadcast();
            break;
        }
        }

        // Release our strong reference, to let a chance to the thread
        // to die a peaceful death.
        strong.clear();
        // And immediately, re-acquire a strong reference for the next loop
        strong = weak.promote();
    } while(strong != 0);

    return 0;
}
```

线程函数，一旦执行完，线程就会退出。这里的 `_threadLoop` 里面是一个循环，取决于 threadLoop 这个的返回值（怪不得叫 loop）。所以 Thread 的子类只要重载 threadLoop 然后根据自己的需要返回合适值就不用管线程的创建、运行之类的了。


然后我们回去看看 mCanCallJava 那个分支上的那个函数：

```cpp
// AndroidThreads.h ==============================

// inline 又挂马甲
// Create thread with lots of parameters
inline bool createThreadEtc(thread_func_t entryFunction,
                            void *userData,
                            const char* threadName = "android:unnamed_thread",
                            int32_t threadPriority = PRIORITY_DEFAULT,
                            size_t threadStackSize = 0,
                            thread_id_t *threadId = 0)
{
    return androidCreateThreadEtc(entryFunction, userData, threadName,
        threadPriority, threadStackSize, threadId) ? true : false;
}

// Threads.cpp ==============================

// 这个函数指针一开始指向的就是 mCanCallJava == false 那个分支的那个函数
static android_create_thread_fn gCreateThreadFn = androidCreateRawThreadEtc;

int androidCreateThreadEtc(android_thread_func_t entryFunction,
                            void *userData,
                            const char* threadName,
                            int32_t threadPriority,
                            size_t threadStackSize,
                            android_thread_id_t *threadId)
{
    // 是调用上面那个函数指针说指向的函数
    return gCreateThreadFn(entryFunction, userData, threadName,
        threadPriority, threadStackSize, threadId);
}

// 有接口设置的哦
void androidSetCreateThreadFunc(android_create_thread_fn func)
{
    gCreateThreadFn = func; 
}
```

这个分支，按照代码写的最开始和 mCanCallJava == false 是一样的。但是注意，Threads 有个接口可以设置这个函数指针的。我搜了一下代码，发现有一个地方设置了这个函数指针：

```cpp
/*
 * Register android native functions with the VM.
 */
/*static*/ int AndroidRuntime::startReg(JNIEnv* env) 
{
    /*
     * This hook causes all future threads created in this process to be
     * attached to the JavaVM.  (This needs to go away in favor of JNI
     * Attach calls.)
     */
    androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

    ALOGV("--- registering native functions ---\n");

    /*
     * Every "register" function calls one or more things that return
     * a local reference (e.g. FindClass).  Because we haven't really
     * started the VM yet, they're all getting stored in the base frame
     * and never released.  Use Push/Pop to manage the storage.
     */
    env->PushLocalFrame(200);

    if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) { 
        env->PopLocalFrame(NULL);
        return -1;
    }
    env->PopLocalFrame(NULL);

    //createJavaThread("fubar", quickTest, (void*) "hello");

    return 0;
}
```

这个是 AndroidRuntime.cpp 里面注册 jni 函数的时候设置的。这个前面说 SystemService 好像有说过启 Zygote 会调用这个东西。反正就是初始化 java 虚拟机的时候设置了，看注释是为能让在 natvie 的线程中调用 java 中的东西。设置的函数是 javaCreateThreadEtc:

```cpp
/*
 * This is invoked from androidCreateThreadEtc() via the callback
 * set with androidSetCreateThreadFunc().
 *
 * We need to create the new thread in such a way that it gets hooked
 * into the VM before it really starts executing.
 */
/*static*/ int AndroidRuntime::javaCreateThreadEtc(
                                android_thread_func_t entryFunction,
                                void* userData,                
                                const char* threadName,        
                                int32_t threadPriority,        
                                size_t threadStackSize,        
                                android_thread_id_t* threadId) 
{
    void** args = (void**) malloc(3 * sizeof(void*));   // javaThreadShell must free
    int result;

    assert(threadName != NULL);

    // 真正的线程执行函数
    args[0] = (void*) entryFunction;
    // 用户自定义数据
    args[1] = userData;
    // 线程名字
    args[2] = (void*) strdup(threadName);   // javaThreadShell must free

    // 知道前面为什么先说这个函数了吧，最后还是调用同一个
    result = androidCreateRawThreadEtc(AndroidRuntime::javaThreadShell, args,
        threadName, threadPriority, threadStackSize, threadId);
    return result;
}
``` 

知道前面为什么先说 androidCreateRawThreadEtc 了不，其中最终还是调用同一个，所以之后都是 pthread 的调用。但是执行函数不一样，这里的是 javaThreadShell：

```cpp
/*
 * When starting a native thread that will be visible from the VM, we
 * bounce through this to get the right attach/detach action.
 * Note that this function calls free(args)
 */
/*static*/ int AndroidRuntime::javaThreadShell(void* args) {
    void* start = ((void**)args)[0];
    void* userData = ((void **)args)[1];
    char* name = (char*) ((void **)args)[2];        // we own this storage
    free(args);
    JNIEnv* env;
    int result;

    // 就是多了一个 java attach
    /* hook us into the VM */
    if (javaAttachThread(name, &env) != JNI_OK)
        return -1;

    /* start the thread running */
    result = (*(android_thread_func_t)start)(userData);

    // 和 unattach 吧
    /* unhook us */
    javaDetachThread();
    free(name);

    return result;
}
```

这里就多了一个 attach 到 JVM 中。这里不多分析这个，因为这篇不是主要说 JVM 的，反正这么整一下之后，natvie 的线程对 JVM 可见，native 的线程也可以调用 java 的东西。

## 多线程接口实现.下
简单的说了下 android 的线程后，回来继续到 ProcessState 中。前面说了 spawnPooledThread 那里 new 了一个 PoolThread 出来，然后跑 run，之后就会执行 PoolThread 的 threadLoop：

```cpp
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
```

调用的正好是我们下面要说的 IPCThreadState 的 joinThreadPool。这里 mIsMain，PorcessState startThreadPool 转过来的是 true。然后 threadLoop 返回的是 false。就是说这个函数只会执行一次，但是 binder 的 Bn 端不是应该循环等待 Bp 的请求么，往下看你就知道只要一次就够了。

前面那个模板 BinderService 最后也调用了 joinThreadPool 了，这个函数是然当前线程加入到线程池，是当前线程。所以那个模板 BinderService 是主线程加入，这里是 new PoolThread 的线程（pthread 创建的）。我来具体看下：

```cpp
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
       
    // This thread may have been spawned by a thread that was in the background
    // scheduling group, so first we will make sure it is in the foreground
    // one to avoid performing an initial transaction in the background.
    set_sched_policy(mMyThreadId, SP_FOREGROUND);
        
    status_t result;
    do {
        processPendingDerefs();        
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);  
            abort();
        }
        
        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%p\n",
        (void*)pthread_self(), getpid(), (void*)result);
       
    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
```

这个函数其实之前通信模型篇里有说过。里面有个 do while 的循环（前面知道为什么那个 threadLoop 返回 false 了吧，一次就够了，一直在循环咧）。那个 isMain 的区别就是，如果 isMain 是 false， getAndExecuteCommand 如果返回是 `TIMED_OUT` 的话就会退出这个线程。然后 getAndExecuteCommand 返回 `TIMED_OUT` 的条件是 kernel binder 驱动给你返回 `BR_FINISHED`，但是目前的 binder 驱动（通信协议版本 version7）根本就没返回 `BR_FINISHED` 这个值，所以说目前这个 isMain 可以忽略不计，可以认为都是 true。 

getAndExecuteCommand 这些我就不说了，通信模型篇里说得比较清楚了。但是这里你会有个疑问，按照 BinderService 的写法，目前也就只有2个线程再等待 Bp 端的请求而已。如果同一个时候的 Bp 多于2个的话，不是要等待么。别急，binder 在 kernel 会自动帮你处理这种情况的。接下来我们要去 kernel 里面看看 binder 驱动的处理。

## 自动创建线程
通信模型篇，我们知道 binder 驱动中结构体 `binder_proc` 代表一个进程，结构体 `binder_thread` 代表一个线程。我们先看看 `binder_proc` 中几个变量：

```cpp
struct binder_proc {

... ...

    // 保存这个进程中所有线程的红黑树的根节点
    struct rb_root threads;

    // 运行开启的最大线程数
    int max_threads;
    // 请求开启的线程数
    int requested_threads;
    // 应请求运行的线程数
    int requested_threads_started;
    // 准备好的线程（空闲的线程）
    int ready_threads;

... ...

};
```

这个结构在 `binder_open` 的时候会创建（binder open 的操作在 ProcessState 的构造函数那，所以一个进程只会 open 一次，所以一个进程 `binder_proc` 只会创建一次）：

```cpp
static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc; 

    binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
             current->group_leader->pid, current->pid);

    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    if (proc == NULL)
        return -ENOMEM;       
    get_task_struct(current); 
    proc->tsk = current;
    INIT_LIST_HEAD(&proc->todo);
    init_waitqueue_head(&proc->wait);
    proc->default_priority = task_nice(current);
    mutex_lock(&binder_lock); 
    binder_stats_created(BINDER_STAT_PROC);
    hlist_add_head(&proc->proc_node, &binder_procs);
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death);
    filp->private_data = proc;
    mutex_unlock(&binder_lock);

    if (binder_debugfs_dir_entry_proc) {
        char strbuf[11];      
        snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
        proc->debugfs_entry = debugfs_create_file(strbuf, S_IRUGO,
            binder_debugfs_dir_entry_proc, proc, &binder_proc_fops);
    }

    return 0;
}
```

没有显示的初始化那几个变量，不过默认都是 0。然后来看 `binder_thread` 的创建：

```cpp
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;

    /*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret)
        return ret;

    mutex_lock(&binder_lock);
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    } 

... ...

}
```

`binder_thread` 的创建在 `binder_ioctl` 的 `binder_get_thread` 这个函数中。只要调用 binder 的 ioctl 接口就会触发这个：

```cpp
// 咋顺带把这个结构体也贴一下
struct binder_thread {
    // 这个线程所在的进程对象
    struct binder_proc *proc; 
    struct rb_node rb_node;
    int pid;
    // 线程当前运行状态
    int looper;
    struct binder_transaction *transaction_stack;
    struct list_head todo;
    uint32_t return_error; /* Write failed, return error code in read buf */
    uint32_t return_error2; /* Write failed, return error code in read */
        /* buffer. Used when sending a reply to a dead process that */
        /* we are also waiting on */   
    wait_queue_head_t wait;
    struct binder_stats stats;
};

static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
    struct binder_thread *thread = NULL;
    struct rb_node *parent = NULL; 
    struct rb_node **p = &proc->threads.rb_node;

    // proc 中的 thread 按 pid 保存在 proc 的红黑树中
    while (*p) {
        parent = *p;
        thread = rb_entry(parent, struct binder_thread, rb_node);

        if (current->pid < thread->pid)
            p = &(*p)->rb_left;            
        else if (current->pid > thread->pid)
            p = &(*p)->rb_right;           
        else
            break;            
    }
    // 如果之前没创建过，就 new 一个新的出来
    if (*p == NULL) {
        thread = kzalloc(sizeof(*thread), GFP_KERNEL);
        if (thread == NULL)
            return NULL;
        binder_stats_created(BINDER_STAT_THREAD);
        // 设置线程所在的 proc 和 pid
        // 这里的 pid 相当于是 tid 
        thread->proc = proc;
        thread->pid = current->pid;    
        init_waitqueue_head(&thread->wait);
        INIT_LIST_HEAD(&thread->todo);
        // 保存到 proc 中 
        rb_link_node(&thread->rb_node, parent, p);
        rb_insert_color(&thread->rb_node, &proc->threads);
        // 注意初始化状态
        thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
        thread->return_error = BR_OK;  
        thread->return_error2 = BR_OK; 
    }
    return thread;
}
``` 

这里要说下 current 这个全局变量。这个东西代表当前运行的线程的一些结构（其实结构应该是 `task_struct`）。具体的可以去看 linux 内核相关的书。反正用这个东西可以取得到当前运行的线程的一些信息就对了（好像是通过取堆栈最顶的东西）。

然后我们看下 `binder_thread_read` 这里：

```cpp
enum { 
    BINDER_LOOPER_STATE_REGISTERED  = 0x01,
    BINDER_LOOPER_STATE_ENTERED     = 0x02,
    BINDER_LOOPER_STATE_EXITED      = 0x04,
    BINDER_LOOPER_STATE_INVALID     = 0x08,
    BINDER_LOOPER_STATE_WAITING     = 0x10,
    BINDER_LOOPER_STATE_NEED_RETURN = 0x20
};

// Bn 会阻塞等待在这等待 Bp 的请求的到来
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    int ret = 0;
    int wait_for_proc_work;

    if (*consumed == 0) {
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }

retry:
    // transaction_stack == NULL 代表是第一次的 read（Bn 的阻塞read就是）
    // Bn 的阻塞等待的 read todo list 也是空的
    // 所以 Bn 的阻塞 read 这里的 wait_for_proc_work 是 true
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);

    if (thread->return_error != BR_OK && ptr < end) {
        if (thread->return_error2 != BR_OK) {
            if (put_user(thread->return_error2, (uint32_t __user *)ptr))
                return -EFAULT;
            ptr += sizeof(uint32_t);
            if (ptr == end)
                goto done;
            thread->return_error2 = BR_OK;
        }
        if (put_user(thread->return_error, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        thread->return_error = BR_OK;
        goto done;
    }

    // 前面说了这个 looper 是当前线程的状态，
    // 注意这里设置为 WAITING 了，表示正在等待
    thread->looper |= BINDER_LOOPER_STATE_WAITING;
    // Bn read 这里是 true，表示本进程空闲的进程数加1
    if (wait_for_proc_work)
        proc->ready_threads++;
    mutex_unlock(&binder_lock);
    if (wait_for_proc_work) {
        // 这里检测 thread 是不是有下面这2个标志，这2个标志后面会说到。
        // 还有注意前面设置那个 WAITTING 的是用 | 设置的，然后这里检测是用 &
        // 然后看看这几个标志定义的值，会发现这里微妙的用法
        if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
                    BINDER_LOOPER_STATE_ENTERED))) {
            binder_user_error("binder: %d:%d ERROR: Thread waiting "
                "for process work before calling BC_REGISTER_"
                "LOOPER or BC_ENTER_LOOPER (state %x)\n",
                proc->pid, thread->pid, thread->looper);
            wait_event_interruptible(binder_user_error_wait,
                         binder_stop_on_user_error < 2);
        }
        binder_set_nice(proc->default_priority);
        if (non_block) {
            if (!binder_has_proc_work(proc, thread))
                ret = -EAGAIN;
        } else
            // 这里就阻塞在这里，等 thread 的 todo list 不为空（Bp 请求）
            ret = wait_event_interruptible_exclusive(proc->wait, binder_has_proc_work(proc, thread));
    } else {
        if (non_block) {
            if (!binder_has_thread_work(thread))
                ret = -EAGAIN;
        } else
            ret = wait_event_interruptible(thread->wait, binder_has_thread_work(thread));
    }
    mutex_lock(&binder_lock);
    // 如果这个等待的线程被唤醒了（有 Bp 请求来了），
    // 把这个进程空闲的线程数减1，
    // 因为这个线程后面马上就要到用户空间去执行相关业务的函数了。
    if (wait_for_proc_work)
        proc->ready_threads--;
    // 把线程的 WAITTING 标志去掉
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

    // wait 出错的话，返回错误值
    if (ret)
        return ret;

... ...

done:

    *consumed = ptr - buffer;
    // 最后这里 requested_threads 表示发出请求要启动的线程数，
    // ready_threads 表示空闲的线程数。
    // 如果这2个加起来 == 0 就表示当前进程（服务进程）没有空闲的线程来处理请求，
    // 并且还没请求去启动线程，所以需要启动一个新的线程来等待 Bp 的请求。
    // requested_threads_started 表示本进程应请求启动的线程数，
    // 这个不能超过 max_threads 设置的上限。
    if (proc->requested_threads + proc->ready_threads == 0 &&
        proc->requested_threads_started < proc->max_threads &&
        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
         /*spawn a new thread if we leave this out */) {
        // 这里发 BR_SPAWN_LOOPER 到用户去创建新线程去了
        // 然后把请求启动的线程数加1
        proc->requested_threads++;
        binder_debug(BINDER_DEBUG_THREADS,
                 "binder: %d:%d BR_SPAWN_LOOPER\n",
                 proc->pid, thread->pid);
        if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
            return -EFAULT;
    }
    return 0;
}
```

这里 binder 驱动其实用了个很简单的办法来管理线程。就是假设 Bn 端有一个线程 wait 在 read 那了，就相当于多了一个空闲线程（能够处理 Bp 的请求）。上面分析过了，BinderService 一开始就整个2个线程 wait read 那了。然后如果来一个 Bp 请求，`binder_transation` 那里找到目标进程，然后把请求放到目标进程的 todo list 中：

```cpp
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,  
                   struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
    struct binder_work *tcomplete; 
    size_t *offp, *off_end;
    struct binder_proc *target_proc;
    struct binder_thread *target_thread = NULL;
    struct binder_node *target_node = NULL;
    struct list_head *target_list; 
    wait_queue_head_t *target_wait;
    struct binder_transaction *in_reply_to = NULL;
    struct binder_transaction_log_entry *e;
    uint32_t return_error;

... ...

    // 第一次 Bp 发请求给 Bn target_thread 是 null，走的是下面那个
    if (target_thread) {
        e->to_thread = target_thread->pid;
        target_list = &target_thread->todo;
        target_wait = &target_thread->wait;
    } else {
        // todo list 是 target_proc 的
        target_list = &target_proc->todo;
        target_wait = &target_proc->wait;
    }

... ...

    t->work.type = BINDER_WORK_TRANSACTION;
    // 把请求打包成工作（work），加入到 todo list 中
    list_add_tail(&t->work.entry, target_list);
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    // 唤醒等待队列
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;

... ...

}
```

`binder_transtion` 就是消耗 Bn 工作线程的地方了。通信模型篇里分析过了，Bp 第一次发请求给 Bn，是没 `target_thread` 的，所以请求就加入到 `target_proc` 的 todo list 中了。然后唤醒 Bn 在 read 那休眠的线程。

这里说下 wait queue 的小知识，`binder_thread_read` 使用的 `wait_event_interruptible_exclusive` 第二参数是一个检测条件，这里是检测 proc 的 todo list 是否为空，这个会是不停的检测的（应该不怎么耗 cpu 吧），一旦条件为 true 就唤醒继续执行。所以 `binder_transation` 应该没那个唤醒的操作也可以，不过也还是保险一点好。

然后你也发现上面 wait proc 的用的是 `wait_event_interruptible_exclusive`，下面那个 wait thread 用的是 `wait_event_interruptible`。这2个有啥区别咧。我查到的是说 `wait_event_interruptible_exclusive` 在检测唤醒条件的时候是一个互斥过程，是不是说如果有多个线程 wait 的时候只检测一个线程的条件，因为之前的例子，Bn 那已经有2个线程 wait proc 那了。下面那个不带互斥，下面那个由于是 wait 本线程的，所以只会有一个线程 wait。还是说 wait queue 一次只会唤醒一个线程而已。我加的打印发现是唤醒的是最后那个等待的线程。由于比较难写 kernel 的小例子，我这里就不验证这个等待队列的用法了。反正由于种种原因这里一次就只能唤醒一个等待线程。


然后说下 `transaction_stack` 这个东西，看上面知道第一次 `transaction_stack` 是 NULL，也就是 Bn 等待 Bp 来请求的时候是 NULL，然后之后 Bp 通过 `binder_transaction` 把 work 加到 Bn 的 todo list， `transaction_stack` 就保存了 Bp 的 thread，然后 Bn `binder_thread_read` 之后，Bn proc 的 `transaction_stack` Bn 的 thread。然后 `binder_transaction` 一开始判断 reply == true 从 `transaction_stack` 去取 target。这么搞是为了保证， Bp 发送请求到 Bn 的线程处理后，Bn 能返回到正确的 Bp 线程，就是保证返回值能送到发送请求的那个线程处理。这个通信原理有具体分析代码，这里再点一下。这里结合后面说一个进程跑多个服务，binder 的多线程机制并不会导致混乱。 


那这样就差把 binder 驱动里面做的事说完了。每当一个 Bn 的线程在 `binder_thread_read` 等待，`proc->ready_threads` 就会 +1，如果 todo list 里接到请求，最后那个等待的线程被唤醒，`proc->ready_threads` -1，然后唤醒的线程去执行 IPC 请求业务函数，在最后判断是否还有空闲的线程（已经在等待的（`ready_threads`）+发送启动请求的（`requested_threads`）是否为0）。如果没有 Bn 没空闲的线程，并且已经启动的线程（`requested_threads_started`）没超过限制（`max_threas`）就发一个 `BR_SPAWN_LOOPER` 给用户空间去创建线程。

我们来看看用户空间怎么处理 `BR_SPAWN_LOOPER` 的：

```cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;
    
    switch (cmd) {
    ... ...
    case BR_SPAWN_LOOPER:
        // 调用的是 ProcessState 的 spawnPooledThread，参数是 false
        mProcess->spawnPooledThread(false);
        break;
        
    default:
        printf("*** BAD COMMAND %d received from Binder driver\n", cmd);
        result = UNKNOWN_ERROR;
        break; 
    }
        
    if (result != NO_ERROR) {
        mLastError = result;
    }
    
    return result;
}
```

结果就是 kernel 帮我们自动调用 ProcessState 的 PooledThread 而已。这里的参数是 false 了，就是说 isMain 是 false 了。这个 isMain 前面说没啥用的，不过还是有点小区别。回去看 joinThreadPool（spwanPooledThread 最后还是调用了 joinThreadPool） 那：

```cpp
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
```

isMain 的对 binder 发的是 `BC_ENTER_LOOPER`，false 的发的是 `BC_REGISTER_LOOPER`。我去看下这2个有啥区别不：

```cpp
int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
            void __user *buffer, int size, signed long *consumed)
{
    uint32_t cmd;
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    while (ptr < end && thread->return_error == BR_OK) {
        if (get_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;   
        ptr += sizeof(uint32_t);       
        if (_IOC_NR(cmd) < ARRAY_SIZE(binder_stats.bc)) { 
            binder_stats.bc[_IOC_NR(cmd)]++;
            proc->stats.bc[_IOC_NR(cmd)]++;
            thread->stats.bc[_IOC_NR(cmd)]++;
        }
        switch (cmd) {
... ...
        case BC_REGISTER_LOOPER:
            // 不允许已经 ENTER_LOOPER 的线程再 REGISTER_LOOPER
            binder_debug(BINDER_DEBUG_THREADS,
                     "binder: %d:%d BC_REGISTER_LOOPER\n",
                     proc->pid, thread->pid);
            if (thread->looper & BINDER_LOOPER_STATE_ENTERED) {
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
                binder_user_error("binder: %d:%d ERROR:"
                    " BC_REGISTER_LOOPER called "
                    "after BC_ENTER_LOOPER\n",
                    proc->pid, thread->pid);
            // 没发起请求的也不允许 REGISTER_LOOPER
            } else if (proc->requested_threads == 0) {
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
                binder_user_error("binder: %d:%d ERROR:"
                    " BC_REGISTER_LOOPER called "
                    "without request\n",
                    proc->pid, thread->pid);
            } else {
                // 请求线程数 -1
                proc->requested_threads--;
                // 已经启动的线程数 +1
                proc->requested_threads_started++;
            }   
            // 设置下线程状态
            thread->looper |= BINDER_LOOPER_STATE_REGISTERED;
            break;
        case BC_ENTER_LOOPER:
            // 不允许 REGISTER_LOOPER 的线程再 ENTER_LOOPER
            binder_debug(BINDER_DEBUG_THREADS,
                     "binder: %d:%d BC_ENTER_LOOPER\n",
                     proc->pid, thread->pid);
            if (thread->looper & BINDER_LOOPER_STATE_REGISTERED) {
                thread->looper |= BINDER_LOOPER_STATE_INVALID;
                binder_user_error("binder: %d:%d ERROR:"
                    " BC_ENTER_LOOPER called after "
                    "BC_REGISTER_LOOPER\n",
                    proc->pid, thread->pid);
            }
            // 设置下线程状态
            thread->looper |= BINDER_LOOPER_STATE_ENTERED;
            break;
        case BC_EXIT_LOOPER:
            // 这个退出只是把状态设置了下
            binder_debug(BINDER_DEBUG_THREADS,
                     "binder: %d:%d BC_EXIT_LOOPER\n",
                     proc->pid, thread->pid);
            thread->looper |= BINDER_LOOPER_STATE_EXITED;
            break;

... ...

        default:
            printk(KERN_ERR "binder: %d:%d unknown command %d\n",
                   proc->pid, thread->pid, cmd);
            return -EINVAL;
        }
        *consumed = ptr - buffer;
    }
    return 0;
}
```

isMain 的除去是否会超时自动退出外（前面说了，这个机制目前是没用的），就是上面那些判断了，isMain 是 true 是手动创建的，所以没什么限制。false 则是 binder 驱动根据当前可用线程数的情况自动请求创建的，所以确定 binder 驱动确实请求了才允许创建（requested_threads 不为 0）。还有 `BC_ENTER_LOOPER` 的是不算在 `requested_threads_started` 里面的，所以手动启动的线程理论上可以无限个（不过我看到 SS 除了一开始手动调用一次 startPoolThread，然后把当前线程 joinThreadPool 之外，就再也没手动启动过线程，都是交由驱动管理）。


然后说下既然有创建线程，那什么时候退出呢。前面说了超时暂时没用，joinThreadPool 那里 io 错误也会导致退出，这些异常的不算。正常目前好像没那个地方会退出，binder 启动的线程。是的，没错，目前 binder 的线程一旦启动就不会退出的，直到达到上限为止，只要 binder 一发现当前服务进程没空闲线程了就会 spawn 一个出来。但是由于服务线程一旦执行完 IPC 调用就会变成空闲的，所以只要同一个时候没很多 Bp 请求过来，不会创建太多线程的。而且就算创建了不少线程，这些线程只是休眠而已，不乎占用 cpu 资源，所以没啥太大关系，可能后续的 binder 版本会考虑一段时间这个线程没有执行请求就把这个线程退出吧。

然后 binder 自动创建的线程数可以设置上限的。说起这个 binder 有条命令： `BINDER_SET_MAX_THREADS`:

```cpp
// binder_thread_write:

    // 就是把进程的 max_threads 设置了一下
    case BINDER_SET_MAX_THREADS:
        if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
            ret = -EINVAL;
            goto err;
        }
        break;
```

说到这个，我们来看看默认设置最大线程数是多少吧：

```cpp
static int open_driver()
{
    int fd = open("/dev/binder", O_RDWR); 
    if (fd >= 0) {
        fcntl(fd, F_SETFD, FD_CLOEXEC);
        int vers;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
            ALOGE("Binder driver protocol does not match user space protocol!");
            close(fd);
            fd = -1;
        }
        // 最大线程数为 15
        size_t maxThreads = 15;        
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
    }
    return fd;
}
```

ProcessState 的构造函数会调用 `open_driver()`，binder 默认允许最大的线程数为 15 个。一般来说继承 BindService 的默认就是用这个数量了，但是也有特殊的。SurfaceFlinger（SF）没 BindService，手动写的：

```cpp
// main_surfaceflinger.cpp =============================

int main(int argc, char** argv) {
    // 最大线程数为 4
    // When SF is launched in its own process, limit the number of
    // binder threads to 4.
    ProcessState::self()->setThreadPoolMaxThreadCount(4);

    // 还有一个主线程（isMain）
    // start the thread pool
    sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();

    // instantiate surfaceflinger
    sp<SurfaceFlinger> flinger = new SurfaceFlinger();

#if defined(HAVE_PTHREADS)
    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
#endif
    set_sched_policy(0, SP_FOREGROUND);

    // initialize before clients can connect
    flinger->init();

    // publish surface flinger
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false);

    // run in this thread
    flinger->run();

    return 0;
}

// ProcessState.cpp ===============================

status_t ProcessState::setThreadPoolMaxThreadCount(size_t maxThreads) {
    status_t result = NO_ERROR;
    if (ioctl(mDriverFD, BINDER_SET_MAX_THREADS, &maxThreads) == -1) {
        result = -errno;
        ALOGE("Binder ioctl to set max threads failed: %s", strerror(-result));
    }
    return result;
} 
```

SF 就只设置了4个可以自动创建的线程，还有一个主线程（应该还有一个吧，当前那个线程跑哪去了，以后分析 SF 再说吧）。这里注意一点，如果要手动写的话，主要设置要放到 IPCThreadState->joinThreadPool 的前面，因为调用 joinThreadPool 当前线程就加入到 binder 的主线程中去了，阻塞循环就开始了，除非线程退出了，否则后面的代码执行不到的。


前面讨论了线程过多的情况，如果 Bn 中的线程不够的话，会怎么样咧。就是当前服务进程中没空闲的线程了（都在执行业务函数），答案是 Bp 端等待。前面不是说 proc 有 todo list 么（thread 也有的），既然是 list 就可以保存多个工作请求（work），然后一次处理一个，当没空闲线程马上处理的时候，把请求加到 todo list 后面，等线程执行完当前的业务函数，回到 `binder_thread_read` 那，准备等待的时候，发现 todo list 中不空，就马上取下一个 work 然后执行，就不会进入休眠，直到 todo list 为空。所以如果服务进程空闲线程不足，然后这个时候 Bp 请求又多的话，会导致 IPC 调用非常慢（要排队等，哥又想起排队抢火车票了）。所以要根据自身业务的情况去设置 binder 自动创建线程的上限，不要设得太小。顺带贴下线程取工作请求：

```cpp
// binder_thread_read: 

    while (1) {
        uint32_t cmd; 
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;

        // 去 todo list 中取工作请求
        if (!list_empty(&thread->todo))
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        else if (!list_empty(&proc->todo) && wait_for_proc_work)
            w = list_first_entry(&proc->todo, struct binder_work, entry);
        else {
            if (ptr - buffer == 4 && !(thread->looper & BINDER_LOOPER_STATE_NEED_RETURN)) /* no data added */
                goto retry;
            break;
        }    

        if (end - ptr < sizeof(tr) + 4) 
            break;

... ...

    }
```

所以那些 SS 里面很多函数都加了锁，因为有多线程的支持，可能同一个时间在执行不同客户端的业务函数。有可能 Bp1 的执行的函数是读一个变量，而 Bp2 执行的函数正好要写这个变量，这个时候需要互斥操作，所以很多操作都带锁了。但是注意不要什么操作都带锁，因为互斥锁会影响执行效率的，对于可以重入的函数不需要加锁（例如函数里面全是局部变量）。

还有要注意 SS 协同完成一些任务的时候的问题，例如一个 Proc A IPC 调用 Proc B 的某个函数，这个函数又跑去调用 AM 里面的某个函数，然后 AM 这个函数正好又去 IPC 调用 Proc B 中的某个函数。如果这些 IPC 函数都加了锁的话，就会死锁。关于这个例子可以去看 Binder 对象传递的普通服务篇。

## java 层的 SS
前面差不多 binder 多线程的支持说完了，也看了一些 native SS 的例子。最后来看下 java 层的，也是我们应用开发中接触最多的那一票系统服务（AM、WM、PM 等）。前面说这个有点麻烦，主要是它还得启动 java 虚拟机。我们来一点点看吧，顺带把 SS 启动过程也走一下。

首先说了 init.rc 里有启动 zygote（不是 SS 么，怎么变成 zygote 了，别急，往下看）：

<pre config="brush:bash;toolbar:false;">
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
</pre>

注意那个 `--start-system-server` 和 `--zygote` 的参数。其实是启动 `app_process` 这个 native 程序。我们来先上一张序列图，还挺麻烦的说（因为是通过启动一个程序（zygote），然后再 fork 出另外一个（SS））：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-thread/1.png)

我来来看下 app_process 的 main 函数：

```cpp
// app_main.cpp ================================

int main(int argc, char* const argv[])
{

... ...

    AppRuntime runtime;
    const char* argv0 = argv[0];

    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    // Everything up to '--' or first non '-' arg goes to the vm

    int i = runtime.addVmArguments(argc, argv);

    // init.rc 传过来的参数 --zygote 和 --start-system-server
    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    const char* parentDir = NULL;
    const char* niceName = NULL;
    const char* className = NULL;
    while (i < argc) {
        const char* arg = argv[i++];
        if (!parentDir) {
            parentDir = arg;
        } else if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = "zygote";
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName = arg + 12;
        } else {
            className = arg;
            break;
        }
    }

    if (niceName && *niceName) {
        setArgv0(argv0, niceName);
        set_process_name(niceName);
    }

    runtime.mParentDir = parentDir;

    if (zygote) {
        // 那就是走这里的
        runtime.start("com.android.internal.os.ZygoteInit",
                startSystemServer ? "start-system-server" : "");
    } else if (className) {
        // Remainder of args get passed to startup class main()
        runtime.mClassName = className;
        runtime.mArgC = argc - i;
        runtime.mArgV = argv + i;
        runtime.start("com.android.internal.os.RuntimeInit",
                application ? "application" : "tool");
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

init.rc 传过来的参数来看，跑去调用 AppRuntime 的 start 去了。AppRuntime 在 `app_main.cpp` 中：

```cpp
class AppRuntime : public AndroidRuntime
{
public:
    AppRuntime()
        : mParentDir(NULL)
        , mClassName(NULL)
        , mClass(NULL)
        , mArgC(0)
        , mArgV(NULL)
    {
    }

... ...

    // 注意这个函数
    virtual void onZygoteInit()
    {
        // Re-enable tracing now that we're no longer in Zygote.
        atrace_set_tracing_enabled(true);

        // 这里初始化了 ProcessState，并调用了 startThreadPool
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }

... ...

    const char* mParentDir;
    const char* mClassName;
    jclass mClass;
    int mArgC;
    const char* const* mArgV;
};
``` 

AppRuntime 继承 AndroidRuntime，我们得去看父类里面的 start：

```cpp
// 前面传过来的 className 是： com.android.internal.os.ZygoteInit
// options 是： start-system-server
/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const char* options)
{
    ALOGD("\n>>>>>> AndroidRuntime START %s <<<<<<\n",
            className != NULL ? className : "(unknown)");

    /* 
     * 'startSystemServer == true' means runtime is obsolete and not run from
     * init.rc anymore, so we print out the boot start event here.
     */
    if (strcmp(options, "start-system-server") == 0) { 
        /* track our progress through the boot sequence */
        const int LOG_BOOT_PROGRESS_START = 3000;
        LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,
                       ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
    }

    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";  
        if (!hasDir("/system")) {      
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

    // 启动 VM，这里不管这个
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env) != 0) {
        return;
    }
    // 之前的子类 AppRuntime 有重载，不过这里启动 zygote 这个函数直接返回的
    onVmCreated(env);

    // 这个 startReg 还记得不，前面说 android 线程的时候
    // 改变 libutils Threads 里面的线程函数指针就是在这个函数里面
    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;
    jstring optionsStr;

    // 下面这一大串就是取 className 的 main 函数，然后执行而已
    // className 是 com.android.internal.os.ZygoteInit
    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(2, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
    optionsStr = env->NewStringUTF(options);
    // 把 start-system-service 参数打包成 java 函数的参数
    env->SetObjectArrayElement(strArray, 1, optionsStr);

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        // 取 className 的 main 函数
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            // 通过反射调用 main 函数，
            // 注意把前面的 start-system-service 当作参数传过去了
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

这里忽略了 JVM 的启动过程（和 binder 关系不大），然后后面主要就是启动 java 层里面的东西了。这里是去执行了 com.android.internal.os.ZygoteInit 的 main 函数（下面欢迎进入 java 世界）：

```java
    public static void main(String argv[]) {
        try {
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();

            // 打开 zygote 通信用的 socket
            registerZygoteSocket();
             // Finish profiling the zygote initialization.
            boolean isFirstBooting = false;
            //if first time booting and zygote restart need preload full class
            if(Process.myPid() > 300 || SystemProperties.getBoolean(PROPERTY_FIRST_TIME_BOOTING, true)){
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                    SystemClock.uptimeMillis());   
                preload();
                isFirstBooting = true;         
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                    SystemClock.uptimeMillis());   
            }                 
            // Finish profiling the zygote initialization.
            SamplingProfilerIntegration.writeZygoteSnapshot();

            // Do an initial gc to clean up after startup
            gc();

            // Disable tracing so that forked processes do not inherit stale tracing tags from
            // Zygote.
            Trace.setTracingEnabled(false);

            // If requested, start system server directly from Zygote
            if (argv.length != 2) {        
                throw new RuntimeException(argv[0] + USAGE_STRING);
            }

            // 那个参数传了半天终于有用了，这个函数就是去启动 SS 的进程
            if (argv[1].equals("start-system-server")) {
                startSystemServer();           
            } else if (!argv[1].equals("")) {
                throw new RuntimeException(argv[0] + USAGE_STRING);
            }

            Log.i(TAG, "Accepting command socket connections");
            if(!isFirstBooting){           
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                    SystemClock.uptimeMillis());   
                preload();
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                    SystemClock.uptimeMillis());   
                gc();
            }
            // 这个就是 zygote 循环阻塞等待请求的函数了
            runSelectLoop();

            // 上面那个函数退出就说明 zygote 退出了，关闭之前打开的 socket
            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```

我们这里主要看 startSystemServer 这个函数，其它的是 zygote 的，zygote 的话，我有一篇工作小笔记（换系统字体那个）有说到，这里不多说。

```java
    /*
     * Prepare the arguments and fork for the system server process.
     */
    private static boolean startSystemServer()
            throws MethodAndArgsCaller, RuntimeException {
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_RESOURCE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG
        );  
        // 下面一堆参数，注意最下面那个类名（我们 SS 终于露面了）
        /* Hardcoded command line to start the system server */
        String args[] = { 
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };  
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            // 将上面那堆字符串参数解析成一个专门用来 zygote 通信的数据结构
            // 不过这里启动 SS 根本没和 zygote 通信，这里纯粹借用这个数据而已
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            // fork SS 进程
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }   

        // pid == 0 是 fork 出来的 SS 进程，要去启动 SS 服务了
        /* For child process */
        if (pid == 0) {
            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
```

我们去看下 Zygote.forkSystemServer 这个函数：

```java
    /*
     * Special method to start the system server process. In addition to the
     * common actions performed in forkAndSpecialize, the pid of the child
     * process is recorded such that the death of the child process will cause
     * zygote to exit.
     *
     * @param uid the UNIX uid that the new process should setuid() to after
     * fork()ing and and before spawning any threads.
     * @param gid the UNIX gid that the new process should setgid() to after
     * fork()ing and and before spawning any threads.
     * @param gids null-ok; a list of UNIX gids that the new process should
     * setgroups() to after fork and before spawning any threads.
     * @param debugFlags bit flags that enable debugging features.
     * @param rlimits null-ok an array of rlimit tuples, with the second
     * dimension having a length of 3 and representing
     * (resource, rlim_cur, rlim_max). These are set via the posix
     * setrlimit(2) call.
     * @param permittedCapabilities argument for setcap()
     * @param effectiveCapabilities argument for setcap()
     *
     * @return 0 if this is the child, pid of the child
     * if this is the parent, or -1 on error.
     */
    public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        preFork();
        int pid = nativeForkSystemServer(
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        postFork();
        return pid;
    }
```

看注释这个函数专门为 SS 写的，不去深究这个函数了，最后应该 jni 调用 linux 的 fork 函数吧。SS 是 zygote 第一个 fork 出来的值进程。然后回到 startSystemService 那里，后面有 pid == 0 的，这个就表示 SS 的进程，然后进到 handleSystemServerProcess：

```java
    // 注意前面打包的那个参数数据结构
    /*
     * Finish remaining work for the newly forked system server process.
     */
    private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {

        // 由于 fork 继承了父进程的 socket，
        // SS 不需要这个，所以关掉
        closeServerSocket();

        // set umask to 0077 so new files and directories will default to owner-only permissions.
        Libcore.os.umask(S_IRWXG | S_IRWXO);

        // 这个 niceName 是上面那个 system_server
        // adb shell ps 能看得到 SS 的名字是 system_server
        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        // 上面没设置 inovkeWith 所以走的下面那个分支
        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    null, parsedArgs.remainingArgs);
        } else {
            // 这个remainingArgs是最后那个 com.android.server.SystemServer
            /*
             * Pass the remaining arguments to SystemServer.
             */
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs);
        }

        /* should never reach here */  
    }
```

上面看那个 nineName 设置为 `system_server`，这个在 adb shell ps 能看得到的，还能看得出 SS 确实是 zygote 的子进程（看 SS 的父进程号）：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-thread/2.png)

这里终于跑到 SS 进程里面去了，然后我接下去看 RuntimeInit.zygoteInit：

```java
    /*
     * The main function called when started through the zygote process. This
     * could be unified with main(), if the native code in nativeFinishInit()
     * were rationalized with Zygote startup.<p>
     * 
     * Current recognized args:
     * <ul>
     *   <li> <code> [--] <start class name>  <args>
     * </ul>
     * 
     * @param targetSdkVersion target SDK version
     * @param argv arg strings
     */
    public static final void zygoteInit(int targetSdkVersion, String[] argv)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

        // 这个不管
        redirectLogStreams();

        // 这个 commonInit 这里也没什么要特别注意的
        commonInit();
        // 下面那个要注意下
        nativeZygoteInit();

        // 启动 SS 的 java 类的在这个函数里面
        applicationInit(targetSdkVersion, argv);
    }
```

这个 nativeZygoteInit 在 AndroidRuntime 里面：

```cpp
static AndroidRuntime* gCurRuntime = NULL;

static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}

AndroidRuntime::AndroidRuntime() :
        mExitWithoutCleanup(false)     
{
    SkGraphics::Init();
    // this sets our preference for 16bit images during decode
    // in case the src is opaque and 24bit
    SkImageDecoder::SetDeviceConfig(SkBitmap::kRGB_565_Config);
    // This cache is shared between browser native images, and java "purgeable"
    // bitmaps. This globalpool is for images that do not either use the java
    // heap, or are not backed by ashmem. See BitmapFactory.cpp for the key
    // java call site.
    SkImageRef_GlobalPool::SetRAMBudget(512 * 1024);
    // There is also a global font cache, but its budget is specified in code
    // see SkFontHost_android.cpp

    // Pre-allocate enough space to hold a fair number of options.
    mOptions.setCapacity(20);

    // gCurRuntime 每个进程一个
    assert(gCurRuntime == NULL);        // one per process
    gCurRuntime = this;
}
```

还记得最开始 AppRuntime 这个之类重载的 onZygoteInit 么，没错就是在这里调用了。这里初始化了 ProcessState（打开 binder 驱动，设置 binder 自动线程最大数目为 15），然后手动启动了一个主线程。

然后我们继续看 applicationInit：

```java
    private static void applicationInit(int targetSdkVersion, String[] argv)
            throws ZygoteInit.MethodAndArgsCaller {
        // If the application calls System.exit(), terminate the process
        // immediately without running any shutdown hooks.  It is not possible to
        // shutdown an Android application gracefully.  Among other things, the
        // Android runtime shutdown hooks close the Binder driver, which can cause
        // leftover running threads to crash before the process actually exits.
        nativeSetExitWithoutCleanup(true);

        // 设置一下虚拟机的一些参数
        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
            args = new Arguments(argv);    
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());  
            // let the process exit        
            return;
        }

        // 终于快到了
        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs);
    }
```

继续看 invokeStaticMain（名字都叫这个份上了）：

```java
    /*
     * Invokes a static "main(argv[]) method on class "className".
     * Converts various failing exceptions into RuntimeExceptions, with
     * the assumption that they will then cause the VM instance to exit.
     * 
     * @param className Fully-qualified class name
     * @param argv Argument vector for main()
     */
    private static void invokeStaticMain(String className, String[] argv)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl;

        // com.android.server.SystemServer 这个类名终于有用了
        try {                 
            cl = Class.forName(className); 
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(    
                    "Missing class when invoking static main " + className,
                    ex);
        }

        // 终于去取 main 函数了
        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(    
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) { 
            throw new RuntimeException(    
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(    
                    "Main method is not public and static on " + className);
        }

        // 这里搞了个花样，好像是通过抛出异常来清理调用堆栈
        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```

这个函数前面都没什么，关键是最后一句，抛出一个异常。看注释说这个异常在 ZygoteInit.main 中有捕获，我们回去看一下：

```java
    public static void main(String argv[]) {
        try {
... ...
         
        } catch (MethodAndArgsCaller caller) {
            // 在这里运行抛过来的函数
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```

还真有捕获，然后在 catch 直接运行传递过来的 main 函数。看注释说这么做的原因是为了清除函数调用堆栈（确实从调用到 SS 的 main 函数，前面函数堆栈有好几层了）。这样能让 SS 感觉更像直接启动的吧（忽悠谁咧）。


然后我们终于能够去看 SS 的业务了，SS 的 main 函数：

```java
    public static void main(String[] args) {

        /*
         * In case the runtime switched since last boot (such as when
         * the old runtime was removed in an OTA), set the system
         * property so that it is in sync. We can't do this in
         * libnativehelper's JniInvocation::Init code where we already
         * had to fallback to a different runtime because it is
         * running as root and we need to be the system user to set
         * the property. http://b/11463182
         */
        SystemProperties.set("persist.sys.dalvik.vm.lib",
                             VMRuntime.getRuntime().vmLibrary());

        if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
            // If a device's clock is before 1970 (before 0), a lot of
            // APIs crash dealing with negative numbers, notably
            // java.io.File#setLastModified, so instead we fake it and
            // hope that time from cell towers or NTP fixes it
            // shortly.
            Slog.w(TAG, "System clock is before 1970; setting to 1970.");
            SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        }    

        if (SamplingProfilerIntegration.isEnabled()) {
            SamplingProfilerIntegration.start();
            timer = new Timer();
            timer.schedule(new TimerTask() {
                @Override
                public void run() {
                    SamplingProfilerIntegration.writeSnapshot("system_server", null);
                }    
            }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
        }    

        // Mmmmmm... more memory!
        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();

        // The system server has to run all of the time, so it needs to be
        // as efficient as possible with its memory usage.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

        Environment.setUserRequired(true);

        System.loadLibrary("android_servers");

        Slog.i(TAG, "Entered the Android system server!");

        // 启动 native 服务
        // Initialize native services.
        nativeInit();

        // 启动 java 层服务
        // This used to be its own separate thread, but now it is
        // just the loop we run on the main thread.
        ServerThread thr = new ServerThread();
        thr.initAndLoop();
    }
```

这个 main 其实不长（后面的在后面那个 ServerThread 里面），先是 nativeInit 启动 native 的服务：

```cpp
// com_android_server_SystemServer.cpp  ========================

static void android_server_SystemServer_nativeInit(JNIEnv* env, jobject clazz) {
    char propBuf[PROPERTY_VALUE_MAX];
    property_get("system_init.startsensorservice", propBuf, "1");
    // 就启动了一个 SensorService 
    if (strcmp(propBuf, "1") == 0) {
        // Start the sensor service
        SensorService::instantiate();
    }
}
```

然后主要工作在 ServerThread 中。这个是 SystemServer 的内部类，看名字就很形象，服务线程啊：

```java
    // 这个函数巨长，有 1000 多行，我只贴一些有代表性的
    public void initAndLoop() {
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN,
            SystemClock.uptimeMillis());   

        // 创建主线程 Looper
        Looper.prepareMainLooper();    

        android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);

        BinderInternal.disableBackgroundScheduling(true);
        android.os.Process.setCanSelfBackground(false);

        // Check whether we failed to shut down last time we tried.
        {
            final String shutdownAction = SystemProperties.get(
                    ShutdownThread.SHUTDOWN_ACTION_PROPERTY, "");
            if (shutdownAction != null && shutdownAction.length() > 0) { 
                boolean reboot = (shutdownAction.charAt(0) == '1');

                final String reason;           
                if (shutdownAction.length() > 1) {
                    reason = shutdownAction.substring(1, shutdownAction.length());
                } else {
                    reason = null;                 
                }

                ShutdownThread.rebootOrShutdown(reboot, reason);
            }
        }

        String factoryTestStr = SystemProperties.get("ro.factorytest");
        int factoryTest = "".equals(factoryTestStr) ? SystemServer.FACTORY_TEST_OFF
                : Integer.parseInt(factoryTestStr);
        final boolean headless = "1".equals(SystemProperties.get("ro.config.headless", "0"));

        // 看到这一票熟悉的 Manager 了没
        // 这里不是全部的，下面还有，不贴了
        Installer installer = null;
        AccountManagerService accountManager = null;
        ContentService contentService = null;
        LightsService lights = null;
        PowerManagerService power = null;
        DisplayManagerService display = null;
        BatteryService battery = null;
        VibratorService vibrator = null;
        AlarmManagerService alarm = null;
        MountService mountService = null;
        NetworkManagementService networkManagement = null;
        NetworkStatsService networkStats = null;
        NetworkPolicyManagerService networkPolicy = null;
        ConnectivityService connectivity = null;
        EthernetService eth = null;
        WifiP2pService wifiP2p = null;
        WifiService wifi = null;
        NsdService serviceDiscovery= null;
        IPackageManager pm = null;
        Context context = null;
        WindowManagerService wm = null;
        BluetoothManagerService bluetooth = null;
        DockObserver dock = null;
        UsbService usb = null;
        SerialService serial = null;
        TwilightService twilight = null;
        UiModeManagerService uiMode = null;
        RecognitionManagerService recognition = null;
        NetworkTimeUpdateService networkTimeUpdater = null;
        CommonTimeManagementService commonTimeMgmtService = null;
        InputManagerService inputManager = null;
        TelephonyRegistry telephonyRegistry = null;
        ConsumerIrService consumerIr = null;

... ...

        try {
            // new Service 对象，然后 add 到 SM 中，非常规范的操作
            // 这里也只贴前面几个，下面都差不多，无非有几个玩非主流
            // 后面分析到那些再说。
            Slog.i(TAG, "Display Manager");
            display = new DisplayManagerService(context, wmHandler);
            ServiceManager.addService(Context.DISPLAY_SERVICE, display, true);

            Slog.i(TAG, "Telephony Registry");
            telephonyRegistry = new TelephonyRegistry(context);
            ServiceManager.addService("telephony.registry", telephonyRegistry);

            Slog.i(TAG, "Scheduling Policy");
            ServiceManager.addService("scheduling_policy", new SchedulingPolicyService());

            AttributeCache.init(context);

            if (!display.waitForDefaultDisplay()) {
                reportWtf("Timeout waiting for default display to be initialized.",
                        new Throwable());
            }

... ...

        } catch (RuntimeException e) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting core service", e);
        }

... ... 

        // 主线程 Looper 开始循环
        Looper.loop();
        Slog.d(TAG, "System ServerThread is exiting!");
    }
```

看 SS main 最后那里注释说，这些 SS 应该在单独的线程里面跑，但是现在暂时都在主线程中了。看 ServerThread 中代码，是都是在主线程中。SS 是一个典型的一个进程里面多个服务的例子（服务还很多，好像有十几个吧 -_-||）。DDMS 里面的截图很明显了：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-thread/3.png)
（pid 430 是上面 ps 看到的 SS 的）

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/Binder-thread/4.png)

之前还觉得 binder 默认给自动线程设置 15 个是不是有点多，现在看起来这个值是不是就是对着 SS 来设的。结合前面的讨论，15 个加上前面那个主动开的线程应该能应付大多数情况。看截图一般高峰期就 11、12 个 Bp 吧（`Binder_C`、`Binder_D`，前面说了 kernel 自动调节的线程目前一旦传了，不会退出的，所以这个最大数差不多能说明最大负载的情况吧）。然后看到这些线程的名字都是 Binder_X、这个名字是在 ProcessState 里取的：

```cpp
String8 ProcessState::makeBinderThreadName() {
    int32_t s = android_atomic_add(1, &mThreadPoolSeq);
    String8 name;
    name.appendFormat("Binder_%X", s);
    return name;
}
```

这不是 isMain 的线程连名字都是大众脸（所以看到 Binder_X 的线程，就知道是 kernel binder 驱动自动调节创建出来的）。


然后这里再讨论一个问题，就是 SS 一个进程同时在跑多个不同的服务，那不同的线程如果能够把不同服务的 Bp 请求发送到对应的服务处理呢。刚开始我也觉得有点不好理解，但是从上一篇 binder 对象传递之后我就明白了，binder 根本不需要区分这个请求是服务进程中哪个服务的。还记得 binder 对象传递篇中，服务向 SM 注册的时候，传递过去的对象是 Bn，然后把本地对象的指针传递过去了，然后在 kernel 的 `binder_node` 中有保存这个本地指针的。所以只要能找到正确的 node （通过的 Bp 的 handle 值取 ref，再通过 ref 取 node，忘记了的回去看 binder 对象传递篇）就能取得服务的本地对象指针，然后在服务进程的线程中执行这个本地对象的方法就能执行正确的服务函数了。所以这些对于多线程，一个进程多个服务来说是透明，它们不用管这个 binder 对象是本进程内哪个服务的，只管执行它的 transation 方法就行。

## 总结
android 真的为 IPC 做了很多事情，前面说的效率、框架封装不说，这里直接帮你把并发多线程支持给你解决了。但是还有 framework 还是有些服务不是用 binder 的（vold、zygote），唉，人多了，另起炉灶就再所难免了。


