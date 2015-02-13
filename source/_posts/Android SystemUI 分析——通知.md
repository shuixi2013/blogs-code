title: Android SystemUI 分析——通知
date: 2015-02-06 10:17:16
updated: 2015-02-09 09:15:16
categories: [Android Framework]
tags: [android]
---

Notification 作为 android 系统中的一种提醒类的 UI，其实不是很复杂。但是过了些时候又有点忘记了，又要去翻代码，所以还是稍微总结记录一下比较好。这里不会说得太深，就是一些 Notification 比较基本的东西（浅析一下吧），
我们先照例把相关代码位置啰嗦一下（4.2.2）：

```bash
# 应用接口相关代码
frameworks/base/core/java/android/app/Notification.java
frameworks/base/core/java/android/app/NotificationManager.java

# 系统内部接口相关代码
frameworks/base/core/java/com/android/internal/statusbar/StatusBarNotification.java
frameworks/base/core/java/com/android/internal/statusbar/StatusBarIcon.java

# System Server 相关代码
frameworks/base/services/java/com/android/server/NotificationManagerService.java
frameworks/base/services/java/com/android/server/StatusBarManagerService.java

# SystemUI 相关代码
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NotificationData.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/StatusBarIconView.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/BaseStatusBar.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/PhoneStatusBar.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/Ticker.java
```

## UI 表现

目前 android 系统中的通知 UI 表现有3种：

* **Ticker**
就是刚发送一条通知的时候，通知栏上会有 一个小图标 + 一条文本 伴随一个滚动动画滚动出来，过一小段时间（3s）后会消失。作用是用一个动态效果告诉用户有一条通知来了。

* **StatusBar Icon**
当 Ticker 消失后，状态栏的左上角就会多一个小图标（和刚刚 Ticker 的小图标一样），告诉用户已经有一条通知了。

* **Notification Row**
发送一条通知后，下拉通知面板中的通知列表中会多一条通知行。上面那2个相当于是预览信息，这里的就是通知的完整信息了（不过话说本来通知就是应用程序的预览信息）。


所以说应用要对系统发送一条通知，显示上可以设置的就是这3大块（效果图见后面的系统分析那里），还有一个可以设置的是不可见的部分——PendingIntent：点击通知要完成的动作，一般是启动某个程序的 Activity 界面（还可以设置一个通知被删除时候的动作）。

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

