title: MiniGUI 自定义控件教程7
date: 2015-01-21 17:48:16
categories: [MiniGUI]
tags: [minigui]
---

接着上次的教程继续。这次给大家介绍的是界面美观的进度条控件。它功能上和MiniGUI原有的进度条控件（CTRL_PROGRESSBAR）是一样的（其实进度条也就是那些功能，哪还能整出别点什么花样哦）。

## 一、功能确定

1. 要具有MiniGUI原有进度条控件的所有功能，像设置范围，设置步进值，设置当前位置等；垂直、水平风格；还有一些通知码。因为本控件的主要目的是美观，但是这要保证实用性，所以原有的功能必需要有。
2. 可以设置2套图片，分别用来表示进度条本身和背景。每套图片分为3个部分：头、身体、尾。头和尾可有，可无；背景也是可有，可无。图片格式支持bmp、gif、jpg、png（jpg、png依赖于MiniGUI的配置）。

最终效果见下图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control7/1.jpeg "图 1 CTRL_PROGRESSBAREX效果图")

## 二、概要设计

我把控件取名为CTRL_PROGRESSBAREX，自己重新写（继承顶层的DefaultContorl）。虽然ProgressBarEx需要MiniGUI原有控件的功能，但是我认为实现这些功能的代码量都不大；并且自己重新写能有更高的灵活性，权衡了一下我还是选择自己重新写。其实这里主要工作在于用图片表现控件的外观上了。

## 三、详细设计

### 1：数据结构

为了实现ProgressBarEx最基本功能，用4个int型来保存当前控件的进度的最大值、最小值、当前位置和步进值。控件有2套图片，每套3张，一共是6个bitmap指针（这里采用和之前ButtonEx同样的方式，控件只保存指针）。然后是用一个BOOL变量保存是否实现百分比文本，用2个gal_pixel变量保存进图条超过文本和没超过时的2种不同的颜色。最后是我在测试的时候发现的问题，在有些平台上进度条重绘会闪烁，所以我增加了一个条件编译是否使用内存DC（绘图缓冲）。ProgressBarEx的数据结构如下所示：

```cpp
// ProgressBarEx 数据数据结构
typedef struct _pbarexdata_st
{
    int nMin;               // 最大值
    int nMax;               // 最小值
    int nPos;               // 当前位置
    int nStep;              // 步进值
        
    PBITMAP pbmpHead;       // 进度条头部图片指针
    PBITMAP pbmpBody;       // 进度条身体图片指针
    PBITMAP pbmpTrail;      // 进度条尾部图片指针
        
    PBITMAP pbmpBkHead;     // 进度条背景头部图片指针
    PBITMAP pbmpBkBody;     // 进度条背景身体图片指针
    PBITMAP pbmpBkTrail;    // 进度条背景尾部图片指针
         
    BOOL bShowPrecent;      // 是否显示百分比
    gal_pixel pixelOff;     // 进度还没到文本区域时文本的颜色
    gal_pixel pixelOn;      // 进度条已经达到文本区域时文本的颜色
        
    HDC hMemDC;             // 绘图内存DC 
        
} PBAREXDATA;
typedef PBAREXDATA* PPBAREXDATA;
```

### 2：接口

注册和卸载的接口，老规矩了：

```cpp
BOOL RegisterProgressBarExControl (void);
BOOL UnregisterProgressBarExControl (void);
```

消息为什么定义成下面这个样子，也是老规矩了的，25*3是因为之前已经有了3个自定义控件了。

```cpp
#define MSG_PBEXMBASE   (0xEFFF-25*3)
#define MSG_PBEXMXX     MSG_PBEXMBASE – 1
```

* 1：设置控件数据。这个和ButtonEx的设计思路一样了：

```cpp
#define PBEX_IMAGE      0x00000001L
#define PBEX_PRECENT    0x00000002L
#define PBEX_ALL        ((PBEX_IMAGE) | (PBEX_PRECENT))

#define PBEXM_SETDATA   MSG_PBEXMBASE – 1
```

