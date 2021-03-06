title: MiniGUI 自定义控件教程2
date: 2015-01-19 20:34:16
updated: 2015-01-19 20:34:16
categories: [MiniGUI]
tags: [minigui]
---

## 控件功能确认

咋接着上次的教程继续。这次我们依托一个例子来说明如何使用MiniGUI中的第一种方法（也就是子类化已经创建的控件实例）。假设我们的例子是：某个学校的某个年级的某些班在某个时间搞了某次考试。考试过后经老师研究决定把考试成绩按班为单位分成3个分数段：差、中等、好。现在咱们就要用MiniGUI来整一个统计图来直观的显示这个3个分数段的学生比例。但是有几个班就有几个统计图啊。你可能说，我可以只画一个图然后切换数据显示不就行啦。哎呀，那么大的窗口你就忍心只放一个图么，再说了你偏要这么弄，那咋的教程也进没办法进行下去了。所以我决定在界面上放上2个统计图（其实放多少个都可以，反正代码只有写一次 ^_^）。然后再弄1个按钮在切换不同的班级数据显示。最终的效果如下图：

 ![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control2/1.jpeg)


## 控件设计

OK，首先让我们确定下要“继承”哪一个MiniGUI的控件。哎，其实这问题太明显了。我们要的是一个统计直方图，基本上控件都是自己画出来的了，MiniGUI原来控件的一些特性基本上用不到，那当然是选择继承CTRL_STATIC啦，这个控件本身没多少自己数据（占用资源少），而且如果你不设置任何信息在上面的话，就是白花花的一块画布啊，赶紧画吧。

第二次OK，让我们设计下我们这个控件的数据结构：我们需要保存3个分数段的学生人数（弄个int的3元素的数组啦）；班级的名字（弄个32个字符的char数组啦）；控件附加数据（DWORD类型，这个为什么要要，在后面的注意事项里再说）。所以我们的控件的数据结构就是：

```cpp
typedef struct _statgdata_st
{
    int nData[3];
    char strName[32];
        
    DWORD dwUserAddData;
} STATGDATA;
typedef STATGDATA* PSTATGDATA;
```

第三次OK，然后我们再来设计下我控件对外的接口。MiniGUI 采用消息机制，那和外部打交道的当然就是消息啦。注意我这里说设计的消息是指控件自定义的消息，不是MiniGUI 原有的消息。

1. 初始化。观众又要说了不是有MSG_CREATE消息么，为什么还要自己再弄一个。嘿嘿，这个在同样在后面的注意事项再说。我把它整成STATGM_INIT；不要任何参数；主要负责初始化控件数据变量。
2. 设置控件数据。这个就不多什么了，没这个你还怎么操作控件？我把它整成STATGM_SETDATA；WPARAM 传入字符串指针来设置strName；LPARAM传入int数组首地址来设置nData。
3. 设置控件附加数据。这个说了后面再说的。我把它整成STATGM_SETADDDATA；WPARAM传入DWORD的附加数据。
4. 获取控件附加数据。有了前面的设置附加数据能没这个么。我把它整成STATGM_GETADDDATA；WPARAM传入获取的DWORD指针。为啥不用SendMessage的返回值，那个是int的，DWORD在MiniGUI的定义为unsigned long 其实还是和int的有区别的。

对了，应该还有一个STATGM_GETDATA的来获取当前控件的数据值的，但是我一想反正是弄了小例子而已，大家有兴趣的自己去改我的源代码啦。所以我们的消息就是：

```cpp
#define STATGM_BASE         MSG_USER + 3000

#define STATGM_INIT         STATGM_BASE + 1
#define STATGM_SETDATA      STATGM_BASE + 2
#define STATGM_SETADDDATA   STATGM_BASE + 3
#define STATGM_GETADDDATA   STATGM_BASE + 4
```

第四次OK，控件数据结构和消息都设计好了，可以开工写了。其实C语言，MiniGUI的API的调用大家都会，我这里就是主要说一下使用这样方式自定义控件的注意事项，也算为把之前挖的坑给填了 ^_^ 。

## 需要注意的问题

### 1：如何继承父类（我们这里是CTRL_STATIC）

飞漫的编程指南里，写的是用一个全局的WNDPROC 变量来保存SetWindowCallbackProc() 的返回值来获取原来控件的过程处理函数指针。然后在新的过程处理函数消息处理过后再利用这个全局变量来跳转到父类控件的过程处理函数去执行。我个人不怎么喜欢这样方式。我的方法是用如下这段代码来获得父类控件的过程处理函数：

```cpp
WNDCLASS wcStatic;

wcStatic.spClassName =  CTRL_STATIC;
wcStatic.opMask = 0x00FF;

GetWindowClassInfo (&wcStatic);
wpOldStatic = wcStatic.WinProc;
```

其中wpOldStatic是自定义控件源文件的文件变量。我定义一个接口函数把这段代码封装起来，然后在子类化控件实例前调用。注意这个一定要获取正确不然得不到正确的继承效果的，一般来整个项目的初始化过程里调用。然后我再解释下那个opMask为啥要设置成0x00FF。这个是飞漫的API编程手册里没写清楚的地方之一（至少我看的2.0.4的没写）。这个DWORD的掩码告诉下面的GetWindowClassInfo() 函数你想要获取的控件类的信息。然后这个函数会把控件类的相关信息填入你传入的WNDCLASS指针里。不过要注意你传入的WNDCLASS的spClassName字段你要填好，不然这个函数不知道你要获取的是哪个控件类的信息。opMask掩码含义：

