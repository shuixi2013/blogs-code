title: ubuntu 使用问题总结
date: 2015-01-19 09:54:16
tags: [linux]
---

## 激活 root 账号

默认 root 账号是不能切换和登陆的，只能用 sudo 来了临时获取一个命令的 root 执行权限。有些时候为了方便可以激活 root

账号。 sudo passwd root ，设置一个密码就可以了。

## 中文输入安装之后无法调出

安装了 ibus 无法调出，一般是 ibus demon 没启动，然后输入法也没设置为中文。ibus-setup 可以调出输入法设置，然后把

中文输入法设置好。不过要想个办法让 ibus demon 自动启动（没启动的，ibus-setup 守护进程就启动了）。可以装一个 im-swtich 然后设置默认输入法为 ibus

