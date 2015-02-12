title: 开启 Android SDK 所有的 API 的方法
date: 2015-01-27 23:07:16
updated: 2015-01-27 23:07:16
categories: [Android Framework]
tags: [android]
---

android 有很多类的 public 的接口被 google 给屏蔽了，一般做一些高级的操作或者是优化的话，需要访问这些接口或是成员变量。这就需要自己做的小手脚。

## 编译所有公开的 api 的 jar 包

下载 android 源码。然后你可以编译你想要的所有公开 api 的模块。例如说 framework/base/core/java/android/app 下面的一些类的 api（ActivityManager 等等）。这个时候你需要去 frame/base/ 敲 mm 编译 framework.jar 。但是你要注意，你需要的不是最后在 out/target/product 下最终生成的 framework.jar ，那个是经过 google 屏蔽后的 jar。你需要的是编译时的中间文件。注意看编译时的输出，一般是在 out/target/common/obj/JAVA_LIBRARIES/framework_intermediates 下的 classes.jar 的。你可以对比下这 2 个 jar 的大小就知道了。如果你需要别的模块的话，方法是类似的。

## 导入到 eclipse

得到有所有公开 api 的 jar 包后，你需要把这个 jar 导入到你的 eclipse 中，然后你的 android 程序才能使用所有公开的 api。

###  首先是导入到 eclipse 的环境中： 
* eclipse -> project -> properties -> Java Build Path -> Libraries -> Add Library -> User Library：点击 new 

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/use-hide-api/1.png)

* 然后编辑刚刚新创建的 library：点击 Add Jars -> 把之前你编译出来的 jar 添加进来

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/use-hide-api/2.png)

* 另外你还可以在 jar 中挂载源代码，这样调试的时候会更加的方便。但是一般来说你也可以不用挂载（在 source  attachment 那）

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/use-hide-api/3.png)

### 然后是在你 android 工程中添加上一步创建的 user library：
* 右键工程 -> properties -> Java Build Path -> Libraries -> Add Library -> User Library：然后点 next，选择上一步创建的 user library

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/use-hide-api/4.png)

* 在 Java Build Path -> Order and Export：把自己的添加的 user library 提升到 google 官方的 android 库的前面（点击 Up 按钮）

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/use-hide-api/5.png)

## 最后一步

然后你就可以在你的 android 程序里使用所有的 public 接口了，然后编译吧。 o(∩∩)o

