title: MiniGUI 自定义控件教程3
date: 2015-01-19 20:52:16
categories: [MiniGUI]
tags: [minigui]
---

接着上次的教程继续。之前介绍了子类化已有的控件实例的方法，现在介绍子类化类和完全自己重新开始写控件类的方法。这个2种区别就是：子类化类，其实就是OOP里的继承，继承一个已有的控件类，在其基础上作扩展。完全自己重新开始写控件类是我自己的叫法，可以理解为MFC（我个人对MFC相对来说熟悉些，就拿这个做类比了）里的继承自CObject。 这次先介绍完全自己重新开始写控件类的方法。这里我以我自己写的一个MiniGUI的扩展控件类为例子来介绍。MiniGUI里原来的有CTRL_BUTTON这个控件类。刚开始2.0.4 classic风格我觉得不怎么好看（其实是我头头觉得不怎么好看），后来折腾了下弄成fashion风格了的，BS_AUTOCHECKBOX和 BS_RADIOBUTTON已经感觉不错了，不输给台式机的那些控件库了，不过基本的Button功能和那些.net，java的Button比起来还是有一定差距。其实最主要的是我接手的一个项目里，之前负责人，弄了一个类似.net，java，winXP那样的Button，就是鼠标放上去会有外观变化的，然后全部用图片来表现Button。刚开始感觉不错，后来拿到代码一看。额的个神啊，他竟然每个窗口里使用一些CTRL_STATIC来充当Button，然后窗口相应鼠标事件，检测当鼠标移动到某个STATIC上的时候就加载不同的图片。额的个神、额的个神，光光是这些判断代码都快烦死了，而已都是一大堆、一大堆的坐标计算；最要命的是每个窗口都有。一个项目那么多个界面（窗口），要我维护这个，想整死我啊。以前用MFC起手的我，立刻想到了把这玩意封装成控件。他原意其实就是要Button好看点，好我就以这个扩展的Button为例子来说明。

## 一、功能确定

首先我把这个扩展控件取名为CTRL_BUTTONEX（”button_ex”）。ButtonEx首先就要尽量具备MiniGUI CTRL_BUTTON BS_PUSHBUTTON的基本功能，这样才能在功能上不影响使用者的使用。其次，就是美观，这也是扩展这个控件的原始目的。看看.net，java的Button，他们首先Button的背景都比较好看；其实他们支持在Button上放图标（icon），并且放了图标后还能写文本上去；最后他们Disable的状态也比较好看。所以我们基本就可以确定ButtonEx的功能了。（其实以下这些功能是经过我好几个版本的更新才得到的，其中参考了MiniGUI原来Button的实现和网上不少其他扩展控件的实现）：

 1. Button的基本功能。能发送按钮按下通知码；在WS_TABSTOP风格下能在Dialog中使用TAB键遍历焦点；在焦点状态能使用Space和Enter键执行按下操作。这些都是MiniGUI PushButton? 的基本功能。
 2. Button原有的界面表现。正常状态，按下状态，焦点状态和无效状态。新增鼠标移动到控件客户区时自动进入焦点状态。
 3. 支持图标文本混合显示方式。

## 二、概要设计

好，功能确定了，下面进行设计。首先我把这个控件取名为CTRL_BUTTONEX（”button_ex”）。然后先设计ButtonEx的最基本功能Button它有4种状态：正常（Normal），控件在没有焦点，使能情况下的状态；焦点（Focus），控件出处输入焦点的状态；点击、按下（Click），控件在焦点状态下被按下或者鼠标点击的状态；无效（Disable），控件被EnableWindow设置成无效时的状态。这4种状态的转化我在MiniGUI原来基础上新增鼠标移入客户区进入Focus状态。其他就是MiniGUI原来Button的状态转化了：Noraml就多说了。控件在tab（left、up、right、down）便利到的情况下下进入Focus状态；在Focus下，按下鼠标左键、space、enter键进入到Click状态；在Click下，鼠标左键在客户区弹起、space、enter键弹起，进入到Focus状态（并判定发送Click事件，向父窗口发送Click通知码）；如果鼠标左键在客户区之外弹起则进入Normal状态（并判定无Click事件发生，不向父窗口发送通知码）；Disable windows事件发生进入到Disable状态；Enable window事件发送进入到Normal状态。状态图转化详见下图：

!["图 1 状态转化图"](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control3/1.jpeg)
 