![normal view 模式](http://7u2hy4.com1.z0.glb.clouddn.com/android/SystemUI-notification/normal_notification_callouts.png "normal view 模式")

从 4.0 开始多了一个 big view 的模式，是这样的：

![big view 模式](http://7u2hy4.com1.z0.glb.clouddn.com/android/SystemUI-notification/bigpicture_notification_callouts.png "big view 模式")


前面看了通知的接口。应用需要构造出一个 Notification 对象出来。在 2.x 的时候是直接设置 Notification 的字段的（里面的字段都是 public 的），后面 android 本着封装的设计感觉原来的不太好，所以搞了一个 Notification.Builder 出来，之后建议开发者用这个 Builder 来构造 Notification。其实2个原理上基本上是一样的，但是某些地方有点点不一样。就算是新的 sdk，你仍然可以坚持直接设置 Notification。所以我们来分开说一下：

### 直接设置 Notification

我们来列一下 Notification 中我们需要设置的一些数据。

* **icon(int)**
通知图标的资源 id。是上面 normal 模式的 2。

* **iconLevel(int)**
上面那个 icon 可以设成 LevelListDrawable 的，然后可以通过设置 iconLevel 来改变 icon 的图标 level，如果定时改变的话，可以形成一些下载通知图标的动态效果。

* **setLatestEventInfo**
这是个接口，参数如下：

```java
// ============== Notification.java =====================

    /*
     * Sets the {@link #contentView} field to be a view with the standard "Latest Event"
     * layout.
     *
     * <p>Uses the {@link #icon} and {@link #when} fields to set the icon and time fields
     * in the view.</p>
     * @param context       The context for your application / activity.
     * @param contentTitle The title that goes in the expanded entry.
     * @param contentText  The text that goes in the expanded entry.
     * @param contentIntent The intent to launch when the user clicks the expanded notification.
     * If this is an activity, it must include the
     * {@link android.content.Intent#FLAG_ACTIVITY_NEW_TASK} flag, which requires
     * that you take care of task management as described in the
     * <a href="{@docRoot}guide/topics/fundamentals/tasks-and-back-stack.html">Tasks and Back
     * Stack</a> document.
     *
     * @deprecated Use {@link Builder} instead.
     */
    @Deprecated
    public void setLatestEventInfo(Context context,
            CharSequence contentTitle, CharSequence contentText, PendingIntent contentIntent) {
        // TODO: rewrite this to use Builder
        RemoteViews contentView = new RemoteViews(context.getPackageName(),
                R.layout.notification_template_base);
        if (this.icon != 0) { 
            contentView.setImageViewResource(R.id.icon, this.icon);
        }    
        if (priority < PRIORITY_LOW) {
            contentView.setInt(R.id.icon,
                    "setBackgroundResource", R.drawable.notification_template_icon_low_bg);
            contentView.setInt(R.id.status_bar_latest_event_content,
                    "setBackgroundResource", R.drawable.notification_bg_low);
        }    
        if (contentTitle != null) {
            contentView.setTextViewText(R.id.title, contentTitle);
        }    
        if (contentText != null) {
            contentView.setTextViewText(R.id.text, contentText);
        }
        if (this.when != 0) {
            contentView.setViewVisibility(R.id.time, View.VISIBLE);
            contentView.setLong(R.id.time, "setTime", when);
        }
        if (this.number != 0) {
            NumberFormat f = NumberFormat.getIntegerInstance();
            contentView.setTextViewText(R.id.info, f.format(this.number));
        }

        this.contentView = contentView;
        this.contentIntent = contentIntent;
    }
```

这个可以说以前（4.0 之前）通知的最主要的部分了。看这个函数里面的实现知道，系统帮我们提供了一个通知的模板（xml 在 frameworks/base/core/res/res 下面），帮我们 new 了一个 RemoteViews 来当作通知的 ContentView。一个通知必须要有的2个元素，一个是 id，另外一个就是 contentView。这个 RemoteViews 是跨进程的 UI 组件，这个东西以后再分析，这里先不管它。系统的模板弄出来的样子就和上面的 normal view 那样。这里参数可以让你设置通知的标题（title）和内容（text）。分别对应 normal view 的 1 和 3。

这里除了 UI 元素设置，最后还有一个 PendingIntent 的参数（contentIntent），算是功能性设置。注释中说是用户点击通知的时候对应完成的功能。可以是 activity（点击时 launch），也可以是 broadcast（点击时发送）。可以说还是挺灵活的，这里就不展开分析了。

* **number(int)**
标准模板中的右下角显示一个小数字（一些消息、邮件类的通知很有用），对应 normal view 的 4。注意如果不想显示把这个设置 0 就可以了（默认是 0，上面 setLatestEventInfo 处理那里代码能看得出的）。

* **when(long)**
标志模板中右上角显示发送通知的时间，对应 normal view 的 6。看到类型是 long 的就知道这个时间是自从 1970-01-01 00:00:00.0 的 UTC，可以通过 System.currentTimeMillis() 取当前的时间，当然你也可以故意设置成别的时间。顺带一提，如果你不想显示时间，把这个设置成 0 就行了（和 number 一样的）。

* **contentView(RemoteViews)**
其实上面 setLatestEventInfo 是让系统帮你生成一个标准的通知模板。但是你会发现一些音乐播放器和一些手机助手，在通知栏上有一些比较高级的通知，这些是通过自定义的 content view 来实现的。不要用系统的模板，自己写 content view 就行了，这个是 RemoteViews，和写 widget 差不多的。

* **bigContentView(RemoteViews)**
这个是 Notification 新增的 big view 模式，和 contentView 一样，可以自定义，详见下面 Builder 的说明。

* **deleteIntent**
上面那个 contentIntent 是点击通知时候的动作。这个顾名思义，是通知被删除的时候的动作，例如说通知被删掉了，可以做一些清除发送通知相关的数据的操作。

* **fullScreenIntent**
这个看名字不是太直白（但是也有点沾边），这个是指发送通知的时候做的动作，例如说闹钟来了，对通知栏发送一条通知，顺带 launch 闹钟界面的 activity（系统闹钟的做法）；或者是插入 usb，发送 usb 连接通知，顺带 launch usb storage 操作界面。

* **tickerText(Char CharSequence)**
设置前面说的那个 ticker 显示的文本信息。这个也是系统有一个标准模板的，左边一个 icon，右边是文本。模板中 ticker 的 icon 是取 Notification 的 icon（源码中本着封装的原则，不同模块搞了一堆数据结构，又绕了半天，这篇侧重应用，代码这里不分析了）。所以 ticker 图标是和状态栏上的显示的图标是一样的，独立的只能设置文本。如果你没设 tickerText（为 null），那你发通知的时间就没 ticker 显示了（就是没那段通知滚动的动画了）。

* **tickerView(RemoteViews)**
看到前面 tickerText 那里提到模板就能猜到 ticker 也是可以自定义的。但是注意一点原生的 android 系统只有 Tablet UI 才允许你自定义 ticker view，因为只有 Tablet UI 才会使用 Notification 中设置的 tickerView。这里顺带说下 android SystemUI 的一些策略问题：

SystemUI 的状态栏（包括虚拟按键、通知面板）以分为好几种 UI 风格，目前已有的是 phone、table、tv。它共同继续只同一个父类 BaseStatusBar。然后实现不同的 UI 风格。在 frameworks/base/package/SystemUI/src/com/android/systemui/statusbar 下面有对应的包。framework 中可以在 xml 中指定要运行的 UI 风格。要想知道自己的设备状态用的是哪一种风格，拿 hierarchyviewer 看下状态栏的包名和类名就行了。

当然这是原生定的，你也可以改成 phone 也可以使用自定义的 tickerView，或者你自己都可以完全搞一套状态栏 UI 风格（只要实现了父类抽象的功能即可）。不过 android 内部也是矛盾不断，4.0 ~ 4.2 的时候原生还是有 phone 和 table 2种风格的（tv 的现在基本上是摆设），到了 4.4 之后 android 就把 table 给干掉了，然后全部只有 phone 的（也许是想统一手机和平板的风格吧）。所以说现在这个接口在原生系统上就是摆设，老老实实用系统标准的模板。 


经过上面的介绍，你会发现前面 normal view 下的通知栏，好像还有一个元素（6）没哪个接口可以设置。对的，在直接使用 Notification 设置是没办法设置右边那个小图标的（我不知道这个小图标是不是后面新加入的新元素）。因为这种方法已经被系统标记为过时的（deprecated），新的 sdk 推荐你使用 Notification.Builder 来设置通知。当然系统为了兼容性还保留着以前的接口，你要坚持使用老接口也可以，就是功能没那么全而已。 

### 使用 Notification.Builder

自从 4.0（应该是吧，我就不具体去考查版本了）通知新加入了一种模式，就出现了这个 Builder 的辅助类。我们先来说下新加入的这种通知模式。这模式官方叫 big view。一般通知的表现是在通知面板上一条一条的，虽然可以自定义 content view，但是系统规定死了 normal view 下每一条通知的高度的，所以所有 normal view 的通知的大小都是一样的（这样才好看）。但是你会发现某些音乐播放器的通知栏上的控制工具会比普通的通知要大，这个就是 big view 模式：

1. big view 高度没有限制（我看代码是，具体没试过），可以比 normal view 高很多（宽是不可能了，屏幕就那么宽），效果见上面的 big view 图。
2. 一个 Notification 可以有 normal view 和 big view。
3. 通知普通情况下显示 normal view，可以通过双指滑动展开 big view（隐藏 normal view），双指缩放关闭 big view（显示 normal view），其实好像不是双指，但是我实在没明白官方怎么个操作法的，但是双指很容易能搞出来。

big view 系统也是提供了若干模板，通过 new 不同的 Notiifcation.Style 可以设置不同风格的 big view 模板，当然也可以自定义（自己设置 bigContentView）。这就不细说 Builder、Style 有哪些接口了，自己去看下官方文档一目了然（其中包括上面说的设置右边的那个小图标的接口）。这里稍微看下系统是在 Builder 和 Style 中怎么帮我们创建 Notification 的 content view 的。

* **不使用风格**
如果直接 new Notification.Builder 然后设置一堆东西，然后调用 builder 的话，会是这样的：

```java
// ============== Notification.java =====================

        /*  
         * Combine all of the options that have been set and return a new {@link Notification}
         * object.
         */
        public Notification build() {
            if (mStyle != null) {
                return mStyle.build();
            } else {
                return buildUnstyled();
            }    
        }   
```

你如果没有调用 setStyle 设置任何风格的话，使用的是 buildUnstyled：

```java
// ============== Notification.java =====================

        /*
         * Apply the unstyled operations and return a new {@link Notification} object.
         */
        private Notification buildUnstyled() { 
            Notification n = new Notification();
            n.when = mWhen;
            n.icon = mSmallIcon;           
            n.iconLevel = mSmallIconLevel; 
            n.number = mNumber;            
            // 创建 normal view 的 content view
            n.contentView = makeContentView();
            n.contentIntent = mContentIntent;
            n.deleteIntent = mDeleteIntent;
            n.fullScreenIntent = mFullScreenIntent;
            n.tickerText = mTickerText;
            // 创建 ticker view    
            n.tickerView = makeTickerView();
            n.largeIcon = mLargeIcon;      
            n.sound = mSound;
            n.audioStreamType = mAudioStreamType;
            n.vibrate = mVibrate;          
            n.ledARGB = mLedArgb;          
            n.ledOnMS = mLedOnMs;          
            n.ledOffMS = mLedOffMs;        
            n.defaults = mDefaults;        
            n.flags = mFlags;
            // 创建 big view 的 content view
            n.bigContentView = makeBigContentView();
            if (mLedOnMs != 0 && mLedOffMs != 0) {
                n.flags |= FLAG_SHOW_LIGHTS;   
            }
            if ((mDefaults & DEFAULT_LIGHTS) != 0) {
                n.flags |= FLAG_SHOW_LIGHTS;   
            }
            if (mKindList.size() > 0) {    
                n.kind = new String[mKindList.size()];
                mKindList.toArray(n.kind);     
            } else {
                n.kind = null;
            }
            n.priority = mPriority;        
            n.extras = mExtras != null ? new Bundle(mExtras) : null;
            if (mActions.size() > 0) {     
                n.actions = new Action[mActions.size()]; 
                mActions.toArray(n.actions);   
            }
            return n;
        }
```

我们稍微看下 normal view 和 big view 的创建：

```java
// ============== Notification.java =====================

        private RemoteViews makeContentView() {
            if (mContentView != null) {    
                return mContentView;           
            } else {
                return applyStandardTemplate(R.layout.notification_template_base, true); // no more special large_icon flavor
            }
        }

        private RemoteViews applyStandardTemplate(int resId, boolean fitIn1U) {
            RemoteViews contentView = new RemoteViews(mContext.getPackageName(), resId);
            boolean showLine3 = false;
            boolean showLine2 = false;
            int smallIconImageViewId = R.id.icon;
            if (mLargeIcon != null) {
                contentView.setImageViewBitmap(R.id.icon, mLargeIcon);
                smallIconImageViewId = R.id.right_icon;
            }
            if (mPriority < PRIORITY_LOW) {
                contentView.setInt(R.id.icon,
                        "setBackgroundResource", R.drawable.notification_template_icon_low_bg);
                contentView.setInt(R.id.status_bar_latest_event_content,
                        "setBackgroundResource", R.drawable.notification_bg_low);
            }
            if (mSmallIcon != 0) {
                contentView.setImageViewResource(smallIconImageViewId, mSmallIcon);
                contentView.setViewVisibility(smallIconImageViewId, View.VISIBLE);
            } else {
                contentView.setViewVisibility(smallIconImageViewId, View.GONE);
            }
            if (mContentTitle != null) {
                contentView.setTextViewText(R.id.title, mContentTitle);
            }
            if (mContentText != null) {
                contentView.setTextViewText(R.id.text, mContentText);
                showLine3 = true;
            }
            if (mContentInfo != null) {
                contentView.setTextViewText(R.id.info, mContentInfo);
                contentView.setViewVisibility(R.id.info, View.VISIBLE);
                showLine3 = true;
            } else if (mNumber > 0) {
                final int tooBig = mContext.getResources().getInteger(
                        R.integer.status_bar_notification_info_maxnum);
                if (mNumber > tooBig) {
                    contentView.setTextViewText(R.id.info, mContext.getResources().getString(
                                R.string.status_bar_notification_info_overflow));
                } else {
                    NumberFormat f = NumberFormat.getIntegerInstance();
                    contentView.setTextViewText(R.id.info, f.format(mNumber));
                }
                contentView.setViewVisibility(R.id.info, View.VISIBLE);
                showLine3 = true;
            } else {
                contentView.setViewVisibility(R.id.info, View.GONE);
            }

            // Need to show three lines?
            if (mSubText != null) {
                contentView.setTextViewText(R.id.text, mSubText);
                if (mContentText != null) {
                    contentView.setTextViewText(R.id.text2, mContentText);
                    contentView.setViewVisibility(R.id.text2, View.VISIBLE);
                    showLine2 = true;
                } else {
                    contentView.setViewVisibility(R.id.text2, View.GONE);
                }  
            } else {   
                contentView.setViewVisibility(R.id.text2, View.GONE);
                if (mProgressMax != 0 || mProgressIndeterminate) {
                    contentView.setProgressBar(
                            R.id.progress, mProgressMax, mProgress, mProgressIndeterminate);
                    contentView.setViewVisibility(R.id.progress, View.VISIBLE);
                    showLine2 = true;
                } else {
                    contentView.setViewVisibility(R.id.progress, View.GONE);
                }
            }
            if (showLine2) { 
                if (fitIn1U) {
                    // need to shrink all the type to make sure everything fits
                    final Resources res = mContext.getResources();
                    final float subTextSize = res.getDimensionPixelSize(
                            R.dimen.notification_subtext_size);
                    contentView.setTextViewTextSize(R.id.text, TypedValue.COMPLEX_UNIT_PX, subTextSize);
                }
                // vertical centering
                contentView.setViewPadding(R.id.line1, 0, 0, 0, 0);
            }
            
            if (mWhen != 0 && mShowWhen) {
                if (mUseChronometer) {
                    contentView.setViewVisibility(R.id.chronometer, View.VISIBLE);
                    contentView.setLong(R.id.chronometer, "setBase",
                            mWhen + (SystemClock.elapsedRealtime() - System.currentTimeMillis()));
                    contentView.setBoolean(R.id.chronometer, "setStarted", true);
                } else {
                    contentView.setViewVisibility(R.id.time, View.VISIBLE);
                    contentView.setLong(R.id.time, "setTime", mWhen);
                }
            } else {
                contentView.setViewVisibility(R.id.time, View.GONE);
            }

            contentView.setViewVisibility(R.id.line3, showLine3 ? View.VISIBLE : View.GONE);
            contentView.setViewVisibility(R.id.overflow_divider, showLine3 ? View.VISIBLE : View.GONE);
            return contentView;
        }
```

具体细节不看了，就是根据 Builder 设置的参数设置 R.layout.notification_template_base 中的界面元素。从这里能看出使用 Builder 比上面使用 Notification.setLatestEventInfo 创建的模板多很多东西。所以要想更多的控制自己程序发送的通知样式还是使用 sdk 推荐的接口比较好。

然后看下 big view 的：

```java
// ============== Notification.java =====================

        private RemoteViews makeBigContentView() {
            if (mActions.size() == 0) return null;

            return applyStandardTemplateWithActions(R.layout.notification_template_big_base);
        }

        private RemoteViews applyStandardTemplateWithActions(int layoutId) {
            RemoteViews big = applyStandardTemplate(layoutId, false);

            int N = mActions.size();       
            if (N > 0) {
                // Log.d("Notification", "has actions: " + mContentText);
                big.setViewVisibility(R.id.actions, View.VISIBLE);
                big.setViewVisibility(R.id.action_divider, View.VISIBLE);
                if (N>MAX_ACTION_BUTTONS) N=MAX_ACTION_BUTTONS;
                big.removeAllViews(R.id.actions);
                for (int i=0; i<N; i++) {      
                    final RemoteViews button = generateActionButton(mActions.get(i));
                    //Log.d("Notification", "adding action " + i + ": " + mActions.get(i).title);
                    big.addView(R.id.actions, button);
                }
            }
            return big;
        }
```

不设置任何风格的 Builder，并且又没设置任何 Actions，是没有 big view 的。

* **使用风格**
Builder 有一个 setStyle 的接口，可以设置一个风格，系统 Notification 目前提供了下面几种风格：

1. BigPictureStyle: 就是能设置很大一张的
2. BigTextStyle: 有很大的空间（高度）显示一大串文本
3. InboxStyle： 就是上面贴图的那个 big view

我这里偷下懒就不一个一个上效果图了（自己试一下就能看到效果了）。然后这里的风格的区别主要是在 big view 上，这几个风格分别提供了3种 big view 的风格，设置也是设置相应的 big view 的模板元素。我们来稍微看下源码，还记得上面那个 Builder 的 build 函数么，如果 mStyle 不是 null 的话，那么就调用相应 Style 的 build 函数，我们这里稍微看下 BigPictureStyle 的：

```java
// ============== Notification.java =====================

    /*
     * Helper class for generating large-format notifications that include a large image attachment.
     *
     * This class is a "rebuilder": It consumes a Builder object and modifies its behavior, like so:
     * <pre class="prettyprint">
     * Notification noti = new Notification.BigPictureStyle(
     *      new Notification.Builder()
     *         .setContentTitle(&quot;New photo from &quot; + sender.toString())
     *         .setContentText(subject)
     *         .setSmallIcon(R.drawable.new_post)
     *         .setLargeIcon(aBitmap))
     *      .bigPicture(aBigBitmap)
     *      .build();
     * </pre>
     *
     * @see Notification#bigContentView
     */
    public static class BigPictureStyle extends Style {
        private Bitmap mPicture;       
        private Bitmap mBigLargeIcon;  
        private boolean mBigLargeIconSet = false;

        public BigPictureStyle() {     
        }

        public BigPictureStyle(Builder builder) {
            setBuilder(builder);           
        }

... ...

        private RemoteViews makeBigContentView() {
            RemoteViews contentView = getStandardView(R.layout.notification_template_big_picture);

            contentView.setImageViewBitmap(R.id.big_picture, mPicture);

            return contentView;
        }

        @Override
        public Notification build() {
            checkBuilder();
            Notification wip = mBuilder.buildUnstyled();
            if (mBigLargeIconSet ) {
                mBuilder.mLargeIcon = mBigLargeIcon;
            }
            wip.bigContentView = makeBigContentView();
            return wip;
        }
    }
```

从代码中看，这些 Style 就是先调用 Builder 的无风格 build 函数，然后再重新把 bigContentView 自己生成一个，覆盖以前的而已。其他2个我就不贴代码了，套路基本上是一样的。通知栏智能机时代是 android 发明的（IOS 的在后面，android 在界面设计方面有领先 IOS 的地方了哟），加入了 big view 之后 UI 更加灵活了，好像 5.0 又多了一个新功能，以后有时间再看看。

## 系统处理流程 

上面说了一些 Notification 的基本用法，也稍微分析下源代码的一些实现。这些稍微说下，应用对 NM 发送一条通知，framework 中会对应生成什么数据，然后通知的 content view 是怎么加到 SystemUI 的状态栏和通知面板中的。先是上一张图先：

![流程图](http://7u2hy4.com1.z0.glb.clouddn.com/android/SystemUI-notification/Send-notification.png "流程图")

从图中可以看到一个通知从 app 自己构造通知然后调用 NotificationManager（NM）的接口发送，经过了 NotificationManagerService（NMS）到 StatusBarManagerService(SBMS) 最后到添加 Notification 的 UI 元素到 SystemUI 中。好像也有点麻烦的样子，然后图中蓝色的部分是每个模块（NMS、SBMS、SystemUI）保存通知的相关数据（前面说了每个模块都不一样，还真是不一样）。然后我们再贴下通知相关的 UI 元素最后在 SystemUI 中的表现：

![UI 效果](http://7u2hy4.com1.z0.glb.clouddn.com/android/SystemUI-notification/notification-ui.jpeg "UI 效果")
(PS: 这不是 android 原生的 SystemUI，是被我定制过的，通知没分组了)

上面我们稍微注意下每个模块对应的数据结构，和最后扔到 SystemUI 中的 view 就差不多能摸清系统处理通知的流程了。下面开始跟流程：

* **1.App**
首先是构造 Notification，前面分析过了，然后获取 NM 调用 notify 的接口，把 id 和 Notificaiton 传给 NM。这里代码前面贴过了，回去看看就行。


* **2.NotificationManagerService**
然后 NM 会调用 NMS 的 enqueueNotificationInternal（Notification 是 Parcelable 所以可以当作 Binder 的参数传给 NMS）：

```java
    // Not exposed via Binder; for system use only (otherwise malicious apps could spoof the
    // uid/pid of another application)
    public void enqueueNotificationInternal(String pkg, int callingUid, int callingPid,
            String tag, int id, Notification notification, int[] idOut, int userId)
    {
        if (DBG) {
            Slog.v(TAG, "enqueueNotificationInternal: pkg=" + pkg + " id=" + id + " notification=" + notification);
        }
        // 判断是不是当前用户应用发出来的通知
        checkCallerIsSystemOrSameApp(pkg);
        // 判断是不是系统应用发出来的通知
        final boolean isSystemNotification = isUidSystem(callingUid) || ("android".equals(pkg));

        final int userId = ActivityManager.handleIncomingUser(callingPid,
                callingUid, incomingUserId, true, false, "enqueueNotification", pkg);
        final UserHandle user = new UserHandle(userId);

        // 如果不是系统应用的话，限制一下一个包发送总共应用的条数，原生限制 50 条
        // 通过统计 NotificationRecord 的 list 包名一样的通知记录 
        // Limit the number of notifications that any given package except the android
        // package can enqueue.  Prevents DOS attacks and deals with leaks.
        if (!isSystemNotification) {
            synchronized (mNotificationList) {
                int count = 0; 
                final int N = mNotificationList.size();
                for (int i=0; i<N; i++) {
                    final NotificationRecord r = mNotificationList.get(i);
                    if (r.sbn.getPackageName().equals(pkg) && r.sbn.getUserId() == userId) {
                        count++;
                        if (count >= MAX_PACKAGE_NOTIFICATIONS) {
                            Slog.e(TAG, "Package has already posted " + count
                                    + " notifications.  Not showing more.  package=" + pkg);
                            return;
                        }
                    }
                }
            }
        }

        // This conditional is a dirty hack to limit the logging done on
        //     behalf of the download manager without affecting other apps.
        if (!pkg.equals("com.android.providers.downloads")
                || Log.isLoggable("DownloadManager", Log.VERBOSE)) {
            EventLog.writeEvent(EventLogTags.NOTIFICATION_ENQUEUE, pkg, id, tag, userId,
                    notification.toString());
        }

        if (pkg == null || notification == null) {
            throw new IllegalArgumentException("null not allowed: pkg=" + pkg
                    + " id=" + id + " notification=" + notification);
        }
        if (notification.icon != 0) {
            // content view 必须要有
            if (notification.contentView == null) {
                throw new IllegalArgumentException("contentView required: pkg=" + pkg
                        + " id=" + id + " notification=" + notification);
            }
        }

        // === Scoring ===

        // 通知有个分数排序的，分数高的排在列表前面，在 SystemUI 的显示通知列表中也靠前，我们以后再分析这个分数怎么算的
        // 0. Sanitize inputs
        notification.priority = clamp(notification.priority, Notification.PRIORITY_MIN, Notification.PRIORITY_MAX);
        // Migrate notification flags to scores
        if (0 != (notification.flags & Notification.FLAG_HIGH_PRIORITY)) {
            if (notification.priority < Notification.PRIORITY_MAX) notification.priority = Notification.PRIORITY_MAX;
        } else if (SCORE_ONGOING_HIGHER && 0 != (notification.flags & Notification.FLAG_ONGOING_EVENT)) {
            if (notification.priority < Notification.PRIORITY_HIGH) notification.priority = Notification.PRIORITY_HIGH;
        }

        // 1. initial score: buckets of 10, around the app 
        int score = notification.priority * NOTIFICATION_PRIORITY_MULTIPLIER; //[-20..20]

        // 2. Consult external heuristics (TBD)

        // 3. Apply local rules

        // 在系统设置中，可以屏蔽掉某一个应用的通知
        // blocked apps
        if (ENABLE_BLOCKED_NOTIFICATIONS && !isSystemNotification && !areNotificationsEnabledForPackageInt(pkg)) {
            score = JUNK_SCORE;
            Slog.e(TAG, "Suppressing notification from package " + pkg + " by user request.");
        }
        
        if (DBG) {
            Slog.v(TAG, "Assigned score=" + score + " to " + notification);
        }

        if (score < SCORE_DISPLAY_THRESHOLD) {
            // Notification will be blocked because the score is too low.
            Slog.i(TAG, "Notification=" + notification + " score is low than display threshold, we don't show it.");
            return;
        }

        // Should this notification make noise, vibe, or use the LED?
        final boolean canInterrupt = (score >= SCORE_INTERRUPTION_THRESHOLD);

        // 喜闻乐见的 SS 业务函数多线程同步锁
        synchronized (mNotificationList) {
            // 这里用发通知 app的包名、通知的 tag、id、app 的 uid、app 的 pid、
            // app 的用户 id 和 Notiifcaiton 构造出一个 NotiifcationRecord（nr）
            NotificationRecord r = new NotificationRecord(pkg, tag, id,  
                    callingUid, callingPid, userId,
                    score,
                    notification);
            NotificationRecord old = null;

            // 这里去看之前应用有没有发送过相同的通知（通过前面的标示区分）
            int index = indexOfNotificationLocked(pkg, tag, id, userId);
            if (index < 0) { 
                // 如果是新通知就加入到 NR 列表中
                mNotificationList.add(r);
            } else {
                // 如果原来有，取出来原来的，顺带把原来的从 list 删掉，再把新的加进去 -_-||
                old = mNotificationList.remove(index);
                mNotificationList.add(index, r);
                // Make sure we don't lose the foreground service state.
                if (old != null) {
                    notification.flags |=
                        old.notification.flags&Notification.FLAG_FOREGROUND_SERVICE;
                }    
            }   
            
            // Ensure if this is a foreground service that the proper additional
            // flags are set.
            if ((notification.flags&Notification.FLAG_FOREGROUND_SERVICE) != 0) {
                notification.flags |= Notification.FLAG_ONGOING_EVENT
                        | Notification.FLAG_NO_CLEAR;
            }

            final int currentUser;
            final long token = Binder.clearCallingIdentity();
            try {
                currentUser = ActivityManager.getCurrentUser();
            } finally {
                Binder.restoreCallingIdentity(token);
            }

            if (notification.icon != 0) {
                // 通过和构造 NR 差不多的东西构造出一个 StatusBarNotification(sbn)
                final StatusBarNotification n = new StatusBarNotification(
                        pkg, id, tag, r.uid, r.initialPid, score, notification, user);
                // 判断下原来的和新的是不是同一样，通过以前通知保存的 SBMS 返回的 IBinder 来判断
                // 如果是一样的话那么调用 SBMS 的 updateNotification
                if (old != null && old.statusBarKey != null) {
                    r.statusBarKey = old.statusBarKey;
                    long identity = Binder.clearCallingIdentity();
                    try {
                        mStatusBar.updateNotification(r.statusBarKey, n);
                    }
                    finally {
                        Binder.restoreCallingIdentity(identity);
                    }
                } else {
                    // 如果是新发送的通知，那么调用 SBMS 的 addNotification 添加通知
                    long identity = Binder.clearCallingIdentity();
                    try {
                        // NR 保存一下 SBMS 返回的 IBinder(Bp)
                        r.statusBarKey = mStatusBar.addNotification(n);
                        if ((n.notification.flags & Notification.FLAG_SHOW_LIGHTS) != 0
                                && canInterrupt) {
                            mAttentionLight.pulse();
                        }
                    }
                    finally {
                        Binder.restoreCallingIdentity(identity);
                    }
                }
                // Send accessibility events only for the current user.
                if (currentUser == userId) {
                    sendAccessibilityEvent(notification, pkg);
                }  
            } else {
                // 对于没有设置 icon 的通知，当作 cancel 来处理（删掉通知）
                Slog.e(TAG, "Ignoring notification with icon==0: " + notification);
                if (old != null && old.statusBarKey != null) {
                    long identity = Binder.clearCallingIdentity();
                    try {
                        mStatusBar.removeNotification(old.statusBarKey);
                    }
                    finally {
                        Binder.restoreCallingIdentity(identity);
                    }
                }
            }

// 下面还有一串，不过和发送关系不太大，我们先忽略
... ...

        }

        idOut[0] = id;
    }
```

NMS 处理通知的函数一开始是一些检测性的工作（其实很多 SS 的业务函数一开始都是一些检测工作）。我们稍微看下 checkCallerIsSystemOrSameApp：

```java
    void checkCallerIsSystemOrSameApp(String pkg) {
        // 获取 IPC 调用者（发送通知的应用）的 uid
        int uid = Binder.getCallingUid();
        // 如果是系统应用（uid 0 是 root 组），那么可以向所有用户发送通知
        if (UserHandle.getAppId(uid) == Process.SYSTEM_UID || uid == 0) {
            return;
        }
        try {
            // 如果发送通知的应用（普通应用）的 uid 不属于当前用户，那么不允许发送
            ApplicationInfo ai = AppGlobals.getPackageManager().getApplicationInfo(
                    pkg, 0, UserHandle.getCallingUserId());
            if (!UserHandle.isSameApp(ai.uid, uid)) {
                throw new SecurityException("Calling uid " + uid + " gave package"
                        + pkg + " which is owned by uid " + ai.uid);
            }
        } catch (RemoteException re) { 
            throw new SecurityException("Unknown package " + pkg + "\n" + re);
        }
    }
```

这里可以看得出系统在多用户方面做的一些限制操作。然后 NMS 这里使用的数据结构是 NotificationRecrod，然后 NMS 是用一个 ArrayList 来保存的:

```java
/* {@hide} */
public class NotificationManagerService extends INotificationManager.Stub
{
... ...

    private final ArrayList<NotificationRecord> mNotificationList =
            new ArrayList<NotificationRecord>();

... ...

    private static final class NotificationRecord
    {
        final String pkg;
        final String tag;
        final int id;
        final int uid;
        final int initialPid;
        final int userId;
        final Notification notification;
        final int score;
        IBinder statusBarKey;

... ...

    }

... ...

}
```

这里会判断一下之前应用有没有发送相同标志的通知（indexOfNotificationLocked 前面有贴代码），如果有的话从 NR list 中删掉原来的，再插入新的，如果没有的话直接插入新的。然后再构造出一个 StatusBarNotifcation。注意一下这个数据结构虽然在 NMS 中 new 了出来，但是是在 SBMS 中使用的，这里只是为了传递给 SBMS 而已（SBMS 提供的接口的参数是 sbn），所以 NMS 中只是持有 NR 的数据而已，sbn 在这里这是临时数据。然后根据 indexOfNotificationLocked 的结果是否已经存在 NR 记录会调用 SBMS 不同的接口，如果有会调用 updateNotification，新通知的话会调用 addNotification。这里我们先以新通知来分析，所以到这里就到 SBMS 中了去。细心的会发现在我流程图中 NMS 到 SBMS 那里没有标 IPC 而是标了 Bn（表示本地）。这是因为 NMS 和 SBMS 都是在 SS（SystemServer）进程中的（忘记了的去 Binder 篇复习下），所以它之间可以直接持有对方的对象直接调用相关的接口，无需跨进程。同时 SBMS 提供的 IPC 接口只是占本身接口的一小部分的（aidl 中的），这里调用的接口是没在 aidl 中申明的，所以别的进程只能使用 SMBS 很有限的一部分功能。可以说这里 NMS 转到 SBMS 属于 SS 内部的功能。

* **3.StatusBarManagerService**
现在从 NMS 转到 SBMS 中的 addNotification 中了：

```java
public class StatusBarManagerService extends IStatusBarService.Stub 
    implements WindowManagerService.OnHardKeyboardStatusChangeListener
{
    static final String TAG = "StatusBarManagerService";
    static final boolean SPEW = false;

    final Context mContext;
    final WindowManagerService mWindowManager;
    Handler mHandler = new Handler();
    NotificationCallbacks mNotificationCallbacks;
    volatile IStatusBar mBar;
    StatusBarIconList mIcons = new StatusBarIconList();
    // 保存数据结构的是一个 HashMap
    HashMap<IBinder,StatusBarNotification> mNotifications
            = new HashMap<IBinder,StatusBarNotification>();

... ...

    public IBinder addNotification(StatusBarNotification notification) {
        synchronized (mNotifications) {
            // 生成 NotificationRecord 的 Binder 对象
            IBinder key = new Binder();    
            mNotifications.put(key, notification);
            if (mBar != null) {            
                try {
                    mBar.addNotification(key, notification);
                } catch (RemoteException ex) { 
                }
            }
            return key;
        }
    }

... ...

}
```

这个函数非常简单，因为这里的 SBMS 其实只是起到一个桥接作用，大部分工作在提供了 IStatusBar 接口的 SystemUI 中。不过虽然简单这里 SBMS 还是持有了一个数据结构：StatusBarNotification：

```java
/*
 * Class encapsulating a Notification. Sent by the NotificationManagerService to the IStatusBar (in System UI).
 */
public class StatusBarNotification implements Parcelable {
    public final String pkg;
    public final int id; 
    public final String tag;
    public final int uid;
    public final int initialPid;
    // TODO: make this field private and move callers to an accessor that
    // ensures sourceUser is applied.
    public final Notification notification;
    public final int score;
    public final UserHandle user;

... ...

}
```

可以看到这里 StatusBarNotification 也是支持 Parcelable，不过构造的地方是在 NMS 中（前面 NMS 那里传过来的）。然后 SBMS 保存 sbn 的是一个 HashMap，以这条新通知的 IBinder 对象作为 key。这里的 IBinder 在 SBMS 这里本地生成，所以是 Bn，所以通知的 IBinder 对象在 SS 中都是 Bn 来的。这个 key 被返回给 NMS 同时保存在这条记录对应的 NotificationRecord 的 statusBarKey 中。到后面 SystemUI 中标示通知 view 相关对象的时候也是拿一个 IBinder 对象区分的。 

然后直接调用 SBMS 的 IstatusBar 对象的 addNotification 函数去 SystemUI 中去处理通知 UI 表现相关的东西去了。

* **4.SystemUI**

上面说到 SBMS 有一个 IStatusBar 对象。前面说了 SystemUI 的多 UI 风格，这里就是抽象地方表现之一。SystemUI 抽象了一个状态栏的抽象基类： BaseStatusBar 然后定义了一系列状态栏的功能接口，只要实现了这些接口，那么可以表现出不同的 UI 风格（"小屏"手机，大屏平板，超大屏电视等）。其中抽象出一个 IStatusBar 的 IBinder 接口提供 SS 中的服务调用 SystemUI 相关的接口，让 SystemUI 在 UI 上展现一条通知。然后 BaseStatusBar 的具体子类只要能满足 SS 中（具体是 SBMS）的接口需求就行。所以 BaseStatusBar 中在状态栏初始化的时候会向 SBMS 注册当前实现 IStatusBar 接口的对象（当前使用的 UI 风格）：

```java
// ================= BaseStatusBar.java ========================

    public void start() {
        mWindowManager = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        mWindowManagerService = WindowManagerGlobal.getWindowManagerService();
        mDisplay = mWindowManager.getDefaultDisplay();
        
        // 向 SM 获取 SBMS
        mBarService = IStatusBarService.Stub.asInterface(
                ServiceManager.getService(Context.STATUS_BAR_SERVICE));

        // Connect in to the status bar manager service
        StatusBarIconList iconList = new StatusBarIconList();
        ArrayList<IBinder> notificationKeys = new ArrayList<IBinder>();
        ArrayList<StatusBarNotification> notifications = new ArrayList<StatusBarNotification>();
        mCommandQueue = new CommandQueue(this, iconList);

        int[] switches = new int[7];
        ArrayList<IBinder> binders = new ArrayList<IBinder>();
        try {
            // 调用 SBMS 的注册接口注册
            mBarService.registerStatusBar(mCommandQueue, iconList, notificationKeys, notifications,
                    switches, binders);
        } catch (RemoteException ex) {
            // If the system process isn't there we're doomed anyway.
        }  

... ...

    }
```

然后你发现其实注册的不是 BaseStatusBar 自己，而是一个叫 CommandQueue 的东西：

```java
// ================= CommandQueue.java ========================

/*
 * This class takes the functions from IStatusBar that come in on
 * binder pool threads and posts messages to get them onto the main
 * thread, and calls onto Callbacks.  It also takes care of
 * coalescing these calls so they don't stack up.  For the calls
 * are coalesced, note that they are all idempotent.
 */
public class CommandQueue extends IStatusBar.Stub {
    private static final int INDEX_MASK = 0xffff;
    private static final int MSG_SHIFT  = 16;
    private static final int MSG_MASK   = 0xffff << MSG_SHIFT;

    private static final int OP_SET_ICON    = 1;
    private static final int OP_REMOVE_ICON = 2;

    private static final int MSG_ICON                       = 1 << MSG_SHIFT;
    private static final int MSG_ADD_NOTIFICATION           = 2 << MSG_SHIFT;
    private static final int MSG_UPDATE_NOTIFICATION        = 3 << MSG_SHIFT;
    private static final int MSG_REMOVE_NOTIFICATION        = 4 << MSG_SHIFT;
    private static final int MSG_DISABLE                    = 5 << MSG_SHIFT;
    private static final int MSG_EXPAND_NOTIFICATIONS       = 6 << MSG_SHIFT;
    private static final int MSG_COLLAPSE_PANELS            = 7 << MSG_SHIFT;
    private static final int MSG_EXPAND_SETTINGS            = 8 << MSG_SHIFT;
    private static final int MSG_SET_SYSTEMUI_VISIBILITY    = 9 << MSG_SHIFT;
    private static final int MSG_TOP_APP_WINDOW_CHANGED     = 10 << MSG_SHIFT;
    private static final int MSG_SHOW_IME_BUTTON            = 11 << MSG_SHIFT;
    private static final int MSG_SET_HARD_KEYBOARD_STATUS   = 12 << MSG_SHIFT;
    private static final int MSG_TOGGLE_RECENT_APPS         = 13 << MSG_SHIFT;
    private static final int MSG_PRELOAD_RECENT_APPS        = 14 << MSG_SHIFT;
    private static final int MSG_CANCEL_PRELOAD_RECENT_APPS = 15 << MSG_SHIFT;
    private static final int MSG_SET_NAVIGATION_ICON_HINTS  = 16 << MSG_SHIFT;

... ...

    private StatusBarIconList mList;
    private Callbacks mCallbacks;
    private Handler mHandler = new H();

... ...

    // 这里的 interface 的接口，正好全是 IStatusBar.aidl 中的接口
    /*
     * These methods are called back on the main thread.
     */
    public interface Callbacks {
        public void addIcon(String slot, int index, int viewIndex, StatusBarIcon icon);
        public void updateIcon(String slot, int index, int viewIndex,
                StatusBarIcon old, StatusBarIcon icon);
        public void removeIcon(String slot, int index, int viewIndex);
        public void addNotification(IBinder key, StatusBarNotification notification);
        public void updateNotification(IBinder key, StatusBarNotification notification);
        public void removeNotification(IBinder key);
        public void disable(int state);
        public void animateExpandNotificationsPanel();
        public void animateCollapsePanels(int flags);
        public void animateExpandSettingsPanel();
        public void setSystemUiVisibility(int vis, int mask);
        public void topAppWindowChanged(boolean visible);
        public void setImeWindowStatus(IBinder token, int vis, int backDisposition);
        public void setHardKeyboardStatus(boolean available, boolean enabled);
        public void toggleRecentApps();
        public void preloadRecentApps();
        public void showSearchPanel();
        public void hideSearchPanel();
        public void cancelPreloadRecentApps();
        public void setNavigationIconHints(int hints);
    }

    public CommandQueue(Callbacks callbacks, StatusBarIconList list) {
        mCallbacks = callbacks;
        mList = list;
    }

... ...

    public void addNotification(IBinder key, StatusBarNotification notification) {
        synchronized (mList) {
            NotificationQueueEntry ne = new NotificationQueueEntry();
            ne.key = key;
            ne.notification = notification;
            mHandler.sendMessageDelayed(mHandler.obtainMessage(MSG_ADD_NOTIFICATION, 0, 0, ne), 300);
        }
    }

... ...

    private final class H extends Handler {
        public void handleMessage(Message msg) {
            final int what = msg.what & MSG_MASK;
            switch (what) {
... ...
                case MSG_ADD_NOTIFICATION: {   
                    final NotificationQueueEntry ne = (NotificationQueueEntry)msg.obj;
                    mCallbacks.addNotification(ne.key, ne.notification);
                    break;
                }
... ...
            }
        }
    }
}

// ================= BaseStatusBar.java ========================

public abstract class BaseStatusBar extends SystemUI implements
        CommandQueue.Callbacks {

... ...

}
```

看完上面的代码，就知道 SystemUI 中状态栏的设计架构了吧。就是说 IStatusBar.aidl 定义一系列接口给 SS 用，然后 BaseStatusBar 完成一些状态栏公用的工作（例如生成 notification row 的模板），其他的接口交由子类实现。这里我们以 PhoneStatusBar（好像 4.4 之后 android 原生的 UI 都是用这个了，想看 Tablet UI 的可以拿 4.2 玩一下） 来说明一下 addNotification 的流程。

SBMS IPC 调用 addNotification 之后把通知的 IBinder key 和 sbn 传了过来，我们来看下 PhoneStatusBar 中的具体实现：

```java
    public void addNotification(IBinder key, StatusBarNotification notification) {
        if (DEBUG) Slog.d(TAG, "addNotification score=" + notification.score);
        // 大部分工作在这里
        StatusBarIconView iconView = addNotificationViews(key, notification);
        if (iconView == null) return;

        boolean immersive = false;
        try {
            immersive = ActivityManagerNative.getDefault().isTopActivityImmersive();
            if (DEBUG) {
                Slog.d(TAG, "Top activity is " + (immersive?"immersive":"not immersive"));
            }    
        } catch (RemoteException ex) {}

        // 如果通知设置了 fullScreenIntent，执行 fullScreenIntent 中设置的动作
        //（启动相应的 activity 或是发送相应的广播）
        if (notification.notification.fullScreenIntent != null) {
            // Stop screensaver if the notification has a full-screen intent.
            // (like an incoming phone call)
            awakenDreams();

            // not immersive & a full-screen alert should be shown
            if (DEBUG) Slog.d(TAG, "Notification has fullScreenIntent; sending fullScreenIntent");
            try {
                notification.notification.fullScreenIntent.send();
            } catch (PendingIntent.CanceledException e) { 
            }    
        } else {
            // usual case: status bar visible & not immersive

            // 如果没指定 fullScreenIntent 那么在状态栏上展现一下 ticker 动画
            // show the ticker if there isn't an intruder too
            if (mCurrentlyIntrudingNotification == null) {
                tick(null, notification, true);
            }    
        }    

        // Recalculate the position of the sliding windows and the titles.
        setAreThereNotifications();
        updateExpandedViewPos(EXPANDED_LEAVE_ALONE);
    }
```

这里一开始就调用了基类 BaseStatusBar 中的 addNotificationViews 函数，这个函数就属于状态栏的共用函数之一，通知在 SystemUI 上的 UI 元素基本上都由这个函数生成：

```java
    protected StatusBarIconView addNotificationViews(IBinder key,
            StatusBarNotification notification) {
        if (DEBUG) {
            Slog.d(TAG, "addNotificationViews(key=" + key + ", notification=" + notification);
        }

        // 构造状态栏小图标
        // Construct the icon.
        final StatusBarIconView iconView = new StatusBarIconView(mContext,
                notification.pkg + "/0x" + Integer.toHexString(notification.id),
                notification.notification);
        iconView.setScaleType(ImageView.ScaleType.CENTER_INSIDE);
        
        // 构造状态栏小图标对应数据 -_-||            
        final StatusBarIcon ic = new StatusBarIcon(notification.pkg,
                    notification.user,
                    notification.notification.icon,
                    notification.notification.iconLevel,
                    notification.notification.number,
                    notification.notification.tickerText);
        // 把数据设置给状态栏小图标
        if (!iconView.set(ic)) {
            handleNotificationError(key, notification, "Couldn't create icon: " + ic);
            return null;
        }
        // 构造 SystemUI 通知数据结构
        // Construct the expanded view.
        NotificationData.Entry entry = new NotificationData.Entry(key, notification, iconView);
        // 构造通知面板上的通知 view，并将其加入通知面板上 
        if (!inflateViews(entry, mPile)) {
            handleNotificationError(key, notification, "Couldn't expand RemoteViews for: "
                    + notification);
            return null;
        }
        
        // 保存通知数据
        // Add the expanded view and icon.
        int pos = mNotificationData.add(entry);
        if (DEBUG) {
            Slog.d(TAG, "addNotificationViews: added at " + pos);
        }
        updateExpansionStates();
        // 更新状态栏上的通知小图标（新的加入到状态栏上去）
        updateNotificationIcons();
        
        return iconView;
    }
```

这里 StatusBarIconView 别看是继承了 AnimatedImageView，其实最后 AnimatedImageView 最后是继承了 ImageView，也就是说状态栏上左上角那一排通知的小图标就是一堆 ImageView（右边那一排也是一样的，不过那一排叫 Status Icon，系统状态图标，和通知小图标是不一样的东西，刚开始容易搞混，这个东西后面我单独开一篇来说）。然后后面这里就是 framework 中最后一个模块 SystemUI 中持有的通知的数据结构了 NotificationData：

```java
/*
 * The list of currently displaying notifications.
 */
public class NotificationData {
    public static final class Entry {
        public IBinder key;
        public StatusBarNotification notification;
        public StatusBarIconView icon;
        public View row; // the outer expanded view
        public View content; // takes the click events and sends the PendingIntent
        public View expanded; // the inflated RemoteViews
        public ImageView largeIcon;
        protected View expandedLarge;
        public Entry() {}
        public Entry(IBinder key, StatusBarNotification n, StatusBarIconView ic) {
            this.key = key;
            this.notification = n;
            this.icon = ic; 
        } 

... ...

    // 保存的数据的最后是一个 ArrayList
    private final ArrayList<Entry> mEntries = new ArrayList<Entry>();
    private final Comparator<Entry> mEntryCmp = new Comparator<Entry>() {
        // sort first by score, then by when
        public int compare(Entry a, Entry b) {
            final StatusBarNotification na = a.notification;
            final StatusBarNotification nb = b.notification;
            int d = na.score - nb.score;
            return (d != 0)
                ? d
                : (int)(na.notification.when - nb.notification.when);
        }
    };

... ...

    // 它的大多数接口都是封装对上面那个 ArrayList 的操作 
    public int add(Entry entry) {
        int i;
        int N = mEntries.size();
        for (i=0; i<N; i++) {
            if (mEntryCmp.compare(mEntries.get(i), entry) > 0) {
                break;
            }
        }
        mEntries.add(i, entry);
        return i;
    }

... ...

}
```

这个数据结构内部还有一个 Entry 类，可以看到 Entry 中不光有 sbn，还有好几个 view，这几个 view 就是 SystemUI 中一个通知的 UI 元素了。然后它的插入啊，删除啊都通过一个 Entry 的 ArrayList 来实现的。回到 addNotificationViews 中，后面有一个 inflateViews

```java
    protected  boolean inflateViews(NotificationData.Entry entry, ViewGroup parent) {
        int minHeight =
                mContext.getResources().getDimensionPixelSize(R.dimen.notification_min_height);
        int maxHeight =
                mContext.getResources().getDimensionPixelSize(R.dimen.notification_max_height);
        StatusBarNotification sbn = entry.notification;
        // oneU 是通知 normal view
        // large 是通知 big view
        RemoteViews oneU = sbn.notification.contentView;
        RemoteViews large = sbn.notification.bigContentView;
        if (oneU == null) {
            return false;
        }

        // create the row view
        LayoutInflater inflater = (LayoutInflater)mContext.getSystemService(
                Context.LAYOUT_INFLATER_SERVICE);
        // new 通知的容器（notiifcation row），这个容器是模板来的。
        // 这个是用来装通知的 content view 和 big content view 的。
        // 这里注意一点：LayoutInflater 的第二参数 ViewGroup 如果不是 null 的话，那么 new 出来的 view 
        // 会自动 add 到传递的 ViewGroup 中，所以这里在 new 出 notification row 后，
        // 就自动添加到前面传递过来的 mPile 中去了（这个是通知面板中显示通知的容器）。
        View row = inflater.inflate(R.layout.status_bar_notification_row, parent, false);

        // for blaming (see SwipeHelper.setLongPressListener)
        row.setTag(sbn.pkg);

        workAroundBadLayerDrawableOpacity(row);
        View vetoButton = updateNotificationVetoButton(row, sbn);
        vetoButton.setContentDescription(mContext.getString(
                R.string.accessibility_remove_notification));

        // NB: the large icon is now handled entirely by the template

        // bind the click event to the content area
        ViewGroup content = (ViewGroup)row.findViewById(R.id.content);
        ViewGroup adaptive = (ViewGroup)row.findViewById(R.id.adaptive);

        content.setDescendantFocusability(ViewGroup.FOCUS_BLOCK_DESCENDANTS);

        // 如果通知设置了点击动作，那么设置到通知的 view OnClick 时间中去
        PendingIntent contentIntent = sbn.notification.contentIntent;
        if (contentIntent != null) {
            final View.OnClickListener listener = new NotificationClicker(contentIntent,
                    sbn.pkg, sbn.tag, sbn.id);
            content.setOnClickListener(listener);
        } else {
            content.setOnClickListener(null);
        }

        // TODO(cwren) normalize variable names with those in updateNotification
        View expandedOneU = null;
        View expandedLarge = null;
        try {
            // 把通知的 content view（RemoteViews）加入到 notification row
            expandedOneU = oneU.apply(mContext, adaptive, mOnClickHandler);
            // 如果通知有 big content view（RemoteViews） 也一起加入 notification row
            if (large != null) {
                expandedLarge = large.apply(mContext, adaptive, mOnClickHandler);
            }
        }
        catch (RuntimeException e) {
            final String ident = sbn.pkg + "/0x" + Integer.toHexString(sbn.id);
            Slog.e(TAG, "couldn't inflate view for notification " + ident, e);
            return false;
        }

        if (expandedOneU != null) {
            SizeAdaptiveLayout.LayoutParams params =
                    new SizeAdaptiveLayout.LayoutParams(expandedOneU.getLayoutParams());
            params.minHeight = minHeight;
            params.maxHeight = minHeight;
            adaptive.addView(expandedOneU, params);
        }
        if (expandedLarge != null) {
            SizeAdaptiveLayout.LayoutParams params =
                    new SizeAdaptiveLayout.LayoutParams(expandedLarge.getLayoutParams());
            params.minHeight = minHeight+1;
            params.maxHeight = maxHeight;
            adaptive.addView(expandedLarge, params);
        }
        row.setDrawingCacheEnabled(true);

        applyLegacyRowBackground(sbn, content);

        row.setTag(R.id.expandable_tag, Boolean.valueOf(large != null));

        if (MULTIUSER_DEBUG) { 
            TextView debug = (TextView) row.findViewById(R.id.debug_info);
            if (debug != null) {
                debug.setVisibility(View.VISIBLE);
                debug.setText("U " + entry.notification.getUserId());
            }
        }
        // 设置一下 SystemUI 的通知数据
        entry.row = row;
        entry.content = content;
        entry.expanded = expandedOneU;
        entry.setLargeView(expandedLarge);

        return true;
    }
```

要讲解的我都注释在代码里面了，这里就是稍微注意下 LayoutInflater.inflate 那里会自动把生成的 view 添加到通知面板上的通知容器中，如果第一次不注意会觉得奇怪，不知道在哪里把 notification row 加到通知面板中去的。可以看到其实 BaseStatusBar 又给通知做了一个模板套套，然后才是把应用设置（其实大多时候也是用 Notification 生成的模板）的通知的 content view 加到这个模板套套中，因为这个可以控制每一个 notification row 的大小。这样可以由系统控制最终通知显示的 UI 效果（notification row 的最后大小是由 SystemUI 决定的）。

这样通知面板上的 notification row 就弄好了，然后回到 addNotificationViews 中最后 updateNotificationIcons ，这个虽然不是 IStatusBar 的接口，但是 BaseStatusBar 中的抽象接口，留给子类实现的，BaseStatusBar 有好几这样的抽象几个，基本都是 UI 相关的（毕竟子类要实现不同的 UI 风格么）：

```java
    protected abstract void haltTicker();
    protected abstract void setAreThereNotifications();
    protected abstract void updateNotificationIcons();
    protected abstract void tick(IBinder key, StatusBarNotification n, boolean firstTime);
    protected abstract void updateExpandedViewPos(int expandedPosition);
    protected abstract int getExpandedViewMaxHeight();
    protected abstract boolean shouldDisableNavbarGestures();
```

然后我们去 PhoneStatusBar 中看看：

```java
    @Override
    protected void updateNotificationIcons() {
        if (mNotificationIcons == null) return;

        loadNotificationShade();

        final LinearLayout.LayoutParams params
            = new LinearLayout.LayoutParams(mIconSize + 2*mIconHPadding, mNaturalBarHeight);

        int N = mNotificationData.size();

        if (DEBUG) {
            Slog.d(TAG, "refreshing icons: " + N + " notifications, mNotificationIcons=" + mNotificationIcons);
        }    

        ArrayList<View> toShow = new ArrayList<View>();

        final boolean provisioned = isDeviceProvisioned();
        // If the device hasn't been through Setup, we only show system notifications
        for (int i=0; i<N; i++) {
            Entry ent = mNotificationData.get(N-i-1);
            if (!((provisioned && ent.notification.score >= HIDE_ICONS_BELOW_SCORE)
                    || showNotificationEvenIfUnprovisioned(ent.notification))) continue;
            if (!notificationIsForCurrentUser(ent.notification)) continue;
            toShow.add(ent.icon);
        }    

        ArrayList<View> toRemove = new ArrayList<View>();
        for (int i=0; i<mNotificationIcons.getChildCount(); i++) {
            View child = mNotificationIcons.getChildAt(i);
            if (!toShow.contains(child)) {
                toRemove.add(child);
            }    
        }    

        for (View remove : toRemove) {
            mNotificationIcons.removeView(remove);
        }    

        for (int i=0; i<toShow.size(); i++) {
            View v = toShow.get(i);
            if (v.getParent() == null) {
                mNotificationIcons.addView(v, i, params);
            }    
        }    
    }
```

PhoneStatusBar 中的 mNotificationIcons 是一个叫 IconMerger 的东西，这个是继承自 LinearLayout 的一个自定义的布局。叫 IconMerger 是因为它有一个功能，当通知很多（状态栏的小图标很多）的情况下，它会把显示不下的图标合并显示成一个类似 “+” 号的图标表示显示不下了。这里我们就不去细看了。然后这里也不用细说什么就是玩 ViewGroup（mNotificationIcons）添加（删除）子 view（StatusBarIconView） 而已。然后到这里 BaseStatusBar 中的 addNotificationViews 就处理完了。然后最后回到 PhoneStatusBar 的 addNotification 最后那里，如果通知设置了 fullScreenIntent 就执行相应的操作，否则展现 ticker 动画效果（效果见前面的效果的那个 ticker view）。

到这里一条通知的发送流程就走完了。然后这里是以发送一条新通知来说的，前面看到系统有判断如果发送的这条通知前面已经在系统中存在了，那么就会更新对应的数据（NMS，SBMS中）和对应的 SystemUI 中的 UI 元素（会去通知容器中去找），鉴于这篇已经很长了，后面单独开一篇来说更新的事吧。


然后最后总结一下：这里涉及到系统里面的3个模块：NMS，SBMS 和 SystemUI。其中 NMS 直接是管理通知服务的，SBMS 是界面（SystemUI）系统功能（通知等）桥接，应用通过系统功能的接口（例如 NMS）使用系统提供的一系列 UI 接口。然后这些系统接口再通过系统界面的桥接（SBMS）让界面系统（SystemUI）展现相关 UI 元素（视图和控制分工明确，可以学习一下 android 的设计）。最后我们来列下相关模块的对应的数据结构（第一次看还是有点晕的）：

<pre>
App                        --> Notification
NotificationManagerService --> mNotificationList (ArrayList<NotificationRecord>)
StatusBarManagerService    --> mNotifications (HashMap<IBinder, StatusBarNotification>)
SystemUI                   --> mNotificationData(NotificationData.Entry[ArrayList])
                            |--> StatusBarIconView(StatusBarIcon)
                            |--> notification row
                            |--> Ticker
</pre>

## 小技巧

看完上面流程分析，大家应该会发现默认状态栏的小图标和通知 content view 那个显示的图标都是用 Notification 的 icon 的，就是默认是一样的。但是有些时候想让它们不一样，没有没办法咧，仔细看上面的代码会发现方法是用的。因为 content view 在 Notification build 那里创建，而且 SystemUI 状态栏上的 StatusBarIconView 是在 SystemUI 这边创建的（NM 发送 notify 后），那么我们就有曲线救国的方法了：那就是在构造 Notification 的 content view 前把 icon 设置为 content view 想要的图标（如果不使用 Builder 就是在调用 setLatestEventInfo 前，如果使用 Builder 的话在 Builder 的接口中设），然后再改为状态栏想要的图标，最后再调用 notify 就行咯（大家看到我贴的效果图的这2个图标是不一样的了没，这里我可是没改系统实现的哦）。哎，还是上下代码比较直接：

```java
NotificationManager nm = (NotificationManager)getSystemService(
    Context.NOTIFICATION_SERVICE);
Notification n = new Notification(
    // 这里先设置通知面板那通知要显示的图标
    R.drawable.stat_sys_data_usb, 
    label, System.currentTimeMillis());
// 然后构造 content view 设置 notification row 那的图标
n.setLatestEventInfo(this, label, 
    "No. " + ID + ": info This is just for test notification bar info",
    intent);
// 然后马上把 icon 改成状态栏想要显示的小图标
n.icon = R.drawable.stat_sys_data_usb_small;
// 最后用 NM 发送通知，这样在 SystemUI 那生成的 StatusBarIcon 就和通知面板上的图标不一样了
nm.notify(ID, n);
```

