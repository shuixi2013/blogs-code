title: Android 设置广播标志启动还没启动程序
date: 2015-01-25 23:35:16
categories: [Android Development]
tags: [android]
---

在 android 中可以通过 context.sendBroadcast(Intent intent) 发送广播，让监听了该广播的程序处理业务。其中 Intent 中有一个 flag：

```java
    /**
     * If set, this intent will always match any components in packages that
     * are currently stopped.  This is the default behavior when
     * {@link #FLAG_EXCLUDE_STOPPED_PACKAGES} is not set.  If both of these
     * flags are set, this one wins (it allows overriding of exclude for
     * places where the framework may automatically set the exclude flag).
     */
    public static final int FLAG_INCLUDE_STOPPED_PACKAGES = 0x00000020;
```

设置了这个 flag 之后，能够让还没启动的程序（如果这个程序在 AndroidManifest 里面静态注册了该广播的话）也能处理这个广播。当然相应的还有一个标志： FLAG_EXCLUDE_STOPPED_PACKAGES 就是相反的功能（好像默认就是这个）。