然后控件能在WS_TABSTOP风格下在Dialog中能被tab、left、up、right、down便利焦点。在焦点状态下按下space、enter键发送按下事件。这些靠处理MSG_KEYDOWN来实现。
最后我们来设计ButtonEx的外观表现形式。这里我经过了几个版本的升级，参考了MiniGUI原来Button的表现手法和网上一些扩展控件的表现方法，决定设计出几种不同的风格来供使用者选择。

### 1：BEXS_IMAGE

我把这个叫做图片风格。分别用4张不同的图片来表现4种状态。图片支持bmp（支持透明色）、png（支持alpha通道）、gif、jpeg（支持透明色）。外观全部交由图片负责，因此这个风格不支持显示文本。好不好看全靠外部图片PS处理。这里说明下，现在一般使用MiniGUI的色深是16bit，但是载入24bit的bmp只要PS里颜色渐变不是特别BT，不会有太大的失真。32bit带alpha通道的png在2.0.4的API下也能保留alpha通道信息。因此这种可以做出外观上不规则的Button（它响应鼠标还是以矩形来算的）。bmp、jpg在设置了透明色后也能有这种效果，不过图层边缘的渐变造成透明时的锯齿问题（这里的透明平滑算法我至今还没弄出来，现在只能在图像处理软件里弄）。最终效果如下：

!["图 2 BEXS_IMAGE效果"](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control3/2.jpeg)

### 2：BEXS_BKIMAGE

我把这个风格叫做背景图片风格。它用4张不同的图片来变现4种状态的背景。然后能在其基础上放置图标（icon），写文本，图标和文本能在Click中显示动态效果，图标能在Disable状态下表现alpha混合特效。它的背景图片模式与BEXS_IMAGE一致，支持bmp、png、gif、jpeg，也能透过alpha通道或是透明色变现出不规则形状。Icon支持的图片格式和背景图片一样。可以显示文本，在有icon的情况下显示文本。Icon和文本的动态点击效果其实只要在Focus和Click下稍微改动下Icon和文本位置形成一定的位置偏差就即可看到动态效果。至于Disable下Icon的alpha混合特效，是应用了2.0.4的newgal接口提供的高级图像处理API来完成的。方法是在Disable下把Icon以一定的alpha值，半透明的绘制在背景上，这样看上去就像背景透过了一部分Icon一样，从而达到表现Dsiable状态的效果。文本让用户设置2种不同的颜色，分别在Disable和非Disable下显示。这些其实就是.net，java里Button控件的功能。不过这个风格用好了，效果不低于.net，java的Button控件哦。不过折腾这些东西实现费了我不少时间。最终效果如下：

!["图 3 BEXS_BKIMAGE效果"](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control3/3.jpeg)

### 3：BEXS_DRAW

我把这个风格叫做编程绘制风格。顾名思意，4种状态的背景表现方式全部通过代码编程的方式绘制。其实这个风格是我学习之用的，基本上没什么实用价值，因为它的特效BEXS_BKIMAGE中都有，但是又没BEXS_BKIMAGE花哨。而且这个风格的核心颜色渐变算法我是照抄MiniGUI 2.0.4 fashion里Button的。但是在表现形式上有些不同。Normal状态基本一样，都是用2种颜色来混合渐变效果，由中间的基色调向上、下渐变至第2种颜色；不过ButtonEx里可以设置这2种颜色。Focus状态也基本一样，在外围画一圈虚线，ButtonEx也可是设置虚线的颜色。Click状态有所不同，ButtonEx里我强制用Click动态效果来表现这个状态，因为MiniGUI里用了另外2种颜色来渐变，我试了下在加Icon的情况下效果不怎么好，于是就采用动态效果来表现了。Disable状态我另外用一种颜色来进行单一填充，然后Icon alpha混合上这种颜色，类似于winXP上按钮Disable的效果。控件边框用实线画成一个带圆角的矩形，在Disable和非Disable下有2种不同颜色选择。文本和Icon同BEXS_BKIMAGE。不过这个风格还是有点好处的，就是使用者不用PS背景图片了，对于不会PS的人来说比较方便，之用去网上下一些Icon就可以了（一般网上下的Icon可以直接使用了，但是网上一般下不到Button背景图片）。其实这个风格最终效果也还是可以的：

!["图 4 BEXS_DRAW效果"](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control3/4.jpeg)

## 三、详细设计

### 1：数据结构

