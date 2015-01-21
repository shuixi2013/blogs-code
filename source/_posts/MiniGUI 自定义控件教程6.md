title: MiniGUI 自定义控件教程6
date: 2015-01-21 16:07:16
tags: [minigui]
---

接着上次的教程继续。之前就已经介绍完MiniGUI 2.0以前本人掌握的自定义控件的方法了（3.0的好像不太一样了呢，目前本人还没研究过）。恩，让我们来回顾下先：

1. 对已经创建了的控件实例进行子类化。
2. 对某个控件的子类进行子类化；这个又可以分为针对某个已有的控件（继承父类，类似之前的SLEditEx），重新自己写（继承自最顶层的DefaultControl，类似之前的ButtonEx）。

这里主要是把ControlEx工程中剩下的2个自定义控件RollShow和ProgressBarEx的实现过程也介绍下，这里都是采用第2种方法。

## 一、功能确定

滚动控件，顾名思义，就是像街上某些广告牌一样，能将一条字符串信息在指定的空间内滚动的显示。我这里是学习目的，所以功能上就简化些：

1. 控件自定判断当前的字符能否在当前的矩形空间里显示完全，能则正常显示，不能则滚动显示。
2. 滚动只要从右到左滚动就可以，能够持续的滚动，直到显示的字符串改变能够在空间内完整显示为止。

最终效果见下图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control6/1.jpeg "图 1 CTRL_ROLLSHOW效果图")

## 二、概要设计

我把控件取名为CTRL_ROLLSHOW，自己重新写（继承顶层的DefaultContorl）。因为它主要就是一些显示功能，MiniGUI原有的控件的一些功能也不能有效的“帮上什么忙”，所以选择自己重新写，这样灵活性还高些。

## 三、详细设计

### 1：数据结构

ROLLSHOW的主要是用来显示字符串，我设计成控件只保存当前显示字符串的指针，这样外部可以方便的修改，而且控件也无需管理字符串的缓冲空间。滚动的效果用定时器来实现，本来我是想设计成所有的ROLLLSHOW控件只创建一个定时器来管理滚动效果的，不过我发现这样以我目前的水平来实现有些困难。所以我弄成每一个ROLLSHOW实例都自己创建一个定时器，这里要注意下，MiniGUI有限定thread下一个应该用程序最多只能创建16个定时器（这个和MiniGUI的内部实现有关）。不过其实也不要紧，因为一般这个ROLLSHOW控件在显示的界面中不会创建好很多个的，所以这个方案也凑活行得通了^_^ 。用一个int来保存滚动的速度（定时器的间隔）。然后用2个变量分别保存当前的文字显示颜色和背景颜色。最后用一个RECT变量来保存当前文本滚动的区域：

```cpp
// RollShow 数据数据结构
typedef struct _rsdata_st
{
    char* pstrText;         // 显示字符串指针
        
    int nStep;              // 滚动步长
    int nTimerSpeed;        // 定时器时间
        
    gal_pixel pixelText;    // 文本颜色
    gal_pixel pixelBk;      // 背景颜色
        
        
    RECT rcText;            // 文本显示区域
        
} RSDATA;
typedef RSDATA* PRSDATA;
```

### 2：接口

注册和卸载的接口，老规矩了：

```cpp
BOOL RegisterRollShowControl (void);
BOOL UnregisterRollShowControl (void);
```

消息为什么定义成下面这个样子，也是老规矩了的，25*2是因为之前已经有了2个自定义控件了。

```cpp
#define MSG_RSMBASE     (0xEFFF-25*2)
#define MSG_RSMXX       MSG_RSMBASE – 1
```

* 1：设置控件数据。这个和ButtonEx的设计思路一样了：

```cpp
#define RS_TEXT         0x00000001L
#define RS_COLOR        0x00000002L
#define RS_SPEED        0x00000004L
#define RS_ALL          ((RS_TEXT) | (RS_COLOR) | (RS_SPEED))

#define RSM_SETDATA     MSG_RSMBASE – 1
```

* 2：获取控件数据。和设置相对应：wParam传入获取PRSDATA，lParam传入需要获取的变量：

```cpp
#define RSM_GETDATA     MSG_RSMBASE - 2
```

* 3：通知控件显示字符串已被修改。这个消息用于让控件重新计算当前显示字符串的空间，重新判断是否要滚动显示：

```cpp
#define RSM_TEXTMODIFIED    MSG_RSMBASE - 3
```

* 3：通知码 本控件目前不需要通知码。

## 四、功能实现

这里最主要的就是滚动的动态功能实现了。这里如果要用自己方法弄也可以，但是既然我们用的MiniGUI，咋要就要好好挖掘MiniGUI的API，这个时候你就会发现MiniGUI有几个API为ROLLSHOW的实现提供了很好的解决办法。一个是GetTextExtent()、一个是DrawText()。GetTextExtent能够计算出给定字符串在指定的DC中输出所需要的空间大小，这个API可以用来实现判断当前字符串是否需要滚动显示的功能。

DrawText 是在给定的一个RECT空间内按照预定的几种对齐格式进行文本输出。好了，先来让我们想象一下——拿一张白纸，然后再拿一张上面写了字的白纸；然后以一定的速度拿着写了字的白纸从没有字的白纸上从右到左的移动。OK，我们看到了什么——这个就是我们要的滚动效果啊。那张没字的白纸就相当于控件的客户区，有字的白纸就相当于DrawText中指定的RECT空间，我只要在定时器中以一定的速度改变这个RECT的位置就可以实现滚动功能了。我这里只是调用这些上层的API而已，剩下的事情就交给MiniGUI解决吧 ^_^。想象能力差点的，请看下面的图，黑色的矩形表示控件客户区，红色的矩形表示显示文本的DrawText的RECT空间：

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control6/2.jpeg "图 2 滚动效果实现图")

## 五、注意问题

那些自定控件实现方法需要注意的问题我前面的教程已经说过了，这里说下专门针对实现本控件需要注意的问题吧。我认为这里需要注意点的问题，第一个前面的说的定时器数量的问题，我用的MiniGUI版本的thread模式最多只能同时存在16个，这个在使用的时候需要注意，不过创建太多的本控件实例就没啥问题。 第二个就是控件的背景透明问题。如果是父窗口没有使用背景图片的，就把控件的背景颜色设置成父窗口的背景颜色就行了。如果父窗口使用了背景图片的，也好办，使用CreateWindowEx创建控件实例并指定WS_EX_TRANSPARENT风格也可以解决问题。

参考资料：飞漫MiniGUI编程指南2.0.4

## 代码下载
[下载地址]("http://download.csdn.net/detail/mingming_killer/4045894")

