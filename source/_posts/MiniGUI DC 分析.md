title: MiniGUI DC 分析
date: 2015-01-21 21:15:16
categories: [MiniGUI]
tags: [minigui]
---

## 相关术语
这里先解释下相关术语吧（呃～～是按我的理解）：

* **GDI**
GDI（Graphics Device Interface）: 图形设备接口。这应该是一个抽象层，在这一些为上层应用程序提供了一系列图形绘制相关的接口：例如画点、画线、矩形填充、块传送、加载图片文件等等。它衔接底层硬件层与上层应用层。

* **DC**
DC（Device Context）：设备上下文。这个是 gdi 层的基本单位。包含了像素颜色信息、填充函数、图像缩放函数等等，上层应用程序通过 gdi 接口操作封装在里面的底层图像数据。

* **HDC** 
HDC（Handle Device Context）： dc 的句柄。其实可以理解为 dc 的指针。MiniGUI 里本来也就是这么处理的。

* **GAL** 
GAL（Graphics Abstract Layer）：图形抽象层。这个层直接和硬件打交到，是 MiniGUI 的图形驱动。这个抽象了一些列的接口规范，供上层（GDI层）使用，不同的硬件平台使用自己硬件平台提供的接口（直接的 IO 操作或是封装过的库）完成这些接口就完成了 MiniGUI 的图形驱动。

* **Surface** 
Surface：哎～～这个我还真不知道要怎么翻译好，干脆就直接用英文吧。surface 是 GAL 层的基本单位，包括最基本的像素数据（图像数据）、颜色格式、图形驱动回调函数等等。dc 中包含了这个基本单位。

* **层次图如下：** 
![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/minigui-dc/1.png "层次图")

## DC 的创建与销毁
从应用层来看，要绘制图形，首先要获取 dc。来看看总体情况吧，MiniGUI 3.0.x 现有的 dc 创建相关的 API 大致有以下几个：

```cpp
HDC GetDC (HWND hwnd);
HDC GetClientDC (HWND hwnd);
HDC GetSubDC (HDC hdc, int off_x, int off_y, int width, int height);

HDC CreateCompatibleDCEx (HDC hdc, int width, int height);
HDC CreateMemDCEx (int width, int height, int depth, DWORD flags, Uint32 Rmask, Uint32 Gmask, Uint32 Bmask, Uint32 Amask, void* bits, int pitch);
HDC CreateSubMemDC (HDC parent, int off_x, int off_y, int width, int height, BOOL comp_to_parent);

HDC CreatePrivateDC (HWND hwnd);
HDC CreatePrivateClientDC (HWND hwnd);
HDC CreatePrivateSubDC(HDC hdc, int off_x, int off_y, int width, int height);
HDC GetPrivateClientDC (HWND hwnd);

HDC CreateSecondaryDC (HWND hwnd);
HDC GetSecondaryDC (HWND hwnd);
HDC GetSecondaryClientDC (HWND hwnd);
HDC GetSecondarySubDC (HDC secondary_dc, HWND hwnd_child, BOOL client);

void ReleaseDC (HDC hdc);
void ReleaseSecondaryDC (HWND hwnd, HDC hdc);
void ReleaseSecondarySubDC (HWND hwnd, HDC hdc);
void DeleteMemDC (HDC mem_dc);
void DeletePrivateDC (HDC hdc);
void DeleteSecondaryDC (HWND hwnd);
```
以上一系列 API ，可以从几个方面分类：

* **存储位置：**
MiniGUI 内部有一个叫 dc 池的缓冲区，启动 MiniGUI 的时候就分配好了内存空间，是一系列静态变量，在 MiniGUI 周期中是常驻内存的。无需动态申请和释放。有部分 API 是从这个里面创建的，这类 API 通常使用频率较高，创建速度快（仅仅需要简单把 dc 池中的一些标志位设置一些即可，无需内存申请操作）。有部分则是自己申请内存创建的。

* **像素数据类型：** 
有些 API 是以屏幕的像素数据为内容来创建 dc 的。操作这类 API 创建的 dc ，立刻就能在屏幕上体现。有些则不是，需要手动调用位块传送 API 输出到屏幕才能体现来（BitBlt, StrechBlt 等）。

