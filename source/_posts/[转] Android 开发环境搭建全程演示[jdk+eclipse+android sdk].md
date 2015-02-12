title: (转) Android 开发环境搭建全程演示（jdk+eclipse+android sdk）
date: 2015-01-25 22:08:16
updated: 2015-01-25 22:08:16
categories: [Android Development]
tags: [android, install]
---

## 一 相关下载

* (1) java JDK下载:

进入该网页: [下载页面](http://java.sun.com/javase/downloads/index.jsp "下载页面") (或者直接点击下载)，如下图:
选择 Download JDK 只下载JDK，无需下载jre.

图1（挂鸟）

* (2)eclipse下载

进入该网页: [下载页面](http://www.eclipse.org/downloads/ "下载页面") ， 如下图:

图2（挂鸟）

我们选择第一个(即eclipse IDE for java EE Developers)

* (3)下载Android SDK

说明: Android SDK两种下载版本，一种是包含具体版本的SDK的，一种是只有升级工具，而不包含具体的SDK版本，后一种大概20多M，前一种70多M。[完全版下载](https://dl-ssl.google.com/android/repository/android-2.1_r01-windows.zip "完全版下载") (android sdk 2.1 r01)，[升级版下载](http://dl.google.com/android/android-sdk_r04-windows.zip "升级版下载") (建议使用这个，本例子就是使用这个这里面不包含具体版本，想要什么版本在Eclipse里面升级就行)

## 二 软件安装

1. 安装jdk 6u19，安装完成即可，无需配置环境变量
2. 解压eclipse，eclipse无需安装，解压后，直接打开就行
3. 解压android sdk，这个也无需安装，解压后供后面使用
4. 最终有三个文件夹，如下图:

图3（挂鸟）

## 三 Eclipse配置

## 1 安装android 开发插件

打开Eclipse, 在菜单栏上选择 help->Install New SoftWare 出现如下界面:

图4（挂鸟）

点击 Add按钮,出现如下界面

图5（挂鸟）

输入网址: [](https://dl-ssl.google.com/android/eclipse/)    (如果出错，请将https改成http)，名称: Android (这里可以自定义)，点击OK，将出现如下界面

图6（挂鸟）

点击 Next按钮 ，出现如下界面:

图7（挂鸟）

点击Next按钮，出现如下界面:

图8（挂鸟）

选择 I accept the terms of the license agreements   点击Next,进入安装插件界面

图9（挂鸟）

安装完成后，出现如下界面
 
图10（挂鸟）

点击Yes按钮，重启Eclipse

### 2 配置android sdk

* (1)点击菜单window->preferences,进入如下界面

图11（挂鸟）

选择你的android SDK解压后的目录，选错了就会报错，这个是升级工具，目前还没有一个版本的SDK

* (2)升级SDK版本,选择菜单 window->Android sdk and avd manager 出现如下界面

图12（挂鸟）

选择update all按钮，出现如下界面

图13（挂鸟）

选择左边的某一项，点击accept表示安装，点击reject表示不安装，我这里只选了SDK 2.1 和samples for api 7 , 自己可以任意自定义，确定后，选择install按钮，进入安装界面如下:

图14（挂鸟）

安装完成如下:

图15（挂鸟）

* (3)新建AVD(android vitural device)    和上面一样，进入android sdk and avd manager,选中Vitural Devices 在点击New按钮

点击New按钮后，进入如下界面:

图16（挂鸟）

名称可以随便取，target选择你需要的SDK版本，SD卡大小自定义,点击 Create AVD,得到如下结果

图17（挂鸟）

如上显示创建AVD完毕
 
### 3 新建Android项目

* (1)选择菜单file->new->other 进入如下界面:

图18（挂鸟）

选择新建Android Project项目，点击Next按钮，进入如下界面

图19（挂鸟）

名称自定义，应用程序名自定义，报名必须包含一个点以上，min SDK version里面必须输入整数，点击Next出现如下界面:

图20（挂鸟）

注: 若有错误如: Project ... is missing required source folder: 'gen' ,则将gen->Android.Test->R.java这个文件删掉，Eclipse会为我们重新生成这个文件，并且不会报错。

* (3)配置运行

右键项目->Run as -> Run Configuration 进入如下界面:

图21（挂鸟）

该界面，点击Browse 按钮，选择你要运行的项目，选择Target切换到以下界面：

图22（挂鸟）

该界面选择运行的AVD，将AVD前面的方框设置为选择状态。

* (4)测试项目运行

右键项目名称->run as ->Android Application 即可启动运行该Android程序，如下所示:

图23（挂鸟）

正在进入：

图24（挂鸟）

测试程序运行结果

## 四 结束语

至此，android开发环境搭建完毕，有问题请留言。在这里要注意，我这里只是下载了android sdk r4升级工具，没有下载具体的SDK，而是通过在Eclipse里面的Android Sdk管理工具升级的，你也可以直接下载具体的SDK版本，如: Android sdk 2.1 r1 上面有这个的下载链接，但我任务用升级工具更好。


