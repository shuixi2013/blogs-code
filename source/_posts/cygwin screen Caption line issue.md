title: cygwin screen Caption line issue
date: 2015-01-31 11:50:16
updated: 2015-01-31 11:50:16
categories: [Window]
tags: [window]
---

win8.1 64bit cygwin screen 4.1.0 在 screenrc 中设置：caption always "%{= dd}%-w%{+bu}%n %t%{-}%+w" （就是 screen session 的标题一直显示） 会造成当显示内容超过一屏会在最后一行（就是显示标题那一行）重复覆盖显示。 google 了好久，在这发现一个老外的解决办法：[screen output issue](http://stackoverflow.com/questions/18881116/gnu-screen-output-that-causes-the-screen-to-scroll-leaves-garbage-at-the-bottom "screen output issue")  。 说是要在 cygwin 中装 xterm，我把 cygwin xterm 的包全装上了，但是还是有这个问题。不过后面我右键 cygwin 窗口，调出 option --> terminal --> Type 那里改成 xterm-vt220 就好了。

PS：一开始好像默认是 xterm 的，我不知道装不装 xterm 包会不会出 xterm-vt220 的选项，terminal type 那还有好几个别的选项，我不太清楚其它的有什么区别，反正是又能正常用了。64bit win8.1 的 cygwin 问题咋就这么多咧，我以前 win7 用得好好的，蛋蛋疼了。


