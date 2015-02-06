title: Android Notification 分析——原理
date: 2015-02-06 10:17:16
categories: [Android Framework]
tags: [android]
---

Notification 作为 android 系统中的一种提醒类的 UI，其实不是很复杂。但是过了些时候又有点忘记了，又要去翻代码，所以还是稍微总结记录一下比较好。这里不会说得太深，就是一些 Notification 比较基本的东西（浅析一下吧），
我们先照例把相关代码位置啰嗦一下（4.2.2）：

```bash
# AM 广播相关代码
frameworks/base/services/java/com/android/server/NotificationManagerService.java
frameworks/base/services/java/com/android/server/am/BroadcastQueue.java
frameworks/base/services/java/com/android/server/am/BroadcastRecord.java
```

## UI 表现

目前 android 系统中的通知 UI 表现有3种：

* **Ticker**
就是刚发送一条通知的时候，通知栏上会有 一个小图标 + 一条文本 伴随一个滚动动画滚动出来，过一小段时间（3s）后会消失。作用是用一个动态效果告诉用户有一条通知来了。

* **StatusBar Icon**
当 Ticker 消失后，状态栏的左上角就会多一个小图标（和刚刚 Ticker 的小图标一样），告诉用户已经有一条通知了。

* **Notification Row**
发送一条通知后，下拉通知面板中的通知列表中会多一条通知行。上面那2个相当于是预览信息，这里的就是通知的完整信息了（不过话说本来通知就是应用程序的预览信息）。


所以说应用要对系统发送一条通知，显示上可以设置的就是这3大块，还有一个可以设置的是不可见的部分——PendingIntent：点击通知要完成的动作，一般是启动某个程序的 Activity 界面（还可以设置一个通知被删除时候的动作）。

通知分为2种：
1. 可删除的：这种可以在通知面板上由用户用手动删除，删除后，状态栏上对应的这条通知的小图标也会被删除掉（一般的通知类通知可以用这种）。
2. 不可删除的：这种通知用户无法手动删除，只能由发送通知的程序用代码取消掉（一些工具类的挂在通知面板上的快捷方式可以用这种，例如音乐播放器）。当然这种方式可能会被一些流氓应用利用，长按这条通知，然后可以查看到发送这条通知的应用（系统设置中），强制终止掉发送通知的 apk（在系统设置中强制终止 apk 或是进程自己挂掉了，发送的通知会被清楚掉）。当然还可以更狠点，直接禁止这个 apk 发送通知。

上面那2种类型可以通过设置 Notification 的 flags `FLAG_NO_CLEAR` 来表示。同时还有一个容易和这个搞混的标志叫： `FLAG_ONGOING_EVENT` 这个表示发送这条通知的应用正在进行某些工作。设置了这个正在进行的标志，效果和设置不可删除的是一样的，用户同样无法手动去删除这条通知。在老的 android 版本（好像应该是 4.0 以前），通知面板上显示通知是分2组显示的，一组叫“正在进行的”，一组叫“通知”。设置了 `FLAG_ONGOING_EVENT` 会被分到“正在进行的”那组，其他的在“通知”组。不过从 4.0 开始，android 改了设计了，全都归到一组去显示了。所以 4.0 以后，`FLAG_ONGOING_EVENT` 从视觉上看不出和没设置的有啥区别的（当然如果你自己定制的 framework 你可以继续改回分组的显示方式）。

## 通知标示

应用 new 一个 Notification 之后，获取 NotificationManager(NM) 之后，调用相应的接口就可以发送通知：