* **API之间的对比：** 

    * **GetDC**
       * 存储位置： dc 池
       * 像素数据类型： on-screen
       * 包括区域： 整个窗体
       * 特性： 

    * **GetClientDC**
       * 存储位置： dc 池
       * 像素数据类型： on-screen
       * 包括区域： 窗体的非客户区
       * 特性： 

    * **GetSubDC**
       * 存储位置： dc 池
       * 像素数据类型： 使用父 dc 的像素数据，但必须是 off-screen
       * 包括区域： 参数指定大小，但无法超过父 dc 的大小
       * 特性： 与父 dc 共用用一块像素地址，子 dc 的操作将会影响到父

    * **CreateCompatibleDCEx**
       * 存储位置： 动态内存
       * 像素数据类型： off-screen
       * 包括区域： 参数指定大小
       * 特性： 创建的 dc，将会与传入的参考 dc 有相同的颜色格式。CreateCompatibleDC 是创建与参考 dc 同样大小的 dc，是对该 API 的简单封装

    * **CreateMemDCEx**
       * 存储位置： 动态内存
       * 像素数据类型： off-screen
       * 包括区域： 参数指定大小
       * 特性： 能自己指定色深、颜色格式，以及初始的像素数据。CreateMemDC 创建的 dc 初始像素数据为0，是该 API 的简单封装

    * **CreateSubMemDC**
       * 存储位置： 动态内存
       * 像素数据类型： 使用父dc的像素数据，应该是 off-screen 的吧
       * 包括区域： 参数指定大小，但无法超过父 dc 的大小
       * 特性： 参数指定是否与父 dc 具有相同的剪切域

    * **CreatePrivateDC**
       * 存储位置： 动态内存
       * 像素数据类型： on-screen
       * 包括区域： 整个窗体
       * 特性： 

    * **CreatePrivateClientDC**
       * 存储位置： 动态内存
       * 像素数据类型： on-screen
       * 包括区域： 窗体的非客户区
       * 特性： 好像是用于窗口或是控件的一个风格（WS_EX_USEPRIVATECDC、CS_OWNDC）

    * **CreatePrivateSubDC**
       * 存储位置： 动态内存
       * 像素数据类型： 使用父 dc 的像素数据
       * 包括区域： 参数指定大小，但无法超过父 dc 的大小
       * 特性： 

    * **GetPrivateClientDC**
       * 存储位置： 已经存在的变量
       * 像素数据类型： 应该是 on-screen
       * 包括区域： 未知
       * 特性： 简单的返回 PMAINWIN 的 privCDC 变量

    * **CreateSecondaryDC**
       * 存储位置： 动态内存
       * 像素数据类型： off-screen
       * 包括区域： 整个窗体
       * 特性： 最后调用 CreateCompatibleDCEx 创建通过 GetDC 获取兼容的 dc

    * **GetSecondaryDC**
       * 存储位置： 不确定
       * 像素数据类型： 应该是 on-screen
       * 包括区域： 不确定
       * 特性： 这个函数比较复杂，后面再分析

    * **GetSecondaryClientDC**
       * 存储位置： 
       * 像素数据类型： 
       * 包括区域： 
       * 特性： 和 GetSecondaryDC 类似，只不过区域是非客户区而已

    * **GetSecondarySubDC**
       * 存储位置： dc 池 
       * 像素数据类型： 应该是 off-screen 
       * 包括区域： 参数指定大小，但无法超过父 dc 大小 
       * 特性： 这个 API 主要是 secondary dc 的获取 sub dc 版

    * **ReleaseDC**
       * 存储位置： 释放 dc 池（只是简单设置标志位而已）
       * 像素数据类型： on-screen 数据不能销毁
       * 包括区域： 
       * 特性： 从 dc 池创建的 dc，都应该由这个 API 释放

    * **ReleaseSecondaryDC**
       * 存储位置： 释放 dc 池
       * 像素数据类型： 
       * 包括区域： 
       * 特性： 根据不同的情况调用 ReleaseDC 或是 ReleaseSecondarySubDC

    * **ReleaseSecondarySubDC**
       * 存储位置： 释放 dc 池
       * 像素数据类型： 
       * 包括区域： 
       * 特性： 对 ReleaseDC 的简单封装

    * **DeleteMemDC**
       * 存储位置： 释放内存
       * 像素数据类型： 销毁 off-screen 数据
       * 包括区域： 
       * 特性： 申请动态内存，使用 off-screen 像素数据的 dc 都应该由这个 API 销毁

    * **DeletePrivateDC**
       * 存储位置： 释放内存
       * 像素数据类型： on-screen 数据不能销毁
       * 包括区域： 
       * 特性： 由 CreatePrivateDC 创建的 dc 要使用该 API 销毁

    * **DeleteSecondaryDC**
       * 存储位置： 
       * 像素数据类型： 
       * 包括区域： 
       * 特性： 对 DeleteMemDC 的简单封装