控件有3个风格，有些成员变量能够几个风格公用，有些是类变量（C++ Class中的static变量），有些则是实例变量（C++ Class中的普通变量）。我将BEXS_DRAW风格的一些变量设计成类变量，因为我觉得这种风格的背景基本上应该都是一样的，其它2个风格的设计成实例变量。实例变量数据结构我将其命名为BEXDATA：


```cpp
typedef struct _bexdata_st
{
    … …
} BEXDATA;
typedef BEXDATA* PBEXDATA;
```

类变量数据结构我将其命名为BEXCDATA：


```cpp
typdef struct _bexcdata_st
{
    … …
} BEXCDATA;
typdef BEXCDATA* PBEXCDATA;
```

根据上面的说明，控件一共有4中状态，因为我们需要一个变量在保存当前控件的状态，设计一个枚举类型：

```cpp
typedef enum _bexuistate_en
{
    BEXUI_NORMAL = 0,   // 正常状态
    BEXUI_FOCUS,        // 焦点状态
    BEXUI_CLICK,        // 按下状态
    BEXUI_DISABLE       // 禁用状态
        
} BEXUISTATE;
```

* 成员数据结构：

BEXS_IMAGE 和 BEXS_BKIMAGE 都需要4张图片，并且都支持图片透明色。所以设计如下变量：

```cpp
PBITMAP pbmpButton[4];  // 图片指针数组
BOOL bTrans;            // 是否使用图片透明颜色
POINT pointTrans;       // 透明像素点位置
```

这里图片控件内部只引用外部应用程序的图片，图片的加载和卸载都交由外部应用程序负责（我认为这样比较好，因为外部程序的初始化和销毁正好可以做这些事情）。bTrans表示当前控件是否要使用透明颜色，pointTrans是一个点（x，y）的变量，保存当前透明颜色在控件使用的图片中的坐标。BEXS_BKIMAGE和BEXS_DRAW能够在控件中显示文本，因为MiniGUI的基本窗口结构中就保存了窗口的标题文本，所以这里只要设计保存文本颜色的变量就够了：

```cpp
gal_pixel pixelTextNormal;      // 正常文本颜色
gal_pixel pixelTextDisable;     // 控件无效文本颜色
```

所有的风格都支持点击（BEXUI_CLICK）动态效果，设计如下2个变量：

```cpp
BOOL bClickEffect;  // 是否使用Click 状态特效
int  nClickEffect;  // Click 状态特效幅度
```

BEXS_BKIMAGE和BEXS_DRAW能够放置图标在控件上；图标支持透明色；设置摆放位置；无效状态特效；设计如下变量：

```cpp
PBITMAP pbmpIcon;       // 图标图片指针
BOOL bIconTrans;        // 是否使用图标图片透明颜色
POINT pointIconTrans;   // 透明像素点位置

int nIconLeft;          // 图标距控件客户区左端距离
int nIconDx;            // 图标距文本距离

BOOL bIconDisableEffect;            // 是否使用disable 状态特效
HDC hIconDC;                        // ICON Alpha 混合DC
unsigned char nIconDisableAlpha;    // ICON Alpha 混合值
```

* 类数据结构：

只有BEXS_DRAW风格需要用到类成员。BEXS_DRAW风格正常状态的背景由2种颜色渐变形成，从中间由一种颜色（我称为基色）纵向向顶部和底部渐变到另外一种（我称为渐变色），呈对称形状。设计2个变量：

```cpp
RGB rgbRenderNormalBase;    // 基色颜色
RGB rgbRenderNormalShade;   // 渐变颜色
```

这里用RGB变量保存，因为这里需要进行颜色渐变运算，直接用像素值保存容易产生颜色与设置有误差的情况。BEXS_DRAW风格的BEXUI_FOCUS状态在控件上加上一圈虚线表示；BEXUI_DISABLE状态背景换成另外一种颜色表示；控件具有边框。设计如下变量：

```cpp
gal_pixel pixelFocus;       // BEXS_FOCUS状态虚线颜色
gal_pixel pixelDisableBk;   // BEXUI_DISABLE状态背景颜色
gal_pixel pixelBorder;      // 边框颜色
```

### 2：接口

首先就是控件注册和卸载的接口，这个在之前的教程里已经详细说过了：

```cpp
BOOL RegisterButtonExControl (void);
BOOL UnregisterButtonExControl (void);
```

#### 实例接口：