```java
// ================== NotificationManager.java =====================

    /*
     * Post a notification to be shown in the status bar. If a notification with
     * the same id has already been posted by your application and has not yet been canceled, it
     * will be replaced by the updated information.
     *
     * @param id An identifier for this notification unique within your
     *        application.
     * @param notification A {@link Notification} object describing what to show the user. Must not
     *        be null.
     */
    public void notify(int id, Notification notification) {
        notify(null, id, notification);
    }
    
    /*
     * Post a notification to be shown in the status bar. If a notification with
     * the same tag and id has already been posted by your application and has not yet been
     * canceled, it will be replaced by the updated information.
     * 
     * @param tag A string identifier for this notification.  May be {@code null}.
     * @param id An identifier for this notification.  The pair (tag, id) must be unique
     *        within your application.
     * @param notification A {@link Notification} object describing what to
     *        show the user. Must not be null.
     */
    public void notify(String tag, int id, Notification notification) {
        int[] idOut = new int[1];
        INotificationManager service = getService();
        String pkg = mContext.getPackageName();
        if (notification.sound != null) {
            notification.sound = notification.sound.getCanonicalUri();
        }
        if (localLOGV) Slog.v(TAG, pkg + ": notify(" + id + ", " + notification + ")");
        try {
            service.enqueueNotificationWithTag(pkg, tag, id, notification, idOut,
                    user.getIdentifier());
            if (id != idOut[0]) {
                Slog.w(TAG, "notify: id corrupted: sent " + id + ", got back " + idOut[0]);
            }
        } catch (RemoteException e) {
        }
    }

    /*
     * Cancel a previously shown notification.  If it's transient, the view
     * will be hidden.  If it's persistent, it will be removed from the status
     * bar.
     */
    public void cancel(int id) {
        cancel(null, id);
    }

    /*
     * Cancel a previously shown notification.  If it's transient, the view
     * will be hidden.  If it's persistent, it will be removed from the status
     * bar.
     */
    public void cancel(String tag, int id) {
        INotificationManager service = getService();
        String pkg = mContext.getPackageName();
        if (localLOGV) Slog.v(TAG, pkg + ": cancel(" + id + ")");
        try {
            service.cancelNotificationWithTag(pkg, tag, id, UserHandle.myUserId());
        } catch (RemoteException e) {
        }
    }

    /*
     * Cancel all previously shown notifications. See {@link #cancel} for the
     * detailed behavior.
     */
    public void cancelAll() {
        INotificationManager service = getService();
        String pkg = mContext.getPackageName();
        if (localLOGV) Slog.v(TAG, pkg + ": cancelAll()");
        try {
            service.cancelAllNotifications(pkg, UserHandle.myUserId());
        } catch (RemoteException e) {
        }      
    }
```

NMS 中有几种方式可以用来标示一条通知。首先说说为什么要标示一条通知。因为发送一条通知，之后可以更新它的状态（显示的信息），取消它（删除），所以要有一个东西能唯一表示一条通知。首先有一个原则，那就是除了系统以外，**一个应用只能操作自己发送的通知**。所以对于普通应用，虽然说有几种标示，但是都在 pkg（包名）这个大的标示下（其实还有一个大的标示，就是用户，不过我们这里先不讨论多用户）。所以看 NM 接口的实现，都自动把调用接口的应用的 pkg 发给 NMS，不让应用自己设置（免得有些应用故意写别人的包名，干坏事）。

然后我们看到接口，有 id（int） 和 tag（String） 这2种标示。一般应用开发中 id 用得比较多，然后我们看下 NMS 中识别的代码：

```java
// ================== NotificationManagerService.java =====================

    // lock on mNotificationList
    private int indexOfNotificationLocked(String pkg, String tag, int id, int userId)
    {
        ArrayList<NotificationRecord> list = mNotificationList;
        final int len = list.size();
        for (int i=0; i<len; i++) {
            NotificationRecord r = list.get(i);
            if (!notificationMatchesUserId(r, userId) || r.id != id) {
                continue;
            }    
            if (tag == null) {
                if (r.tag != null) {
                    continue;
                }    
            } else {
                if (!tag.equals(r.tag)) {
                    continue;
                }    
            }    
            if (r.pkg.equals(pkg)) {
                return i;
            }    
        }    
        return -1;
    }
```

发现优先匹配 id，然后才是 tag（最后包名那来个保险，确保你只能操作自己发的通知）。所以说要自己设计通知的 id 话，可以随意点，因为系统帮你确保你的 id 只在你的 pkg 中有效了，android 系统里面是直接拿图标的 R 资源的 id 来做 id 用的。也有一些用 tag，不过那个用得少。

## Content

接下来介绍下通知的显示内容。通知的 Content 一般来说是这样的：

![normal view 模式]("http://7u2hy4.com1.z0.glb.clouddn.com/android/Notification-base/normal_notification_callouts.png" "normal view 模式")

从 4.0 开始多了一个 big view 的模式，是这样的：

![big view 模式]("http://7u2hy4.com1.z0.glb.clouddn.com/android/Notification-base/bigpicture_notification_callouts.png" "big view 模式")


前面看了通知的接口。应用需要构造出一个 Notification 对象出来。在 2.x 的时候是直接设置 Notification 的字段的（里面的字段都是 public 的），后面 android 本着封装的设计感觉原来的不太好，所以搞了一个 Notification.Builder 出来，之后建议开发者用这个 Builder 来构造 Notification。其实2个原理上基本上是一样的，但是某些地方有点点不一样。就算是新的 sdk，你仍然可以坚持直接设置 Notification。所以我们来分开说一下：

### 直接设置 Notification

我们来列一下 Notification 中我们需要设置的一些数据。

* **icon(int)**
通知图标的资源 id。是上面 normal 模式的 2。

* **iconLevel(int)**
上面那个 icon 可以设成 LevelListDrawable 的，然后可以通过设置 iconLevel 来改变 icon 的图标 level，如果定时改变的话，可以形成一些下载通知图标的动态效果。

* ****

* **title()**

* ****

### 使用 Notification.Builder

## 系统处理流程 

未完待续 ... ...


