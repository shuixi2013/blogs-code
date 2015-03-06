title: 编译 Android 源码问题总结
date: 2015-01-27 23:28:16
updated: 2015-01-27 23:28:16
categories: [Android Framework]
tags: [android]
---

## 如何在 Android.mk 中添加自定义库
在编译 android 模块的时候，有时候想自己添加一些自己的东西，或者是说链接一些额外的库（例如说自己写的）。这个时候需要去修改 Android.mk 文件。这里以 android 中简单的 toolbox 模块为例说明。例如说要在 toolbox 中加一个自己的命令，除了编写相应的 .c 文件文件外（例如 myhello.c），Android.mk 中要这么改：

```cpp
// myhello 是你自己加的命令
TOOLS := \
    myhello \
    ls \
    mount \
    cat \
    ps \
    kill \
    ln \
    insmod \
    rmmod \
    lsmod \
... ...

// 默认的是只有 libcutils 和 libc 的，后面的那个 libmyhello 假设就是你要链接的库（你自己写的那个）
// 动态库是加到 LOCAL_SHARED_LIBRARIES 这个变量，静态库是加到 LOCAL_STATIC_LIBRARIES 这个变量
// 动态库还需要把你的库复制到 obj/lib 下（弄个连接也可以，运行的时候要把你的库弄到 system/lib 下 ）
// 动态库和静态库是不一样的，具体的可以通过出错信息来看（mm showcommands）
// 然后后面那个 LOCAL_LDFLAGS 可以用来设置一些额外链接信息
// 像这里我链接了 x11 的库，不过一般在 android 里是用不到这个库的
LOCAL_SHARED_LIBRARIES := libcutils libc libmyhello
OLD_LDFLAGS := $(LOCAL_LDFLAGS)
LOCAL_LDFLAGS = $(OLD_LDFLAGS) -L/usr/lib -lX11

... ...
```

## 编译 framework 自动增加自定义资源
在定制 framework 的一些模块的时候（例如 framework-res、SystemUI、Launcher 等），有些时候需要增加一些自己的资源（图片、xml 等）。官方预留了一个路径： device/xx(oem)/xx(product)/overlay 。在这个下面可以覆盖一些原有的资源文件（官方的原意应该是主要给 OEM 改一些 config 的）。我们可以把自己的新增的资源文件加如到自己的模块对应的路径下。不过对于新增的资源，需要在 xml 中声明：。

<pre config="brush:bash;toolbar:false;">
# type 是资源类型（integer 等），name 就是资源的名字了
add-resource type="xx" name="xx"
</pre>

不过每一个资源都要加这个声明确实比较烦，可以在模块的 Android.mk 文件中增加了一个标志，让系统在编译的时候帮你自动添加这个 add-resource。增加这个标志：

<pre config="brush:bash;toolbar:false;">
LOCAL_AAPT_FLAGS := --auto-add-overlay
</pre>


