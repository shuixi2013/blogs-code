title: Android中 Error generating final archive Debug Certificate expired on xx 的错误
date: 2015-01-25 22:49:16
categories: [Android Development]
tags: [android]
---

## 问题概述：

在导入一个app后提示如下错误：

“Error generating final archive: Debug Certificate expired on 10/09/18 16:30”

# 原因分析：

android要求所有的程序必须有签名，否则就不会安装该程序。在我们开发过程中，adt使用debug keystore，在 preference->android->buid中设置。debug的keystore默认有效期为一年，如果你是从一年前开始完android程序，那么在一年后导入这个app的时候很可能出现debug keystore过期，导致你无法生成 apk文件。

此时你只要删除debug keystore就行，系统又会为你生成有效期为一年的私钥。 

## 解决方法：

进入C:\Documents and Settings\Administrator\.android 删除路径下的debug.keystore及 ddms.cfg。

（不同环境下的目录可能略有不同，可在eclipse中查找此路径：Window->Preferences->Android->Build下 Default debug keystore）

然后重新打开 eclipse 就OK了。


