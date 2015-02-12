title: Android Gesture 使用简介
date: 2015-01-25 23:03:16
updated: 2015-01-25 23:03:16
categories: [Android Development]
tags: [android]
---

Gesture 中文名字叫：手势。就是类似一些浏览器（chrome、Firefox、傲游等）里用鼠标快速的画出一些图像（手势），然后根据这些图像执行某些功能（例如：前进、后退、刷新等）。Android 里自带了手势的功能，只要 import android.gesture 下的一些包就可以使用了。先介绍下相关的类：

## 手势相关的类

* **GestureOverlayView**
这个是一个 view。手势需要在一个 view 里画出来，android 里已经帮我们提供了专门画手势的 view 了。这个 view 一般是放在别的 view （使用手势的应用程序的界面）的上面的。或者说别的 view 要放到 GestureOverlayView 里面。这样就可以在应用程序的整个界面上画手势了。通过 GestureOverlayView 可以得到一个 Gestrue 。

* **GestureLibrary**
这个是保存 Gesture 的集合。就和它的名字一样， GestureLibrary 里包含了很多个 Gesture ，然后通过 GestureLibrary 来操作 Gesture ：添加手势、删除手势、匹配手势、加载手势、保存手势等。

* **GestureLibrarys**
名字上和 GestureLibrary 很像。这个类的方法全部是 static 的。一般是通过 GestureLibrarys 得到 GestureLibrary 。一般是通过打开文件之类的， Gesture 可以别保存到文件里。

* **Gesture**
一个手势就是一个 Gesture 。它好像是一个路径，通过保存一些系列点（好像有个最大值）包记录这个路径。然后比较的时候就通过这些点，来进行比较。 Gesture 可以分为单笔画（手指一抬起就结束，只能画一次的）和多笔画类型（在一个短的时间可以画多次）。我在这里暂时只讨论单笔画。

## 使用流程

一般的使用流程如下：

1. 添加 GestureOverlayView 到你需要使用手势的 Activity 中。然后可以设置一些 view 的属性（例如手势的颜色、笔画、监听函数等）。
2. 通过已经保存的文件得到 GestureLibrarys （文件不存在的话，则会创建一个新的文件），然后通过 GestureLibrarys 得到 GestureLibrary。
3. 通过 GestureLibrary 加载文件中的 Gesture 数据。
4. 在 GestureOverlayView 中画 Gesture，然后可以把这个 Gesture 添加到 GestureLibrary 中（添加完成后，可以保存到文件）。
5. 最后可以把从 GestureOverlayView 中得到的 Gesture 和 GestureLibrary 这已有的 Gesture 进行比较（识别），然后根据 Gesture 的定义执行不同的操作。

## 使用方法

### GestureOverlayView 

* 首先要在 Activity 里添加 GestureOverlayView ，在 Activity 的 xml 中添加如下代码：（别用 eclipse 自带的拖界面的方式，很不好用，手动写比较好）

代码：

```html
// 注意如下 xml 的放置方式，把 activity 别的 view 放到 GestureOverlayView 的里面。
// 这样的话就 GestureOverlayView 就覆盖了整个界面了，这样就可以在整个界面画手势。
// 否则的话 GestureOverlayView 不会覆盖整个界面的。
<android.gesture.GestureOverlayView
    android:id="@+id/match_gov" 
    android:layout_width="fill_parent"
    android:layout_height="fill_parent" 
    android:layout_weight="1.0"
    android:gestureColor="#FF00FF00"
    >

    <ListView 
        android:id="@+id/gesture_lv" 
        android:layout_width="wrap_content" 
        android:layout_height="fill_parent" 
        >
    </ListView>

</android.gesture.GestureOverlayView>
```

* 然后可以设置 GestureOverlayView 的一些属性，例如：

1: void setGestureStrokeType(int gestureStrokeType) : 这个是设置 Gesture 笔画的。可以设置为：

```java
// 多笔画
GESTURE_STROKE_TYPE_MULTIPLE

// 单笔画
GESTURE_STROKE_TYPE_SINGLE
```

笔画是指：从手指在触屏上开始画，到手指离开触屏，这个算一笔。多笔画就是说 Gesture 可以由多个笔画组成。单笔画那就是 Gesture 只能由一笔组成啦。

2: void setGestureColor(int color) : 设置 Gesture 的颜色。

3: void addOnGestureListener(GestureOverlayView.OnGestureListener listener) : 这个是设置监听 GestureOverlayView 事件相应函数，其中参数是一个接口，需要实现的接口有：

