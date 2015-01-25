title: Android sdk content loader 0 的解决方法
date: 2015-01-25 22:43:16
tags: [android]
---

有些时候打开 eclipse + adt 会出现 Android sdk content loader 一直卡在 0% 那里。这种时候是没办法编译、调试 apk 的。网上搜罗了几套办法，哎 google 搞开源的还是没 MS 的 VS 省心，但是 VS 又太庞大。

## 断网

断网重启 eclipse，这招对我最管用。估计 google 的 adt 又去他的官网检查啥更新去了，在天朝这不是蛋疼么。

## 删掉 .android

把 .android 文件夹删掉，重启 eclipse。其实也不用完全删掉 .android， debugkey 还是可以留着的。

## 删除 .metadata
删除 workspace 下面的 .metadata 文件夹，重启 eclipse 。


