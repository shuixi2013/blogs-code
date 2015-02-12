title: 64位 ubuntu adb No such file or directory 解决办法
date: 2015-01-25 22:42:16
updated: 2015-01-29 22:42:16
categories: [Android Development]
tags: [android, linux]
---

## 13.04

很简单，安装一个 ia32-libs 就好了（sudo apt-get install ia32-libs）。一看就知道64位系统又悲剧了一次。 

## 13.10

13.10 把 ia32-libs 这个包禁止掉了，它会提用下面几个包来代替：
* lib32z1
* lib32ncurses5
* lib32bz2-1.0

依次安装好之后，又提示 libstdc++.so.6 找不到， 这个是 32bit 的 gun c++ 的库。安装 lib32stdc++6 就可以了。

最后发现还是要把 ia32-libs 这个包装上比较好，不好很多软件用不了，例如 skype、Beyond Compare。 13.10 只是把这个源去掉了而已，可以自己加上：

在设置的 software update 的软件源那里加上： deb http://archive.ubuntu.com/ubuntu/ raring main restricted universe multiverse， 然后 apt-get update 一下就可以安装了。

从官方论坛那看到的信息，说是 ia32-libs 其实在 64bit 下是个不好的东西，所以从 12.04 之后就把这个源给去掉了（话说以前我 13.04 怎么装上去的），如果要使用 32bit 的软件包，应该使用 sudo apt-get install package-name:i386 来安装 32bit 的版本。