```java
// 暂时不太清楚这个接口是什么时候调用的
public abstract void onGesture (GestureOverlayView overlay, MotionEvent event)

// 当 gesture 别取消的时候调用，我目前还不太清楚什么情况下 gesture 会被取消
public abstract void onGestureCancelled (GestureOverlayView overlay, MotionEvent event)

// 当完成一个 gesture 的时候调用，最有用的一个接口。这个接口需要注意一点：
// 当你设置 GestureOverlayView 是多笔画的时候，没画完一笔这个接口都会被调用一次，
// 要想分辨多笔画 gesture 是否完成，可以通过 Gesture 的接口 getStrokesCount() 查询当前 gesture 的笔画来确定
public abstract void onGestureEnded (GestureOverlayView overlay, MotionEvent event)

// 当一个 gesture 开始画的时候调用，比较有用的一个接口
public abstract void onGestureStarted (GestureOverlayView overlay, MotionEvent event)
```

4: 还有一些别的属性可以设置，可以自己去查看 android 的 sdk docs。

### GestureLibrarys

这个全都是 static 的方法，主要是用来读取 Gesture 文件的。它的主要方法是：

```java
// 通过一个 java file io 来得到 GestureLibrary
static GestureLibrary fromFile(File path)

// 通过一个文件路径来打开文件得到 GestureLibrary
static GestureLibrary fromFile(String path)
```

### GestureLibrary
从 GestureLibrarys 得到 GestureLibrary 后，就可以操作 Gesture 了：

* **abstract boolean load() :**
这个接口用来从文件中加载 Gesture 数据。注意从上面的 GestureLibrarys 得到 GestureLibrary，但是文件里的 Gesture 数据并没有马后加载到内存中的。需要调用这个接口才会真正的加载。

* **void addGesture(String entryName, Gesture gesture) :**
这个接口用来向 GestureLibrary 中添加一个 Gesture 。在 GestureLibrary 中 Gesture 的保存形式是以类似于 Map 的形式来保存的。一个 Gesture 对应一个名字，如果有重复的则会忽略（所以说如果要覆盖原有的一个 Gesture，你需要把原来的删掉才行）。 可以通过 GestureOverlayView 的 getGesture() 函数获取在 GestureOverlayView 中绘制的 Gesutre，然后通过这个接口，加入到 GestureLibrary 中。

* **void removeEntry(name) :**
从 GestureLibrary 中删除一个 Gesture，在上面那个接口也提到了。

* **abstract boolean save() :**
将 GestureLibrary 中的 Gesture 保存到文件中。注意使用 addGesture 不会马上保存到文件中的，要调用这个接口才会真正保存到文件中。

* **Set<String> getGestureEntries() :**
返回 GestureLibrary 中所有的 Gesture 的名字。

* **ArrayList<Gesture> getGestures(String entryName) :**
通过 Gesture 的名字返回 Gesture 。目前我还是没怎么搞明白为什么它会返回一个数组，而不是单一个 Gesture ，反正一般来说用这个数组的第一个元素就行了。 sdk 里也没说明。

* **ArrayList<Prediction> recognize(Gesture gesture) :**
用给定的 Gesture 配置 GestureLibrary 中保存的 Gesture。这个接口返回的是一个 Prediction 类型的数组。这个 Prediction 有2个比较重要的属性：

```java
// 匹配的 Gesture 的名字
String name;

// 匹配的分数。越高就表示匹配程度越高。一般来说低于 1.00 的就是连大致形状都不像的。
// 一般来说你需要设定一个判定的分数线。至于是多少就需要自己调试一下了。
doulbe score
```

### Gesture
Gesture 是一个路径，通过一系列点来表示。如果你画类似的 Gesture，但是路径的顺序相反的话，那就是不同的 Gesture 来的。Gesure 的操作一般都集中在 GestureLibrary 里了。 Gesure 本身目前来说我用到的就一个接口：

* **Bitmap toBitmap(int width, int height, int inset, int color) :**
将 Gesture 转化为指定大小、颜色的 Bitmap 。这样的话就可以方便的在 ImageView 这种 view 中显示出当前 Gesture 的形状。

## 小结
使用 Gesutre 的时候注意要申请写 sdcard 的权限，不然 Gesture 不能正常保存的。附件里有一个例子，可以自定义3个 Gesture 来启动 android 自带的3个应用（点击相应的 Gesture 进行编辑），附上运行效果：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/gesture-sample/1.png)

## 附件
[gesture sample](http://s.yunio.com/H73fAq "gesture sample")

