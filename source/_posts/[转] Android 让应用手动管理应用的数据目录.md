title: (转) Android 让应用手动管理应用的数据目录
date: 2015-01-26 22:58:16
updated: 2015-01-26 22:58:16
categories: [Android Development]
tags: [android]
---

在应用程序管理器点击软件显示的页面，我们可以点击清除数据按钮，这样所有关于该 app 的缓存在手机的数据都清除掉了。类似于新安装的一样。但是有时候，用户不想全部删除，比如登录信息等。就有需求如果应用能够手动管理应用的数据目录的需求，那么 android 系统支持这个功能吗？当然支持了，如图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/manage-app-data/1.png)

红框处，显示的叫管理空间，而不是我们常常见到的清除数据。当点击这个按钮能够跳转到我们的空间管理页面就做到了，那么如何实现呢？只需要在 AndroidManifest.xml 中的 application 标签添加一个 android:manageSpaceActivity 指定一个 Activity 来管理数据。实例如下：

```html
<application 
    android:manageSpaceActivity="com.test.ManageSpaceActivity"> 
</application> 
```

ManageSpaceActivity 当然也要在 AndroidManifest.xml 声明为 activity 。综上所述，如果想自己管理数据目录，则可以使用 android:manageSpaceActivity 属性来控制，而不是默认的全部清除了 /data/data/packagename/ 里面的所有文件。当然我们还可以扩展，比如清除 SD 卡上的数据，如果拥有 root 权限，还可以用它当成垃圾清理。

[原始出处](http://blog.csdn.net/mingli198611/article/details/22671919 "原始出处")

