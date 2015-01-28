title: Android 4.2 SystemUI 最近任务列表分析
date: 2015-01-28 19:31:16
categories: [Android Framework]
tags: [android]
---

Android 4.1 SystemUI 的最近任务列表和 4.2 的区别还是很大的。4.1 的是一个 BAR_PANEL，是一个 view 添加到 window manager 里面 show 出来的。4.2 变成了依附一个 Activity 了（其实都是 window）。功能上的影响的话，由于 4.2 最近任务列表是 activity ，所以当前的 activity 会 onPause。视觉上的不一样， 4.2 如果在非桌面调用最近任务列表，会有一个当前 activity 缩小到最近任务列表的动画（旁边的 icon 也会有一个跳动的动画）。还有一点，4.2 最近任务列表 activity 设置了 wallpaper 属性，所以背景是 wallpaper，不再像以前一样能透过看下当前的任务了，另外如果当前设置的是动态壁纸，背景就是黑的（不仅仅动态壁纸在 wallpaper 属性的 activity 背景有问题，在代码里都地方检测了，如果是动态壁纸，就强制设为黑色）。列下源码位置： framework/base/packages/SystemUI/src/com/android/systemui/recent 这个文件夹下， 对应的 xml 文件在代码里也能看得出。

## UI
4.2 虽然包了一层 activity 但是主要功能还是和 4.1 类似，在原来的 RecentsPanelView.java 中。

* **RecentsPanelView.java**
这个 view 就是最近任务列表的主界面。里面主要是有一个自定义的 ScrollView 。这个 view 里面会负责完成一些动画（上面说那个图标跳动动画，注意那个当前任务缩小的动画不是在这里做的）， 自定义 ScrollView adapter 的实现（这个自定义 ScrollView 有点仿 ListView，使用到了 adapter），一些后台数据加载的回调（刷新界面），一些交互回调的实现（点击、长按、滑动删除 item），提供一些 show、hide、refresh 接口给 activity 使用。

* **RecentsVerticalScrollView.java**
这个自定义 ScrollView 主要是为了下面一些定制：滑动删除，类似 ListView 的优化（adapter 机制），自定义的 overscroll fade 效果。滑动删除是类似 webos 的卡片式任务列表的操作。android 为了做这个效果，在 xml 中把 android:clipChildren 和 android:clipToPadding 这2个属性设置为 false 了。这样 ScrollView 的 item 就能超出自身的 view 范围，能够“滑”出去了。还有一个 RecentsHorizontalScrollView.java 和这个是类似的，只不过是横屏的而已。

* **RecentsScrollViewPerformanceHelper.java**
这个是给自定义的 ScrollView 画 over scroll 的 fade 效果的。在 drawCallback 里面。不过好像也有一些绘制上的优化，还没具体分析。

* **StatusBarTouchProxy.java**
这个东西挺无聊的。android 在 RecentsPanelView 的顶部（navigation bar 在底部）加了一条和 navigation bar 一样高的 view（看不到，但是也可以点得到），然后把在这个 view 上 touch event 转发到 navigation bar 上去，这样点击上面也就相当于点击 navigation bar。所以叫 touch proxy，是挺无聊的。

## 数据

未完，待续 ... ... （估计不会填这个坑鸟）