* **情况复杂的API：** 
GetSecondaryDC：这个函数首先分为是主窗口还是控件，其实再看主窗口或是控件的风格：
    * 主窗口：如果有 `WS_EX_AUTOSECONARYDC` 风格，则返回 PMAINWIN 的 secondaryDC 变量（这个变量通过 SetSeconaryDC 设置，一般是 off-screen dc 来的）。否则则返回 `HDC_SCREEN` （屏幕dc）。

    * 控件：如果父窗体有 `WS_EX_AUTOSECONARYDC` 风格，控件如果没有 `WS_EX_CTRLASMAINWIN` 风格，则返回父窗体 secondary dc 的子 dc （通过 GetSeconarySubDC 获取）；控件如果有 `WS_EX_CTRLASMAINWIN` 风格的话，则会通过 GetClientDC 获取；控件如果有 `WS_EX_USEPRIVATECDC` 则会返回控件的 privCDC 变量。如果父窗体没有 `WS_EX_AUTOSECONARYDC` 风格的话，则返回 `HDC_SCREEN`。

哎，说得头晕，还不如直接上代码：

```cpp
HDC GUIAPI GetSecondaryDC (HWND hwnd)
{
    PMAINWIN pWin;
    pWin = MG_GET_WINDOW_PTR (hwnd);

    if (MG_IS_MAIN_WINDOW(hwnd) && pWin->secondaryDC) {
        return pWin->secondaryDC;
    } 
    else if (pWin->pMainWin->secondaryDC){
        return get_valid_dc (pWin, FALSE);
    }
    return HDC_SCREEN;
} 

static inline HDC get_valid_dc (PMAINWIN pWin, BOOL client)
{
    if (!(pWin->dwExStyle & WS_EX_CTRLASMAINWIN) 
            && (pWin->pMainWin->secondaryDC)) {
        if (client && (pWin->dwExStyle & WS_EX_USEPRIVATECDC)) {
            return pWin->privCDC;
        }
        else
            return GetSecondarySubDC (pWin->pMainWin->secondaryDC, 
                    (HWND)pWin, client);
    }
    else {
        if (client && (pWin->dwExStyle & WS_EX_USEPRIVATECDC)) {
            return pWin->privCDC;
        }
        if (client) {
            return GetClientDC ((HWND)pWin);
        }
        else {
            return GetDC ((HWND)pWin);
        }
    }
}
```

还有一对 API 用来获取和释放 dc 的：BeginPaint 和 EndPaint 。首先这一对 API 正常情况应该只用于 `MSG_PAINT` 消息处理里。BeginPaint 是通过内联函数 `get_valid_dc` 来获取。所以主窗体（或是控件）不同的风格获取到的 dc 是不同的。一般来说有双缓冲风格的获取到的一般是 off-screen 的 dc （之前设置的双缓冲 dc，控件得到的是父窗体缓冲的子 dc）；没有双缓冲的就是 on-screen dc 。并且这个 API 获取的 dc 是非客户区的区域的（这里是最大矩形区域），然后是经过剪切的（就是 MiniGUI 窗口系统通过窗口重叠、遮挡计算后，设置剪切好不需要绘制的区域了的）。然后由这个 API 获取到的 dc 一般要由  EndPaint 来释放，因此 EndPaint 不光光会释放 dc，如果有双缓冲风格的话，还会在这里调用双缓冲的更新函数（默认是将缓冲 dc 更新到屏幕）。

