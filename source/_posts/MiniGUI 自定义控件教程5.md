title: MiniGUI 自定义控件教程5
date: 2015-01-21 14:40:16
tags: [minigui]
---

接着上次的教程继续。上次以ButtonEx控件的开发为例介绍了如果自己完全重新开始写控件，这次我以一个扩展单行编辑框控件为例介绍如何在原有控件的基础上扩展自定义功能（继承原有控件功能）。

## 一、功能确定

MiniGUI原来的单行编辑框控件 `CTRL_SLEdit` 除了具有编辑框的基本编辑功能外，就提供了一个限制输入字符长度的功能。没有类似MFC中CEdit限制输入类型，字符还是数字，数字还可以限制范围（不过CEdit是在输入完之后才能判断的）。不过这些功能对于应用程序还是比较有用的。于是我决定在CTRL_SLEDIT的基础上扩展这些功能：

1. 2种编辑模式：文本；数字。文本模式可以输入任意字符，数字模式只能输入0~9、+、-和小数点。
2. 文本模式提供过滤输入字符的功能，能够指定屏蔽掉特定的字符；提供反向过滤功能，就是能够指定只允许输入特定的字符。
3. 数字模式提供指定是否限定输入整数；并在此基础上提供指定输入范围功能。


## 二、概要设计

这里因为继承了 `CTRL_SLEDIT`，所有只需专注控件的扩展功能就行了。要实现以上功能，最关键的就是在 `CTRL_SLEDIT` 接受到键盘输入之后，把输入显示到屏幕上之前，进行自己的过滤算法判定；当输入的是不符合用户设定的字符则截断，不发送给父类 CTRL_SLEDIT 处理（这样它就显示不出来了）；当输入是符合用户设定的字符就发送给父类处理（这样它就能正常显示出来）。流程图如下：

![图 1 CTRL_SLEDIT流程图](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/custom-control5/1.jpeg)

* 1：EEXMODE_TEXT

文本模式能让用户指定不允许输入某些字符（过滤），还是只允许输入某些字符（反过滤）。不过注意，这里针对的是字符，而不是字符串。字符串又要麻烦很多了。这个模式我只是顺带做了一下，下面的数字模式才是比较实用的。（汉字是占2个字符(字节)，所以这里MS也不能过滤了 -_-||）

* 2：EEXMODE_DIGITAL

首先数字编辑模式就只能输入’0~9’、’+’、’-‘、’.’这些字符。在此基础上能让用户选择能否输入小数，开启的话就能输入小数；关闭的话就能输入整数。还能让用户指定数值的输入范围（闭区间）。

## 三、详细设计

### 1：数据结构

SLEditEx 的控件变量都是实例变量。控件变量数据结构我命名为EEXDATA：

```cpp
typdef struct _eexdata_st
{
    ... ...
} EEXDATA;
typdef EEXDATA* PEEXDATA;
```

首先需要一个变量能保存当前的编辑模式。这里我没有使用控件本身的风格变量，因为父类CTRL_SLEDIT把那低16bit用得差不多了，我懒得去搅和了，自己拿一个变量来保存得了：


```cpp
// 编辑模式定义
typedef enum _eexmode_en
{
    EEXMODE_TEXT,         // 文本编辑模式
    EEXMODE_DIGITAL,      // 数字编辑模式
        
} EEXMODE;

EEXMODE nMode;            // 编辑模式 
```

由于继承原有控件，所以控件数据占用了留给应用程序的dwAdditionalData1（附加数据1，忘记了怎么回事的赶快回头看教程2 ^_^），所以还要多加一个DWORD：

<pre>
DWORD dwUserAddData; // 用户控件附加数据
</pre>

文本模式需要一个标志来表示当前是过滤还是反过滤；我把过滤（反过滤）的字符存入一个静态字符数组中（不用动态链表了，那个麻烦，我把这个数组大小设置成512，应该够了）：

<pre>
char strFilterChar[EEX_MAXCHAR];    // 过滤字符串 
BOOL bFilter;                       // 过滤模式标志
</pre>

数字编辑模式也需要一个标志来表示当前是否允许输入小数；还需要2个浮点型来保存允许输入的最大值和最小值（浮点型可以表示整型的整数部分，但是整型就无法表示浮点型的小数部分了，所以要用浮点型）。如果设置的最小值大于最大值的话，我就把这种情况设计成不限制输入范围。这里我还多设计了一个范围的数据结构，其实为了下面的消息接口，因为一个消息只能传递2个参数，我把范围和允许输入小数的开关的设置放到一个接口里去了。但是这需要传递3个参数，所以我就弄了一个范围的数据结构出来，把2个参数整合成1个来传递。（其实完全可以用之前ButtonEx控件的那个钟传递参数的方法：1个参数传递控件变量指针，1个参数传递要设置的变量类型；不过这2个中传递方法我也说不上哪种好，哪种不好，我这里都用了，大家看自己喜欢了。其实MiniGUI原来的控件里也是这2种都用了的，例如：CTRL_LISTVIEW）：

