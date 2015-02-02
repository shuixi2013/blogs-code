title: Android Broadcast 分析——超时或异常
date: 2015-01-22 10:17:16
categories: [Android Framework]
tags: [android]
---

上一篇（处理篇）最后留了一个问题：串行广播需要一个接着一个执行，如果其中某一个接收器处理的时候挂掉了，或者耗时很长还没处理完，是不是后面的接收器就无法处理了呢。我们在这里看一下 android 是不是已经想到这些了。我们先照例把相关代码位置啰嗦一下（4.2.2）：

```bash
# Content 广播相关的代码
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/LoadedApk.java
frameworks/base/core/java/android/app/ActivityThread.java

frameworks/base/core/java/android/content/Intent.java
frameworks/base/core/java/android/content/IntentFilter.java
frameworks/base/core/java/android/content/BroadcastReceiver.java
frameworks/base/core/java/android/content/IIntentReceiver.aidl

# 广播解析相关代码
frameworks/base/services/java/com/android/server/IntentResolver.java

# AM 广播相关代码
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/java/com/android/server/am/RecevierList.java
frameworks/base/services/java/com/android/server/am/BroadcastQueue.java
frameworks/base/services/java/com/android/server/am/BroadcastFilter.java
frameworks/base/services/java/com/android/server/am/BroadcastRecord.java

# PM 广播相关代码
frameworks/base/services/java/com/android/server/pm/PackageManagerService.java
```

## 超时处理



未完待续 ... ...

