title: Android MiniGUI Recovery 笔记
date: 2015-01-27 23:26:16
categories: [Android Framework]
tags: [android, minigui]
---

用 MiniGUI 整的 recovery UI。

## 关于 recovery 模式下的调试
我们常说的 recovery 模式其实就是没有启动 android 框架的 linux（可能还少了一些服务）。在这个模式下仍然可以启动 adb 服务进行调试。由于 recovery 的分区比较小，所以在调试的时候，可以把应用程序放到别的分区去跑。例如 /data 分区。

## ial
通过 zte blade 的实验和 cm 的代码，发现 /dev/input/event0 是按键设备文件，/dev/input/event3 是触屏设备文件。应该从这个设备读出触屏数据转化成 ial 层的坐标就行了吧。

### 2011.12.28
更正下之前说。 在 recovery 模式下的输入驱动。android 应该是启了一个线程在等待 /dev/input 下的一组设备的输入事件(event0, event1, event2 等等，最大从 cm 的代码里来看有个宏定义是16个，相关代码在 /android/bootable/recovery/minui/events.c)。使用 poll() 函数就能等待一组设备。使用的好像是 linux 的标准输入事件（用的是 <linux/input.h> 里定义的接口）。可以照搬 events.c 里代码，写个简单的 main 函数的程序打印出打开的设备路径、名字和相应得到的数据出来看看。在正常 android 启动的环境下。可以使用在 adb shell 下用 getevent 来看看正常启动 android 机子所有的输入设备情况。

哎，后面没下文了，腰斩了 ... ...

