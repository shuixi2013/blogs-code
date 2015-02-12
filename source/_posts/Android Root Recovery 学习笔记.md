title: Android Root Recovery 学习笔记
date: 2015-01-27 23:21:16
updated: 2015-01-27 23:21:16
categories: [Android Framework]
tags: [android]
---

喜欢折腾 android 的继续往下看吧。

## Root

首先先说下 android 获取 root 权限的原理。android 是基于 linux 系统的，所以 android 获取 root 权限就是差一个 su 命令，当然为了更好的管理 root 权限还差一个管理权限的应用层软件 superuser.apk。那获取 root 权限就是把 su 放到 /system/bin 下和把 sueruser.apk 放到 /system/app 下就行了。但是 android 的 system 分区是只读的。目前的破解方法都是根据系统的漏洞（adb 守护进程，这个进程自己有 root 权限），来临时获取 root 权限，然后把 system 分区挂载成可读写的，然后把那2个东西放到相应的目录就行了。

## android 的系统漏洞

这里我就帖几个网站吧，有人已经分析过了。目前来说主要就是 psneuter, zergRush, RageAgainstTheCage? 这几个小程序。其中有几个漏洞在高的 android 版本中已经被修复了，不过我相信迟早还会有新的漏洞被发现的。顺带说一句：如果你是自己做 rom 的话，就不用这么麻烦的了，直接集成进去就行了。

* [Android adb setuid提权漏洞的分析](http://blog.claudxiao.net/2011/04/android-adb-setuid/ "Android adb setuid提权漏洞的分析")
* [Android提权代码zergRush分析](http://blog.claudxiao.net/2011/10/zergrush/ "Android提权代码zergRush分析")
* [一键获取android手机 root权限](http://blog.csdn.net/AndyTsui/article/details/6535085 "一键获取android手机 root权限")

## 手动 root 你的手机

### 工具

* 你手机的 usb 驱动，有些手机不需要，有些需要，自己去官网或者开发网站找吧。
* adb 这个不用说了，android sdk 里就有。
* 临时获取 root 权限的工具（就是上面说的那几个利用 android 系统漏洞的程序）。
* busybox, su, superuser.apk 。 这里有下载，可能不是最新的但是基本上能用。

### 步骤

```shell
adb push busybox /data/local/
adb push psneuter /data/local/
adb push su /data/local/
adb shell chmod 777 /data/local/busybox
adb shell chmod 777 /data/local/psneuter
adb shell
/data/local/psneuter
adb shell
mount -o remount,rw -t ext3 /dev/block/mmcblk0p25 /system
mkdir /system/xbin
/data/local/busybox cp /data/local/su /system/xbin/su
chown 0:0 /system/xbin/su
chmod 6755 /system/xbin/su
ln -s /system/xbin/su /system/bin/su
exit
adb push Superuser.apk /system/app/Superuser.apk
```

注意：上面那个 /dev/block/mmcblk0p25 是指手机的 system 分区，不同的手机会不同。这个可以用 cat /proc/partitions 和 cat /proc/mounts 来判断自己手机的 system 分区是哪个一个。都弄好后就可以重启手机了。网上有个 superoneclick 的东西，其实就是把这些东西了自动化了而已，也可以自己写脚本。

## Recovery

暂无 ... ...