## DC 工作原理
通过上面的 API 获取（或是创建）了 dc 后，再调用 GDI 层提供的一系列图形绘制函数就可以画出图像了。从上层应用程序调用一个图形绘制 API 到真正在屏幕上显示，这一个过程是怎么样的？我以前用 MiniGUI 的时候一直有这么一个疑问。现在把我自己的一些理解总结下：

### 基本流程
现在以 FillBox (HDC hdc, int x, int y, int w, int h) 这样一个GDI 层的简单的 API 为例，走下 MiniGUI 的流程。首先这个 API 的作用是使用 dc 的画刷（可以使用 SetBrushColor 设置，默认为白色）颜色来填充 dc 中的指定矩形区域。它的基本流程是：

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/minigui-dc/2.png "dc流程")

* 通过上面的一些 API 中的一个取得 dc（这里以 on-screen 类的 dc 为例子）。
* 设置你获取的 dc 的画刷颜色（不设置就是之前的颜色，默认为白色）。
* 根据 dc 的剪切域来填充矩形。
* 根据 dc 中 surface 的类型选用填充函数。
* 填充函数完成图像数据填充。
* on-screen 数据被修改（填充）直接表现在屏幕上。

### 要点分析
上面的步骤看似简单，但是其中有些步骤是包括很多事情的。下面来分析下上面步骤中一些需要注意的地方：

* **dc 剪切域：** 
上面提到的 MiniGUI 会根据 dc 中的剪切域来填充指定的矩形。剪切域上面也有提到过的，是 MiniGUI 窗口系统通过一系列窗口遮挡关系计算出来的，保证看不到的地方不绘制，用以提高绘制效率的机制。MiniGUI 会调用内部的填充函数分别填充剪切域。如果被遮挡过的（或是调用专门的接口设置过），那一般矩形会被分割成几个部分（需要绘制的部分，就是把不需要绘制的部分从原来矩形里剔除了）。所以你在上层传入的一个块区域信息，但是到了 MiniGUI 内部不一定一次就填充完成。很有可能这块区域被分成了几块小的区域，填充多次。MiniGUI 里绘制一些不规则的图形就是通过剪切域来实现的。

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/minigui-dc/3.png "剪切域示例")

* **surface 的类型：** 
上面说的根据 surface 的类型选用填充函数。surface 分为软件的还硬件的。软件的意思是说这个 surface 的像素数据是使用系统内存来存储的，各项图形（像素）操作是通过 CPU 来实现的。而硬件 surface 则是像素数据使用显存存储，各项图形（像素）操作是通过 GPU 来实现。这里说 GPU 有点夸张，很多芯片或是平台上，都是只有一块简单的图形处理芯片而已，有些则是集成在了 CPU 里。这里很明显可以看得出硬件 surface 的好处。首先它的像素数据不占用系统内存，其次它的各种图形操作可以通过专用的图形芯片进行加速处理。通常图形芯片处理图像数据比处理通用数据的 CPU 要快得很多，特别是图像数据在显存中的时候。

那什么时候时候 dc 用的 surface 是硬件的，什么时候用的是软件的呢？要想使用硬件的 surface ，首先你运行 MiniGUI 的平台要有图形处理芯片（硬件支持）。其次 MiniGUI 有该图形芯片对应的 GAL （硬件驱动）。还要你 MiniGUI 的运行配置要使用该 GAL （MiniGUI.cfg 中的 gal_engine 设置）。最后你创建 dc 时，要告诉 MiniGUI 你要创建硬件 surface 的 dc （通过指定 `MEMDC_FLAG_HWSURFACE` 或是兼容硬件 dc 来创建）。还有一个关键因素，创建硬件 suraface 要满足底层硬件的条件，例如创建的时候显存要足够、必须是在硬件支持的颜色格式中等等。当你的 MiniGUI 不满足上面任意一个条件时，就会创建软件 surface 的 dc。