```cpp
// 限定输入数字范围数据结构
typedef struct _eexrange_st
{
    double dMin;      // 最小值
    double dMax;      // 最大值
        
} EEXRANGE;
typedef EEXRANGE* PEEXRANGE;


double dMin;       // 允许输入最小值
double dMax;       // 允许输入最大值
BOOL  bInteger;    // 输入是否是整数标志
```

### 2：接口

注册和卸载的接口，之前的教程说过了的：

<pre>
BOOL RegisterSLEditExControl (void);
BOOL UnregisterSLEditExControl (void);
</pre>

消息为什么定义成下面这个样子，之前的教程也数过了的。这里为什么减去25咧。因为我的这些控件都是在一个工程里的，当然要防止消息定义冲突啦，我给之前的ButtonEx预留了25个消息。

```cpp
#define MSG_EEXMBASE    (0xEFFF-25)
#define MSG_EEXMXX      MSG_EEXMBASE – 1
```

* 1：设置控件附加数据。这个之前教程说过了的（我MS重复这句话很多次了-_-||）：

```cpp
DWORD* pdwAddData;
wParam = (WPARAM)pdwAddData;
lParam = 0L;

#define EEXM_SETADDDATA     MSG_EEXMBASE - 1
```

* 2：获取控件附加数据和设置相对应：

```cpp
DWORD* pdwAddData; 
wParam = (WPARAM)pdwAddData;
lParam = 0L;

#define EEXM_GETADDDATA     MSG_EEXMBASE - 2
```

* 3：设置当前编辑模式。

```cpp
int nMode;
wParam = (WPARAM)nMode;
lParam = 0L;

#define EEXM_SETEDITMODE    MSG_EEXMBASE - 3
```

* 4：获取当前编辑模式。和设置对应，直接用返回值获取

```cpp
wParam = 0;
lParam = 0L;

#define EEXM_GETEDITMODE    MSG_EEXMBASE - 4
```

* 5：设置数字模式下的编辑属性。包括是否允许输入小数，输入范围：

```cpp
PEEXRANGE pRange; 
BOOL bInteger;
wParam = (WPARAM)pRange;
lParam = (LPARAM)bInteger;

#define EEXM_SETDIGITALRANGE    MSG_EEXMBASE - 5
```

* 6：获取数字模式下输入范围，和设置相对应，允许输入小数标志用返回值获取：


```cpp
PEEXRANGE pRange;
wParam = (WPARAM)pRange;
lParam = 0L;

#define EEXM_GETDIGITALRANGE    MSG_EEXMBASE - 6
```

* 7：设置文本编辑模式下过滤字符：

```cpp
char* pstrFilter; 
BOOL bFilter;
wParam = (WPARAM)pstrFilter;
lParam = (LPARAM)bFilter;

#define EEXM_SETFILTERCHAR      MSG_EEXMBASE - 7
```

* 8：获取文本模式过滤字符，和设置相对应，过滤标志用返回值获取：

```cpp
char* pstrFilter; 
wParam = (WPARAM)pstrFilter;
lParam = 0L;

#define EEXM_GETFILTERCHAR      MSG_EEXMBASE - 8
```

### 3：通知码

* 1：输入非法通知码。当输入不符合设置的规则时发送。这里定义成3是因为父类把0~2用掉了，这个要注意啊。

<pre>
#define EEXN_INPUTINVALID 3
</pre>

## 四、功能实现

### 1：文本模式过滤

这个功能实现起来很简单。就是拿当前输入的字符去和设置的过滤字符想比较可以了，在过滤字符里就不符合，不在就符合。这里说一下我在判断函数（`EEX_ValidateInput`）用到的一些strex开头的函数（像strexFind()、strexInsert()等）。这些strex开头的函数也是我自己写的，主要用于实现像CString里的一些常用的字符串操作功能，C语言原来的字符串操作函数很多功能没有，只好自己写了。具体的大家可以去看工程里include里的StringEx.h，这里我是打包成了另外一个库了的，里面还包括了一些常用的数据结构的C语言实现（链表、队列等）。大家其实完全可以自己实现的，我个人感觉我写的也不是特别好 -_-||，又需要的话我再放代码上来吧。

### 2：数字模式过滤

这里我判断的方法是：

