title: Android 程序签名问题
date: 2015-01-25 22:34:16
tags: [android]
---

## 一、多个开发环境具有相同的 debug 签名

在多台机器用 Eclipse 开发 Android 程序的时候，签名不一致导致要反反复复删除原程序才能安装、调试很不爽吧。其实让 Eclipse 用一样的 debug 签名就好了。方法是选中其中一个 Eclipse 自动生成的 debug 签名（我曾经试过了用自己的签名，Eclipse 的 ADT 不知道密码，而且也没地方自己输入密码，所以只好用 Eclipse 自己生成的 debug 签名了），然后把签名复制到其他 Eclipse 机子上（linux 下的签名是 ~/.android/debug.keystore，window 的是 C:\Users\Administrator\.android\debug.keystore）。然后在 Window -->  Preferences --> Andrid --> Build 中的 Custom debug keystore：选中从别的 Eclipse copy 过来的 keystore 就可以。然后你把自己的工程（例如 git 上的）重新 clean、build 一次，就可以直接装上调试了。On Yeah ~~ (≧∇≦)

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/keystore/1.jpeg)

## 二、生成自己签名时要注意的问题

用 Eclipse 或者 命令行（keytool）生成的教程网上很多。这里简单记录下就行了。用 Eclipse 的话，右键工程 --> Export --> Android --> Export Android Application ，然后选择一个 Android 工程，然后选择 Create new keystore，然后后面就填自己的一些个人信息。这里需要注意的问题是：**Alias 这里的名字要和前面 key 的名字一样，否则在签名的时候会报 keystore 的证书链不存在的问题的。然后这个 key 最好自己保管好。**

eclipse android debug key 的密码是 android，自己生成的 key 密码要和这个一样，否则无法使用。

要查看 keystore 信息可以用 keytool 来查看： 

keytool -list -v -keystore [enter keystore name] -storepass [enter keystore password]

