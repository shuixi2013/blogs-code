title: 工作小笔记——Android 多用户切换无法启动应用的问题
date: 2015-01-31 09:38:16
categories: [Android Framework]
tags: [android]
---

Android 好像是从 3.0 开始引入多用户相关的支持，到 4.4 的是已经比较成熟了，相关的功能都有了，例如独立的用户空间（4.4 之后，android 把 flash 模拟成几个分区 0、1 等，对应 用户0、用户1），不同用户安装不同应用隔离等等。

多用户功能可以关闭的，把系统中最大用户数配置成 1，就关闭了（在 framework-res 的  config_multiuserMaximumUsers 这个配置里面），目前不少手机 OEM rom 都是关闭多用户的，因为开启会浪费一些空间，而且国内手机 rom 基本上不使用这个功能。

最近在项目要用多用户的相关功能，遇到点问题。就是切换用户之后，发现一些公共的 activity 启动不起来了。android 多用户会有保护，如果这个 apk 不属于这个用户安装的无法启动的，就算用代码 startActivity 也无法启动，am 会判断启动的用户是否是当前的用户的（代码懒得贴了，我目前对这么还没怎么研究，不过大概是这么个意思）。

但是刚开始感觉这些公用的 apk 切换用户后应该能启动才对。后面跟了下代码，发现 startActivity 启动 activity 如果这个 apk 没还没 killed 掉，那么这个 activity 是属于最开始启动这个 apk 的用户的。例如说 SystemUI 里面的某个系统 activity，切换用户，SystemUI 没有被 killed 掉，所以不会重新启动，所属用户为系统默认的用户（机主），这个时候如果调用 startActivity 去启动 SystemUI 的 activity 的话，是启动不了的，因为使用的是默认用户，不是当前这个。

那要怎么办咧，android 早就想好了，用下面这个 api 就好了（以当前用户启动 activity ^_^）：

<pre config="brush:bash;toolbar:false;">
public void startActivityAsUser(Intent intent, UserHandle user)
</pre>

后面有个 UserHandle 的参数，话说以前接触 4.2 之后，发现很多 api 多了个 UserHandle， userID 的东西，刚开始不知道是干什么用的，现在明白了，android 为了多用户支持，大开刀啊。

然后要以当前用户启动的话，就这么用：

<pre config="brush:bash;toolbar:false;">
startActivityAsUser(intent, new UserHandle(UserHandle.USER_CURRENT));
</pre>


然后类似的问题还有发通知的时候，点击通知启动 activity，同理以前有 PendingIntent.getActivity 设置的 PendingIntent 如果是类似 SystemUI 这种公用的 activity，切换用户后也是无法启动的，改成带当前用户的发送就可以了：

<pre config="brush:bash;toolbar:false;">
PendingIntent pi = PendingIntent.getActivityAsUser(
	context, 0, intent, 0, null, UserHandle.CURRENT);
</pre>

不过目前多用户相关的接口都是 hide 的，所以普通应用不需要关注这些东西，但是如果改系统里面的东西就需要注意下这方面的问题哦。当然如果压根不用多用户的话，可以无视。