* **图像数据操作：** 
这里要分开2种情况来讨论： 

    * **软件的 surface：** 
软件的 surface 的图像数据处理，其实就是一些列的写点操作。MiniGUI 的软件 surface 图形操作函数，就是将 surface 里的图像数据写成指定的颜色（简单的赋值操作），只不过某些颜色格式（例如4字节对齐的32位）会用到一些优化的赋值算法。这里再说说图像数据吧。一般计算机用的是 RGB 颜色，这里以32位为例，RGB 每个分量从 0 ～ 255 （占用8位，1个字节），然后组合成各种不同的颜色。在 RGB 颜色方式中，一个像素点的数据就是一个 RGB 数值，所以一张图像就是一系列连续的这些像素点的集合，也就是一系列连续的 RGB 数值。这个就是图像数据，占用一片连续的存储空间（软件的就占用系统内存）。所以说软件的图像操作，都是操作这些 RGB 数值，可以简单的理解为 C 语言里的数组操作。

    * **硬件的 surface：** 
硬件的 surface 图像处理，就调用不同的图形处理芯片的接口来进行的。一般的 2D 图形处理芯片对于色块的传送有比较明显的加速。一般表现为一次一片、一片的处理，比软件的一次一个点要快得多。同时有些硬件还支持一些像素混合的加速功能（alpha、colorkey 等）。

    * **屏幕显示：** 
屏幕显示。不管是软件还是硬件的 surface ，操作它们的数据，是怎么影响到屏幕的呢？我们这个例子的 surface 数据是 on-screen 类型的。这个就比较直接了。软件的话，屏幕的图像数据就是内存中的那些 RGB数值（上面分析了的），所以你直接修改它们，就能马上显示到屏幕上。硬件的，虽然它们需要经过硬件图形芯片的处理，但是显存里面的数据还是屏幕的。所以图形芯片处理过后，屏幕上的图像也马上就更新了。但是 MiniGUI 硬件 surface 也是支持 off-screen 的，这又是怎么回事呢。没错，显存确实是屏幕的，但是一般来说显存都比屏幕大。这句话怎么理解呢。举个例子：假设屏幕是 640x480-16bpp 的，那么以 RGB 颜色格式来说一个屏幕说占用的显存大小就是 600kb，假设显存有10Mb，那么 off-screen 显存的大小就是 10Mb - 600kb = 9.4Mb 。这就说虽然这 9.4Mb 是屏幕的数据，但是由于屏幕只有 640x480 这么大，你操作了这些数据，理论上来说是应该马上在屏幕上显示的，但是由于用户其实是看不到这些内容的（前面说了，屏幕没这么大）。如果要想在屏幕上显示这部分显存的内容，需要色块传 API （也就是 MiniGUI 里的 BitBlt 接口）把这些显存的内容复制到能在屏幕上显示的显存的地址才行。所以硬件的 surface 也是能够有 off-screen dc 这种用法的。

## 其它话题

### 颜色运算
不管是颜色填充（FillBox）还是色块传送（BitBlt），最终都是要做颜色运算。现在填充 MiniGUI 支持的颜色运算就是颜色覆盖和 alpha 混合。

* 颜色覆盖就是直接拿源颜色的数值赋值给目标颜色。上面讲解中例子的用法的话，就是把屏幕图像数据中的某个一片数值改成了 dc 画刷的颜色（RGB值）。

