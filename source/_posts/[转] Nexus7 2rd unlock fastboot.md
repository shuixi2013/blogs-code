title: (转) Nexus7 2rd unlock fastboot
date: 2015-01-31 11:00:16
categories: [Android Framework]
tags: [android]
---

* 电脑上装 android sdk（主要是保证有 platform-tools 下的 adb 和 fastboot）。
* 装 usb 驱动，window 要，linux 下不需要，地址： [google usb driver](http://developer.android.com/sdk/win-usb.html "google usb driver")。
* 在 N7 上开启 adb，这个默认的系统把开发者选项隐藏起来了，在 系统设置 --> 关于 --> 系统版本号 那里多点几下就能开启开发者模式，里面可以开启 adb。
* adb reboot bootloader --> 当 n7 进入 bootloader 后，输入 fastboot oem unlock 然后用音量+、-选择 yes，按电源确定。

PS： unlock fastboot 会格掉机器中的所有数据，整之前记得把数据备份下。


