title: Android 命令备忘
date: 2015-01-25 22:03:16
updated: 2015-01-25 22:03:16
categories: [Android Development]
tags: [android]
---

新手问题多多 -_-|| 

## adb logcat
adb logcat -v time： 可以显示每一条 log 的时间戳。

## mount
如果 adbd 是 root 运行的，直接 adb remount 就可以挂载 system 为可读写的。如果 adbd 不是 root 的，shell 进入先 su（系统要 root 之后），然后需要自己手动 mount system 分区：

```bash
# 先敲 mount 可以查看系统上已经挂载的分区信息，可以找到 system 的分区
$ mount

# 例如像下面这样的
tmpfs /dev tmpfs rw,nosuid,relatime,mode=755 0 0
devpts /dev/pts devpts rw,relatime,mode=600 0 0
none /dev/cpuctl cgroup rw,relatime,cpu 0 0
proc /proc proc rw,relatime 0 0
sysfs /sys sysfs rw,relatime 0 0
/sys/kernel/debug /sys/kernel/debug debugfs rw,relatime 0 0
none /acct cgroup rw,relatime,cpuacct 0 0
tmpfs /mnt/secure tmpfs rw,relatime,mode=700 0 0
tmpfs /mnt/asec tmpfs rw,relatime,mode=755,gid=1000 0 0
tmpfs /mnt/obb tmpfs rw,relatime,mode=755,gid=1000 0 0
/dev/block/platform/emmc/by-name/system /system ext4 ro,relatime,barrier=1,data=ordered 0 0

# 就可以这么敲了
$ mount -r -o remount,rw /dev/block/platform/emmc/by-name/system /system
```

## am (activity manager)
adb shell 进入 android 的 console 输入的命令 
am start -n package/activity 可以用来启动一个 activity 例如：
am start -n com.eebbk.mingming.k7ui.demo/.K7UIDemo

am startservice package/service 可以启动一个 service 例如：
am startservice com.android.systemui/.SystemUIService 

## pm (package manager)
adb shell 进入 android 的 console 输入的命令
pm list packages: 可以显示本地已经安装的 app（包名）

## getprop (setprop)
adb shell 进入 android 的 console 输入的命令
获取（打印）系统当前保存的 property

setprop 是设置 property： setprop key value

## /system/bin/busybox
adb shell 进入 android 的 console 输入 /system/bin/busybox xx 
这个基本要是在 eng 模式才会有这个东西，当然 root 之后可以自己放 busybox 进去。 busybox 不解释了，有了这个玩意，基本啥 linux 命令都有了。

## dmesg
adb shell，打印最近 kernel 的 log。

## top -t(ps -t)
adb shell， top 不带参数是显示当前的进程运行的情况，如果 top -t 则显示每个进程中的线程运行的情况。这个可以用来查看 native 程序线程的情况，java 层的用 DDMS 的带的线程查看工具比较方便。ps -t 也可以，这个就和 DDMS 里面看线程一样了。

## dumpsys
adb shell， dump 系统服务的调试信息，可以接参数（服务的名字，例如 "activity"，这个名字是在 ServiceManager 中注册的名字），不接的话，则是 dump 所有服务的信息。