* alpha 混合就是通过一个指定颜色公式，使用颜色值中的 alpha 信息将源颜色和目的颜色进行混合，从而达到混合后的颜色信息带有目的颜色的目的。这个就是通常说的透明效果。一般来说 alpha 范围为 0 ～ 255，一般的混合公式为下面代码。从上面的公式可以看得出如果 alpha 值越小，那么目标颜色残留的就越多，源颜色的透明度也就越高（通俗的说就是源颜色很淡，而目标颜色很浓，透过源颜色能看到能多的目标颜色）。另外 alpha 混合的方式有很多种，例如是使用源颜色的 alpha 值做为混合标准（`MEMDC_FLAG_SRCALPHA`），还是使用目标颜色的 alpha 值做为混合标准；将不将源颜色的 alpha 值写入到目标颜色的 alpha 中去。目前 MiniGUI 的 alpha 混合只支持以源颜色的 alpha 为标准进行运算，并且不将源颜色的 alpha 值写入目标颜色的 alpha 中。经过上面的分析就能解释一个比较常见的问题：画透明图像的时候，在更新时透明图像会越叠越深，数次更新后，就不透明了。引起这个问题的主要原因就是，透明图像的背景没有被更新。这样的话透明图像（源颜色）第一次和透过的背景图像（目标颜色）进行 alpha 混合时，是正常的。随后，如果够过的背景图像没有被更新的话，那么透明图像就会和第一次混合后的图像进行混合。从上面的分析就可以看出，混合后的图像就像是把透明图像再次叠加到背景上一样（事实本来也是这样）。要想得到正确的效果，每次都必须更新背景图像，也就是说每次都要让透明图像和没有被混合后的图像进行混合。

代码：

```cpp
// MiniGUI 32 位 alpha 混合代码
DUFFS_LOOP4(
{
    Uint32 s;
    Uint32 d;
    Uint32 s1;
    Uint32 d1;
    Uint32 sA;
    Uint32 dA;
    s = *srcp;
    d = *dstp;
    sA = s >> 24;
    dA = d >> 24;
    dA = sA + dA - ((sA * dA) >> 8);
    if(dA > 255) dA = 255; //alpha may be greater than 255
    alpha = s >> 24;
    s1 = s & 0xff00ff;
    d1 = d & 0xff00ff;
    d1 = (d1 + (((s1 - d1) * alpha) >> 8)) & 0xff00ff;
    s &= 0xff00;
    d &= 0xff00;
    d = (d + (((s - d) * alpha) >> 8)) & 0xff00;
    *dstp = d1 | d | (dA << 24);
    ++srcp;
    ++dstp;
}, w);
```

* colorkey：这个也有2种模式的，一种是以源颜色的 colorkey 颜色值做为标准（`MEMDC_FLAG_SRCCOLORKEY`），一种是以目标颜色的 colorkey 颜色值做为标准。这个在进行颜色填充或是色块传送的时候，如果遇到和 colorkey 颜色值一样的颜色，就会跳过这些颜色，不进行像素操作。这样就能形成一些不规则图形的填充（把背景颜色做为 colorkey 过滤掉）。目前 MiniGUI 只支持以源颜色 colorkey 颜色值做为标准。

### 绘制图像导致屏幕闪烁
导致这个问题的一般原因是，直接在屏幕上进行大规模的图像更新操作。这样绘制过程就会在屏幕上展现出来，这样就会感觉屏幕在闪烁（屏幕更新了多次）。在应用程序上导致这个问题的最常见的情况就是获取或是创建了 on-screen dc，然后绘制操作都在这个 dc 上进行。这样 dc 中的每个图像数据点的操作都会反映到屏幕上。短时间内颜色的频繁变化，在人的视觉上就会造成一种突变的感觉，就是通俗上说的闪烁。从上面分析可以看得出要避免这个问题，就是要避免在绘制过程中屏幕多次的改变。这里注意，即便是在应用程序中仅仅调用一次 FillBox 填充一块区域，如果是直接在 on-screen dc 上操作的话，屏幕的变化不仅仅是一次。如果是软件的 surface 应该说是这块区域有多少个像素点就有多少次，当然这个还和屏幕的刷新率有关。所以在应用程序中避免这样的问题，就需要在 off-screen dc 中把需要更新到屏幕的图像完全绘制好，然后使用色块传送 API 把这些绘制好的图像复制到屏幕上。在某些硬件上这个操作几乎可以说就是一次性完成的，计算是 MiniGUI 的软件 surface 操作的话，也在色块传送上也有专门的优化。这样能极大的减少短时间内屏幕的变化次数，从而缓解因为图像更新导致的屏幕闪烁问题。MiniGUI 3.0 的新特性双缓冲机制就是针对解决这个问题而提出的。