实例接口就是消息，这个也在前面的教程说过了的。首先先确定本控件自定义消息的范围。因为MiniGUI留给用户的自定义消息范围是：0x0800 ~ 0xEFFF。一般的应用程序都会有自己的消息，所以作为提供给外部程序使用的控件应尽量避免自定义消息定义冲突，所以这里我设计从自定消息的最大范围从后向前定义（应用程序一般都是从前向后定义的）：

```cpp
#define MSG_BEXMBASE    0xEFFF
#define MSG_BEXMXX      MSG_BEXMBASE – 1
… …
```

* 1: 设置控件通用数据。
我把状态图片（pbmpButton[4]、bTrans、pointTrans）、文本颜色（pixelTextNormal、pixelTextDisable）、点击特效（bClickEffect、nClickEffect）这些变量归结到一个接口来设置，叫通用数据设置。wParam传入设置PBEXDATA，lParam传入需要设置的变量（这个32bit的变量每一个位可以代表一个设置变量的开关）：

```cpp
#define BEX_COM_IMAGE           0x00000001L
#define BEX_COM_TEXTCOLOR       0x00000002L
#define BEX_COM_CLICKEFFECT     0x00000004L
#define BEX_COM_ALL             ((BEX_COM_IMAGE)| (BEX_COM_TEXTCOLOR) | (BEX_COM_CLICKEFFECT))

#define BEXM_SETCOMDATA         MSG_BEXMBASE – 1
```

* 2: 获取控件通用数据。
和设置相对应：wParam传入获取PBEXDATA，lParam传入需要获取的变量：

```cpp
#define BEXM_GETCOMDATA     MSG_BEXMBASE - 2
```

* 3: 设置控件图标数据。
我把图标图片（pbmpIcon），图标位置，图标无效状态特效这些变量归结到一个接口来设置，叫做图标数据设置。wParam和lParam与通用数据的相似：

```cpp
#define BEX_ICON_IMAGE          0x00000001L
#define BEX_ICON_POSTION        0x00000002L
#define BEX_ICON_DISABLEEFFECT  0x00000004L
#define BEX_ICON_ALL            ((BEX_ICON_IMAGE) | (BEX_ICON_POSTION) | (BEX_ICON_DISABLEEFFECT))

#define BEXM_SETICON            MSG_BEXMBASE – 3
```

* 4: 获取控件图标数据。
和设置对应：wParam传入获取PBEXDATA，lParam传入需要获取的数据：

```cpp
#define BEXM_GETICON  MSG_BEXMBASE – 4
```

#### 类接口：

类接口就是函数了。我这里设计的和上面实例的相似。 

* 1: 设置类数据。
PBEXCDATA传入的是设置的变量指针，DWORD传入32bit的设置变量类型：

```cpp
#define BEX_CDATA_RENDER    0x00000001L
#define BEX_CDATA_DEFAULT   0x00000002L
#define BEX_CDATA_ALL       ((BEX_CDATA_RENDER))

int BexSetCData (const PBEXCDATA pSetCData, DWORD dwMask);
```

这里我把目前所有的类变量都统一到一个开关里设置了（测我的测试使用来看，这是比较方便的）；BEX_CDATA_DEFAULT是把所有的类变量设置成默认值。

* 2: 获取类数据。
这个和设置对应，PBEXCDATA传入获取的变量指针，DWORD传入32bit的获取变量类型：

```cpp
int BexGetCData (const PBEXCDATA pSetCData, DWORD dwMask);
```

#### 通知码：

据我的理解，一般的消息是针对控件自己的，通知码是针对别的控件的（一般是父控件）。

* 1: 点击通知码。
在控件处于BEXUI_CLICK状态下在客户区弹起鼠标左键或是在控件BEXUI_CLICK状态弹起enter、space按键发送，表示控件被按下：

```cpp
#define BEXN_CLICKED    0
```

目前就暂时设置这一个通知码，感觉作为Button来说就这个通知码用得最多，就先设计这一个了。

这里以一个Button功能的扩展控件说明如何完全自己写MiniGUI的控件。本次教程先介绍ButtonEx的设计，下次将介绍ButtonEx的具体实现。从1月14号起的头，拖到30号才写完，怎能用茶几、杯具来形容 -_-|| 。

参考资料：飞漫MiniGUI编程指南2.0.4

## 代码下载
[下载地址]("http://download.csdn.net/detail/mingming_killer/4045894")

