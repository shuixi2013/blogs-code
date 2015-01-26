title: ubuntu virtual box 问题小结
date: 2015-01-19 10:14:16
categories: [Linux]
tags: [linux]
---

## usb
要在虚拟机中使用 usb 设置（烧录接口）： 

* 需要先安装 Extension Pack（注意版本要和 virtual box 的版本对应）。最好去虚拟机里把 guset addiations 也装了。

* 查看下 /etc/group ，看看自己的用户有没有在 vboxusers 这个组下面，如果没在的话，把自己的用户加到这个用户组下面： usermod -a -G group1 user1

* 在 Settings --> Usb --> Enable USB Controller (USB 2.0) 都勾上。

然后在虚拟机的 Usb devices 下面就可以勾选要使用的 usb 设置了。