* 2：获取控件数据。和设置相对应：wParam传入获取PBAREXDATA，lParam传入需要获取的变量：

```cpp
#define PBEXM_GETDATA   MSG_PBEXMBASE - 2
```

* 3：以下这些是与MiniGUI原有进度条控件消息接口相对应的，具体的可以查阅MiniGUI的API手册：

```cpp
#define PBEXM_SETRANGE      MSG_PBEXMBASE - 3
#define PBEXM_SETSTEP       MSG_PBEXMBASE - 4
#define PBEXM_SETPOS        MSG_PBEXMBASE - 5
#define PBEXM_DELTAPOS      MSG_PBEXMBASE - 6
#define PBEXM_STEPIT        MSG_PBEXMBASE - 7
```

### 3：通知码

这里也是和MiniGUI原有的进度条控件的通知码相对应的：

```cpp
#define PBEXN_REACHMAX 	 1
#define PBEXN_REACHMIN 	 2
```

### 4：风格

这里也是和MiniGUI原有的进度条控件的风格相对应： -_-||

```cpp
#define PBEXS_NOTIFY        0x00000001L
#define PBEXS_VERTICAL      0x00000002L
```

## 四、功能实现

这里主要说说如何用图片正确的表现进度条外观了。我这里设计成进度条的外观由3个部分组成：头、身体、尾；每个部分由一张图片组成；其中头和尾可有，可无；由于进度条的特性，身体的图片只要能表现本进度条步进的最小单位即可。这样整个控件外观都由图片来表现，图片好的话，控件就会很漂亮。在绘制控件的时候，主要就是处理用图片正确的拼凑当前进度，这个要注意以下几个问题：

* 1: 背景。控件的背景是可有、可无的。没有当然就不要说什么了。如果有的话，背景是不管当前进度如何，都是显示100%的，这个很好理解。可以看看上面效果图的第2个控件实例。

* 2: 进度条身体的拼凑。这个首先来看看没有头、尾的情况。没有头、尾，那进度条就只有身体的图片组成，这个主要就是计算当前进度的区域长度（垂直的是高度），然后这个长度，需要多少张当前的身体图片来拼凑。有头、尾的话，就把头和尾的图片贴上去，然后在计算剩下的部分需要多少身体图片。可以参照下面的图，应该就比较好理解了：

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control7/2.jpeg "图 2 进度条身体拼凑")
 
* 3: 透明。这个问题其实在ButtonEx中已经讨论过了。我这里不提供非png格开启的透明颜色的选项。因为如果要用bmp或是jpg的话，请把图片背景弄成窗口背景颜色就行了。如果窗口背景是图片的话，您还是用png吧 -_-|| 。这里也顺带说说，为什么要把这些图片弄成3张。其实弄成一张也可以，而且这样还方便PS处理。但是这样做有个问题，就是集合在一张里的话，就要用MiniGUI的图像处理API去分割图片；这在16bit的运行环境下处理32bit的png的话，就造成透明颜色的失真。这个也是在ButtonEx中说过了的。

## 五、注意问题

这里也没什么需要注意的问题了，因为需要注意的地方之前的教程都说过了。这里顺带说下，之前说的那个画面闪烁的问题。可以使用内存DC缓冲解决；当然代价就是多一点点系统内存开销。这个可以根据实际情况是否开启（开关条件编译宏即可）。因为不同的硬件平台不一样，有的会闪烁，有的就不会。

到这里我就全部介绍完我之前自己写的那些MiniGUI自定义控件了。目前先到这里吧，看看以后还有没有需要更新的地方了。如果对之前我写的控件有什么好的改进建议或是开发别的MiniGUI控件，欢迎和我一起讨论 ^_^。

参考资料：飞漫MiniGUI编程指南2.0.4

## 代码下载
[下载地址]("http://download.csdn.net/detail/mingming_killer/4045894")

