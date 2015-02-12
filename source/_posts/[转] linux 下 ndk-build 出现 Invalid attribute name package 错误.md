title: (转) linux 下 ndk-build 出现 Invalid attribute name package 错误
date: 2015-01-25 22:47:16
updated: 2015-01-25 22:47:16
categories: [Android Development]
tags: [android]
---

在 ubuntu 上 ndk 编译时遇到以下错误： Invalid attribute name: package non-numeric second argument to `wordlist' function: ''. Stop.

原因：项目是从 Windows 制过来的， 所以 The AndroidManifest.xml file had Windows carriage control (\r\n) which was messing up the ndk-gdb script.

解决：用 vim 打开 AndroidManifest.xml，在命令模式下输入"set filetype=unix"，保存退出即可。或其把所有的 ^M 替换掉也可以。

