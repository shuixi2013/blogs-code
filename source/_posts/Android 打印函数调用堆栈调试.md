title: Android 打印函数调用堆栈调试
date: 2015-01-26 23:37:16
categories: [Android Development]
tags: [android]
---

## java 层
可以利用抛出异常来打印：

```java
RuntimeException here = new RuntimeException("here");
here.fillInStackTrace();
Log.i(TAG, "call statck is", here);
```

## native 层
可以利用 android 的 CallStack 来打印：

```cpp
CallStack stack;
stack.update();
stack.dump("SurfaceFlinger"); 

// 4.4 的接口变了 dump 是用来把 log 保存到文件里面去的
// 单纯的打印用这个: statck.log("SurfaceFlinger")
```

要调用这个类的方法先插入 CallStack 头文件，然后在 Android.mk 中链接 utils 库就可以： 

```cpp
#include <utils/CallStack.h>
LOCAL_SHARED_LIBRARIES := libutils
```

这个可以自己去看 CallStack.h 的头文件（frameworks/base/native/include/utils/CallStack.h），方法的定义:

```cpp
// 第一个参数没去研究啥作用，用默认的1吧，
// 第二个参数好像是设置追踪的最大调用堆栈深度，默认是 31
void update(int32_t ignoreDepth=1, int32_t maxDepth=MAX_DEPTH);

// 这个就是把调用堆栈信息输出到 android 的 log 里面，
// 那个参数是 log 前面的前缀
// Dump a stack trace to the log
void dump(const char* prefix = 0) const;
```

效果如下：(注意 logcat 的 TAG 是 CallStack -_-||)
<pre config="brush:bash;toolbar:false;">
D/CallStack(  101): SurfaceFlinger#00  pc 00028a84  /system/lib/libsurfaceflinger.so (android::SurfaceFlinger::captureScreen(android::sp<android::IBinder> const&, android::sp<android::IMemoryHeap>*, unsigned int*, unsigned int*, int*, unsigned int, unsigned int, unsigned int, unsigned int)+63)
D/CallStack(  101): SurfaceFlinger#01  pc 000262a4  /system/lib/libgui.so (android::BnSurfaceComposer::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+523)
D/CallStack(  101): SurfaceFlinger#02  pc 00029836  /system/lib/libsurfaceflinger.so (android::SurfaceFlinger::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+105)
D/CallStack(  101): SurfaceFlinger#03  pc 0001435e  /system/lib/libbinder.so (android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+57)
D/CallStack(  101): SurfaceFlinger#04  pc 00016f5a  /system/lib/libbinder.so (android::IPCThreadState::executeCommand(int)+513)
D/CallStack(  101): SurfaceFlinger#05  pc 00017380  /system/lib/libbinder.so (android::IPCThreadState::joinThreadPool(bool)+183)
D/CallStack(  101): SurfaceFlinger#06  pc 00000820  /system/bin/surfaceflinger
D/CallStack(  101): SurfaceFlinger#07  pc 00000844  /system/bin/surfaceflinger
D/CallStack(  101): SurfaceFlinger#08  pc 0001271c  /system/lib/libc.so (__libc_init+35)
D/CallStack(  101): SurfaceFlinger#00  pc 00028a84  /system/lib/libsurfaceflinger.so (android::SurfaceFlinger::captureScreen(android::sp<android::IBinder> const&, android::sp<android::IMemoryHeap>*, unsigned int*, unsigned int*, int*, unsigned int, unsigned int, unsigned int, unsigned int)+63)
</pre>

