title: Linux 下 adb usb 连接 usb 设备问题总结
date: 2015-01-25 22:39:16
updated: 2015-01-25 22:39:16
categories: [Android Development]
tags: [android, linux]
---

在 linux 上一般刚开始用 usb 数据线 adb 连接 android 设备会出现 "???????????? no permissions" 的提示。这个是因为要使用 usb 来调试需要 root 权限，使用一下的方法将使用 root 权限来使用 usb 设备。

## 修改 udev 配置

在 /etc/udev/rules.d/ 下新建一个文件：70-android.rules。其中前面的数字 70 可以是别的数字，但是名字开头数字大文件中记录的规则会覆盖名字开头数字小的文件中的规则，所以你需要尽可能设置的文件名大一些，一般 70 就够用了。然后编辑如下规则：

<pre>
SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", MODE="0666"
</pre>

运行命令，重启udev：

<pre config="brush:bash;toolbar:false;">
$sudo chmod a+rx /etc/udev/rules.d/70-android.rules
$sudo service udev restart
</pre>

## 添加设备 id 号

有些时候写了 udev 这个文件，还是连接不上，这个时候需要配置下设备的设备 id 号。在 home 目录下的 .android/adb_usb.ini（没有的话自己新建一个就行） 中加入设备 id 号（usb 连接设备后，可以用 lsusb 看到）。例如：

<pre config="brush:bash;toolbar:false;">
0x2207
0x04e8
</pre>

## 重新启动 adb

拔掉usb重新连上再执行：

<pre config="brush:bash;toolbar:false;">
adb kill-server

# 然后你就可以看到你的 android 设备了
adb devices

adb shell

# 你 android 设备 root 后，进入 shell 后还可以这样，root 后终端提示符号变成 # 则表示成功
su
</pre> 

## adb 网络连接

如果你有安装 Android SDK,应该会知道有一个 ADB 工具，这个工具可以在命令行下控制、调试你的Android 设备，这个工具不仅支持通过 USB 链接，而且可以通过 TCP/IP 来连接，也就是说不需要数据线，通过 wifi 就可以连接了。但是在默认情况下，是无法连接的。下面来讲怎么设置通过 wifi 来连接ADB。在菜场里找一个Android 的终端工具，我用的是 Terminal Emulator ，然后在终端里，依次输入

<pre>
setprop service.adb.tcp.port 5555
stop adbd
start adbd
</pre>

然后，在你的电脑（WIN/LINUX) 里命令行启动 adb，输入 adb connect your-phone-ip