```cpp
#define COP_STYLE       0x0001  // 获取风格信息
#define COP_HCURSOR     0x0002  // 获取鼠标信息
#define COP_BKCOLOR     0x0004  // 获取背景颜色信息
#define COP_WINPROC     0x0008  // 获取过程处理函数信息
#define COP_ADDDATA     0x0010  // 获取附加数据信息
```

我们要的就是 COP_WINPROC(0x0008)这个啦，不过为什么设置为0x00FF咧。哎，这个就代表我以后的信息全部获取，当然也就包括控件的过程处理函数啦。

### 2：如何子类化控件实例

飞漫的编程手册里介绍的是用 SetWindowCallbackProc? () 来替换掉要子类化的控件实例的过程处理函数。没错就是用这个函数来替换，不过需要注意一个问题。你控件需要初始化吧。大家说：响应MSG_CREATE消息不就得啦。嘿嘿，这就不行了吧。大家仔细想想看，MSG_CREATE消息是什么时候发送的？是在控件创建好之后就发送了。那你是什么时候替换掉它的过程处理函数的？你别告诉我可以再控件实例创建好之前替换。你替换时候的控件句柄哪来的，不创建好后你能GetDlgItem到控件句柄？所以如果这个时候你把初始化控件变量的代码写到了MSG_CREATE里的话，你就着道了。你的控件数据是不会被初始化的，这个对于指针来说，意味着什么会C、C++的人不用我说了吧。所以我才在上面说要自己整一个初始化的消息让外部在替换的时候告诉控件：要初始化啦。 这样我就把实例化封装成一个接口函数：

```cpp
void InstanceStatGCtrl (HWND hCtrl)
{
    SetWindowCallbackProc (hCtrl, StatGraphProc);
    SendMessage (hCtrl, STATGM_INIT, 0L, 0L);
}
```

每次子类化控件实例，把该控件实例的句柄传入这个函数就可以啦。

### 3：控件数据如何实现每个实例保存一份

C++里有类和对象的概念，每个类的对象都拥有自己的数据（叫成员变量，或者叫实例变量），也有所有对象公用的数据（叫类数据，在定义类的数据时加上static即可）。可是C语言是没有真正的类的数据的，只有结构体。MiniGUI的每个控件都有2个附件数据，都是DWORD类似的其实就是unsigned long，在一般的linux上是32bit的。用于保存控件的私有数据（其实就是可以理解为每个控件实例自己的数据）。又有人问了，这只是一个32bit的数据，怎么保存各种各样的数据啊？嘿嘿，C语言忘了指针了么。如果你拿这个32bit的空间来保持数值确实不能干什么事，但是如果你是拿来保持地址呢。这个地址指向了你希望保持数据的首地址，那这就把这个问题给解决了。

我把控件的数据类型定义成一个结构体，然后在初始化话的时候给它申请内存空间，然后把地址（也就是这个结构体的变量的指针）保存到附加数据里。然后在后面的消息处理中，通过获取控件的附加数据（获取后强制转化为控件结构体指针），就能够操控每个控件实例的数据了。别了在控件销毁时释放内存哦。

但是这里还要特别强调一点。MiniGUI每个控件有2个附加数据。这不是随便用其中的一个的。其中dwAdditionalData2（附加数据2）是被MiniGUI原有的控件用掉了的，用来保存它们的控件私有数据去啦。所以像这种“继承”至MiniGUI原有控件的绝对不能使用dwAdditionalData2，不然控件就很容易出乱子，甚至程序崩溃，因为MiniGUI原来的控件也是使用指针的方式来保持的，你把它原来的弄掉了，然后它原来的代码又用这个指针干原来的事情，想想看多危险的情况啊。这个在飞漫的编程指南里有提到，但是仅仅是在44页的一个不起眼的地方用了一个小字体一笔带过而已 -_-|| 。

所以，我现在一个例子的这种情况应该使用dwAdditionalData1（附加数据1）来保存我们自定义控件的数据。不过这个附近数据还有一个用途就是应用程序的开发人员来保持一些特定应用程序的一些数据的。但是这里被我们用掉了，咋办咧。嘿嘿，这个也好办，还记得我设计的时候说设计了一个设置控件附加数据的消息么，MiniGUI 有SetWindowAdditianlData() 的API了，我为什么还要设计这个接口咧。这就是原因啦，本来设计保留给应用程序的dwAdditionalDtat1别我的控件占用掉了，我们控件的开发原则就是尽量方便外部应用程序的使用，不能让人家不能使用原有的功能啊。所以我在自定义控件的结构体里增加了一个DWORD类型的数据，然后提供结构给外部来设置和获取。不过这里一定要告诉使用控件的人，让他使用你提供的消息来设置和获取控件附加数据，而不是直接使用SetWindowAdditionalData 和GetWindowAdditionalData 。

这里再啰嗦一点：外部的应用程序不管什么情况下，都不要轻易的使用SetWindowAdditionalData2和GetWindowAdditionalData2。这个之前说过了，是保存MiniGUI 原有控件数据的指针，不是开放给外部应用程序使用的。


其他的一些通常的MiniGUI GUI编程的一些就不多说，看我的源代码相信大家都会。这次介绍子类化已有控件实例的方法。下次我将结合自己写的一些通用自定义控件介绍子类化控件类和完全自己从头开始写自定义控件类（我管某一种方式叫完全自己从头开始写，至于准不准确大家自己看着办了，话说我还没开始介绍咧 -_-||）的方法。

本人比较懒散些，所以更新慢是必然的，请大家多多包涵。飞漫的MiniGUI的3.0版本都出来了，据说和1.6/2.0 区别很多，本人这些全是在2.0的基本上研究的，哎，不知道过时了没 -_-|| 。

参考资料：飞漫MiniGUI编程指南2.0.4


## 代码下载
[下载地址](http://download.csdn.net/detail/mingming_killer/4045894)


