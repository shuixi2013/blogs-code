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

![normal view 模式](http://7u2hy4.com1.z0.glb.clouddn.com/android/Notification-base/normal_notification_callouts.png "normal view 模式")

从 4.0 开始多了一个 big view 的模式，是这样的：

![big view 模式](http://7u2hy4.com1.z0.glb.clouddn.com/android/Notification-base/bigpicture_notification_callouts.png "big view 模式")


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

我这里偷下懒就不上效果图了（自己试一下就能看到效果了）。然后这里的风格的区别主要是在 big view 上，这几个风格分别提供了3种 big view 的风格，设置也是设置相应的 big view 的模板元素。我们来稍微看下源码，还记得上面那个 Builder 的 build 函数么，如果 mStyle 不是 null 的话，那么就调用相应 Style 的 build 函数，我们这里稍微看下 BigPictureStyle 的：

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

上面说了一些 Notification 的基本用法，也稍微分析下源代码的一些实现。这些稍微说下，应用对 NM 发送一条通知，framework 中会对应生成什么数据，然后通知的 content view 是怎么加到 SystemUI 的状态栏和通知面板中的。

## 小技巧

未完待续 ... ...


