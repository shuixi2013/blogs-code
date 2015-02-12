title: 工作小笔记——Android 多用户下的要注意的问题
date: 2015-01-31 10:12:16
updated: 2015-01-31 10:12:16
categories: [Android Framework]
tags: [android]
---

之前有篇小笔记也遇到了类似的问题。就是如果你的系统要使用多用户功能，自己写的一些东西（或是以前写的）就要注意用户问题， android 增加了很多带 userID 的参数的接口，如果发起调用的用户和当前的用户不配对的话，要么崩溃，要么抛权限异常。

不知道从什么时候开始 core/java/android/app/ 下有个 AppGlobals， 要取啥 pm 之类的用这个比较好，然后注意启动一些应用、发啥广播、通知的，都要加上当前用户，当然如果你的应用有 `INTERACT_ACROSS_USERS（INTERACT_ACROSS_USERS_FULL）` 权限就当我没说（这个权限好像只有系统应用可以获取 ... ...）。