1. 首先拿当前输入和”0123456789-+.”比较，也就是说首先只允许输入0~9、-、+、.（如果设置了只能输入整数的话，则不允许输入.）。
2. 对于通过了1的字符来说，只要再同时满足一下2个条件就是有效的数字： "-", "+"只能出现一次并且只能在第一个位置。 "."只能出现一次。
3. 对于通过了2的字符来说，只需要再验证当前输入的数字是否在设置的范围内就完成了判断了。

我感觉我的方法应该还是对的，不知道大家还什么更好的方法没有 ^_^ 。不过这里有个小问题：就是在最后判断范围的时候，在判断下限的时候我使用整数判断。因为我是在显示到屏幕之前进行判断的，如果是小数的话会有问题。例如下限是1.5，本来用户是想输入1.7的，先输入的是1；如果是小数判断的话，1比1.5要小，不在范围内，就会被判定成不合法，从而屏蔽掉输入。所以这里我判断下限的时候采用了整数判断的方法，会造成下限范围设置成小数时，只能限制整数的问题，但总比不能输入要好，不知道大家有什么好办法解决没有。

## 五、注意问题

其实前面那些看代码也能明白，现在说说最关键的，就是继承父类控件需要注意的一些问题（同时也包括了实现本控件需要注意的一些问题 ^_^）。

* 1：调用父类过程处理函数

就是在自己的控件过程处理函数最后，把自己不处理的消息都转发给父类控件过程处理函数处理。这样才能享有父类控件原有的功能，而且这样才能叫得上“继承啊“。代码如下：

```cpp
WNDCLASS EEXClass;

EEXClass.spClassName = CTRL_SLEDIT;
EEXClass.opMask = 0x00FF;
if (GetWindowClassInfo (&EEXClass) == FALSE)
 	return FALSE;

old_ctrl_proc = EEXClass.WinProc;

... ...

return (*old_ctrl_proc) (hCtrl, message, wParam, lParam);
```

至于获取时候的注意事项，忘记了的要回头去看教程2了哦。然后继承的方式也需要在使用前注册，使用完成后卸载，也是只能使用CreateWindow()来创建，方法和教材4中的一样。

* 2：为每份控件实例保存数据

这个上面也提到了的，方法和教程2的一样，要用附加数据1保存，然后自己多加一个DWORD的变量出来提供给外部应用程序使用即可。

* 3：其它

1. 判断输入消息选用 `MSG_CHAR`。为什么选用这个，而不用 `MSG_KEYDOWN` 呢。因为 `MSG_CHAR` 是经过字符翻译了的。我们的控件判断只关心有意义的字符，像delete、insert、home、pageup、pagedown、left、right、up、down之类的我们根本不用管。正好 `MSG_CHAR` 是不会翻译这些按键消息的 ^_^。

2. 有关MSG_GETDLGCODE。还记上次教程我说过控件在dialog中要想正确获取按键所有就要处理这个消息的吗。没错像编辑框这种控件确实是要正确处理这个消息才能正确的获取所有的键盘输入。但是这里我建议大家不要画蛇添足，因为父类已经帮我们处理，直接把这条消息扔给父类的处理函数就行了。

3. 编辑框控件有个能用鼠标选中一串字符然后用输入的字符代替掉选中字符的功能。这个功能在数字模式下要进行特别的处理，这个、这个大家还是看我代码吧，这里不好说呢 ^_^ 。

## 六、总结

好了，我觉得需要注意的就是这么多了，其它的大家看代码就OK了。其实扩展原有控件类（继承）的方法和自己重新开始写，大体上来说基本类似，就是注意下以下的问题就行了：

1. 最后调用的父类处理函数。继承的当然是调用父类的啦（注意获取父类处理函数的注意事项）；自己重新开始写是调用 DefaultControlProc()。
2. 注意一些已经被父类用掉了的资源。像保存实例数据变量的附加数据，风格变量、通知码等。扩展的需要注意，重新写的就不用在意啦，放心用 ^_^。所以大家在写自己的自定义控件的时，一开始就确定好，到底是继承还是重新开始写。

好，到这里MiniGUI2.0版本以前本人会的自定义控件的方法就全部介绍完了。据说3.0的自定义控件的方法比2.0的简单了很多，更接近于OOP。但是本人最近一直没什么时间用3.0，所以只好“落伍“一下了 -_-||。 我写的ControlEx中还有2个扩展控件：进度条和滚动显示。后面的教程都只是说我写的这些控件的具体实现的了，就不说MiniGUI自定义控件的方法了。

哎，2月3号的起的头，又拖到8号才写完。发现我越来越懒了，杯具 -_-|| 。

参考资料：飞漫MiniGUI编程指南2.0.4
 
## 代码下载
[下载地址]("http://download.csdn.net/detail/mingming_killer/4045894")

