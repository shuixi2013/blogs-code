title: Android 4.0 访问 WebService 出现 android.os.NetworkOnMainThreadException 异常
date: 2015-01-25 23:30:16
tags: [android]
---

在开发涉及 WebService 的 Android 程序是出现了个很烦恼的错误 android.os.NetworkOnMainThreadException，找了很久才找到解决方案，可能在 Android 3.0 以上的版本都有这个问题，貌似他们在3.0以上的版本网络上做了更加严格的限制，更多的查询API上的StrictMode 。这个是由于在主线程中访问 Web API 很有可能会阻塞 UI 线程，所以 Android 就禁止你在主线程里访问 Web API 了。

## 方法一：

在访问前调用如下代码：

```java
public void onCreate(){
    StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder()
        .detectDiskReads()
        .detectDiskWrites()
        .detectNetwork()   // or .detectAll() for all detectable problems
        .penaltyLog()
        .build());

    StrictMode.setVmPolicy(new StrictMode.VmPolicy.Builder()
        .detectLeakedSqlLiteObjects()
        .detectLeakedClosableObjects()
        .penaltyLog()
        .penaltyDeath()
        .build());
}
```

## 方法二：

不要在主线程里访问 Web API，开个线程来访问（例如 new Thread 或者 AsyncTask 之类的）。这种方法也是 Android 推荐的方法。


 


