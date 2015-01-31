title: STi7167 GAL 开发笔记
date: 2015-01-21 21:25:16
categories: [MiniGUI]
tags: [minigui]
---

以前没弄过和板子相关，包里的howto文档不太具体，所以这次把一些基本的步骤也记下来。 

## 硬件环境搭建
STi7167开发板、HDMI接口线、usb转com口线、网线、1080p显示器、开发宿主机。用接口线这这些设备连接起来，注意宿主机和开发版都要接入同一个局域网。

## 软件环境搭建
开发环境使用 minicom 连接开发板，通过uboot使用nfs挂载宿主机的根文件系统启动。板子上跑的是 STLinux，图像驱动库是 DirectFB 或者是 ST 自己的 ioctl。DirectFB 是经过封装的，稍微上层点的。其实 DirectFB 的加速插件也就是使用 ST 的 ioctl 的。

### 安装STLinux
* 先在宿主机上搭建nfs服务。nfs-client, nfs-common, nfs-kernel-server, nfs-server（用 sudo apt-get install 安装）。然后修改 /etc/exports 设置nfs目录(这里设置的是 /home 为nfs目录。然后 /home/nfs 目录增加响应权限。)：
<pre config="brush:bash;toolbar:false;">
/home/  *(rw,sync,no_subtree_check,anonuid=1007,anongid=125)
/home/nfs *(rw,sync,no_subtree_check,no_root_squash,anonuid=1007,anongid=125)
</pre>

* 解压 STM.tar.gz（要sudo解压，其中要创建一些链接节点）。将 /STLinux-2.3/devkit/sh4/target 文件夹放到 /home/mingming/nfs 下。要把target文件夹的权限改一下。

* 在宿主机启动 minicom（配置文件在用户目录/.minirc.dlf）。ctrl+A+Z改下配置。查看下/dev/下的usb节点是哪个（一般第一是ttyUSB0），修改串口端口为相应的usb端口（例如ttyUSB0）。关掉硬件、软件流控制。打开 linewrap 功能（因为后面的配置命令很长）。

* 上电启动板子，在 minicom 的终端按任意键进入配置命令行。根据实际情况修改howto文档里给出的配置命令：
<pre config="brush:bash;toolbar:false;">
setenv bootargs console=ttyAS0,115200 ip=192.168.1.233:192.168.1.101:192.168.1.1:255.255.0.0::eth0:off nwhwconf=device:eth0,hwaddr:00:FA:E0:FA:E0:00 rw root=/dev/nfs nfsroot=/home/mingming/nfs/target,nfsvers=2,tcp bigphysarea=2000 
</pre>

这里宿主机的ip是 192.168.1.101，板子的ip是 192.168.1.233，网关为 192.168.1.1，子网掩码为 255.255.0.0。然后saveenv，reset重启，之后就可以正常启动了。在根文件系统中的 /root/ 下有个 load_fb.sh 的脚本，可以通过修改里面的参数设置 framebuffer 的颜色格式（FULLHD 那里，format0 可以改成 RGB565，就可以改掉逐点 alpha 的颜色格式，一般就用这2种颜色格式）：
<pre config="brush:bash;toolbar:false;">
# Resolution for main output 
# --------------------------
if [ $# -eq 0 ] || [ $1 = "PAL" ]; then
    mode0=720x576-50i
    mem0=4
    format0=ARGB1555
    ratio0="4TO3"
else if [ $1 = "HD" ]; then
    mode0=1280x720-50
    mem0=7
    format0=ARGB1555
    ratio0="4TO3"
else if [ $1 = "FULLHD" ]; then
    mode0=1920x1080-50i
    mem0=10
    format0=ARGB1555
    ratio0="4TO3"
fi
fi
fi
</pre>

### DirectFB 测试例子
启动STLinux后，用root登录。在/bin目录下，ln -s telnetd busybox，用busybox链接出telneted。然后用telneted命令启动telnetd服务。之后就可以按照howto文档里说的加载模块，启动DirectFB，在宿主机的另外一个终端用telnet登录到板子上，然后运行 df_dok 的测试例子。

### 验证交叉工具链
宿主机终端命令： export PATH=/opt/STM/STLinux-2.3/devkit/sh4/bin/:$PATH （这里的PATH是解压STLinux平台交叉编译工具的路径）。然后在终端试试能运行 sh4-linux-gcc --version 查看交叉工具链的版本号。从hwoto文档里给出的 DirectFB 的网站里下载到填充背景和画线的simple，用 sh4-linux-gcc simple.c -I/opt/STM/STLinux-2.3/devkit/sh4/target/usr/include/directfb/ -ldirectfb -o dfbsimple 交叉编译（注意这里的路径要是你自己解压的路径）。然后把编译出来的程序复制到之前的 nfs 根文件系统的一个位置（例如/roo/test/）。然后telnet登录的终端就可以运行编译出来的程序。能看到HDMI输出填充指定背景颜色和画线就说明交叉工具链正常。

## 基于 DirectFB 的 GAL
... ...

## 基于 stgfb_ioctl 的 GAL
这个是使用更加底层的方式，直接调用ST提供的ioctl操作，不使用 DirectFB 封装。优点是更加灵活，直接的操作ST的底层芯片，能够使用ST芯片的所有功能，不受封装库的限制。缺点就是比较麻烦，ST的底层ioctl命令参数很多。7167的2D图像加速硬件叫 STBlit ，有2种方式可以访问：

* 通过调用 stblit 的封装函数。这些函数是对 stblit_ioctl 的封装。见图1。

* 直接使用 stgfb_ioctl，这个是 STBlit 扩展的 FrameBuffer 的 ioctl 操作（也就是7167 2D图像硬件特有的ioctl操作，这里就叫做 stgfb_ioctl）。见图2。
ST 是推荐使用第2种方式，因为第二种方式不需要经过 stblit 函数的封装，速度更快。我是选择使用第2种方式的，倒不是因为 ST 的推荐，而是因为我手头上只有第2种方式的例子。ST 的那个 stapi 的源代码，看得我头都晕了。这里要特别感谢佐朝同志提供的示例。

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/sti7167-gal/1.png)

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/sti7167-gal/2.png)

### 基本流程
使用 stgfb 的基本流程如下：

* 首先要加载 7167 上 ST 和图像相关的驱动模块：stblit_ioctl.ko, stlayer_ioctl.ko 等。这些在前面说的按照 howto 文档里加载内核模块驱动的时候就已经加载好了。

* 打开 FrameBuffer 设备。按前面的加载好内核模块，并加载 fb 设备的话。fb 的设备节点应该是 /dev/fb0, /dev/fb1，fb0是主屏，fb1是副屏。这里只使用到了主屏。这里的打开代码可以参照 MiniGUI fb GAL 部分的代码。

* 获取视图（ViewPort）处理句柄（Handler）。

* 初始化 7167 主显示层（layer）。首先要打开层管理设备（/dev/stapi/stlayer_ioctl），然后通过 ioctl 发送使能命令激活该层，最后通过 ioctl 获取该层的一些信息（也就是屏幕的信息）。

* 到这里初始化工作就完成了。可以通过 fb 设备文件描述符号（这个就是之前打开的 FrameBuffer 设备），使用 stgfb_ioctl 发送 STGFB_IO_BLIT_COMMAND 命令（要填写响应的命令参数），让 STBlit 进行图像绘制操作。

* 退出的时候，关闭之前打开的设备即可。

### stgfb io 接口命令

```cpp
#define STGFB_IO_SET_OVERLAY_COLORKEY       _IOW('B', 0x1, int)
#define STGFB_IO_ENABLE_FLICKER_FILTER      _IOW('B', 0x2, int)
#define STGFB_IO_DISABLE_FLICKER_FILTER     _IOW('B', 0x3, int)
#define STGFB_IO_BLIT_COMMAND               _IOW('B', 0x4, STGFB_BLIT_Command_t)
#define STGFB_IO_SET_OVERLAY_ALPHA          _IOW('B', 0x5, int)
#define STGFB_IO_GET_LAYER_HANDLE           _IOW('B', 0x6, int)
#define STGFB_IO_GET_VIEWPORT_HANDLE        _IOW('B', 0x7, int)
#define STGFB_IO_SET_FB_POSITION            _IOW('B', 0x8, int)
#define STGFB_IO_SYNC_BLITTER               _IOW('B', 0x9, int)
#define STGFB_IO_SET_WINDOW                 _IOW('B', 0x10, int)
#define STGFB_IO_ENABLE_LAYER               _IOW('B', 0x11, int)
#define STGFB_IO_SET_PREMULTIPLIED          _IOW('B', 0x12, int)
```
### ViewPort Handler
通过 STGFB_IO_GET_VIEWPORT_HANDLE 命令发送，设备是 fb 设备。参数为 STLAYER_ViewPortHandle_t ，这应该是一个地址。该句柄在是后面获取 layer 信息的参数，一定要取得。

### Layer
* 首先是要激活一个 layer。通过 STGFB_IO_ENABLE_LAYER 命令发送，设备是 fb 设备。参数为 int，1为激活，0为禁用，注意传的时候，要传入该变量的地址，而不是把该变量的值传过去。

* 激活后，就可以通过 STLAYER_IOC_GETVIEWPORTPARAMS 命令发送获取层信息，注意这时候的设备是 layer 的设备（就是 /dev/stapi/stlayer_ioctl）。参数是 STLAYER_Ioctl_GetViewPortParams_t 。其中的 VPHandle 是传入参数，就是之前获取的那个。STLAYER_Ioctl_FlatViewPortParams_t 是输出参数，具体的参数就在里面。在 ST 内部，图像是以 STGXOBJ_Bitmap_t 为单位的，一个图像就是这么一个位图。从这个结构体定义就可以知道屏幕的一些重要信息了（以下的注释是我自己写的，非官方的）：

代码：

```cpp
typedef struct STGXOBJ_Bitmap_s
{
  STGXOBJ_ColorType_t                   ColorType;          // 颜色格式，RGB565, ARGB1555 等
  STGXOBJ_BitmapType_t                  BitmapType;         // 图像类型，作用未知
  BOOL                                  PreMultipliedColor;       // 这个也暂时不知道是干啥的，好像是某种逐点特效（应该不是逐点alpha）
  STGXOBJ_ColorSpaceConversionMode_t    ColorSpaceConversion;     // 颜色转化空间，作用未知
  STGXOBJ_AspectRatio_t                 AspectRatio;         // 长宽比
  U32                                   Width;               // 宽
  U32                                   Height;              // 高
  U32                                   Pitch;               // Pitch值
  U32                                   Offset;              // 地址偏移1，这个应该是显存偏移
  void*                                 Data1_p;             // 物理地址1，这是物理显存地址，这个和用fb获取到的地址是一样的
  void*                                 Data1_Cp;            // 物理地址1，这个是cache地址
  void*                                 Data1_NCp;           // 物理地址2，这个是uncache地址
  U32                                   Size1;               // 地址1的长度
  void*                                 Data2_p;
  void*                                 Data2_Cp;
  void*                                 Data2_NCp;
  U32                                   Size2;
  STGXOBJ_SubByteFormat_t               SubByteFormat;
  BOOL                                  BigNotLittle;        // TRUE：大端存储；FALSE：小端存储
  U32                                   Pitch2;
  U32                                   Offset2;
  YUV_ScalingFactor_t                   YUVScaling;
} STGXOBJ_Bitmap_t;

typedef struct STLAYER_Ioctl_FlatViewPortSource_s
{    
    STLAYER_ViewPortSourceType_t    SourceType;    // 源数据类型，就是告诉你下面的Data联合体是哪个：是位图还是视频流
    union
    {
        STGXOBJ_Bitmap_t           BitMap;
        STLAYER_StreamingVideo_t   VideoStream;
    }                              Data;
    STGXOBJ_Palette_t              Palette;
} STLAYER_Ioctl_FlatViewPortSource_t;


typedef struct STLAYER_Ioctl_FlatViewPortParams_s
{
    STLAYER_Ioctl_FlatViewPortSource_t  Source;
    STGXOBJ_Rectangle_t             InputRectangle;
    STGXOBJ_Rectangle_t             OutputRectangle;
} STLAYER_Ioctl_FlatViewPortParams_t;

typedef struct
{
    /* Error code retrieved by STAPI function */
    ST_ErrorCode_t                      ErrorCode;

    /* Parameters to the function */
    STLAYER_ViewPortHandle_t            VPHandle;
    STLAYER_Ioctl_FlatViewPortParams_t  FlatParams;

} STLAYER_Ioctl_GetViewPortParams_t;
```

### Blit
blit 使用 STGFB_IO_BLIT_COMMAND 命令来实现。stgfb 的 blit 参数主要是设置源和目标（STGXOBJ_Bitmap_t）以及一些 blit 时的一些参数：例如 colorkey、层alpha等。这里调用 ioctl 的设置是 fb 的设备。下面就这些参数分析下吧：

```cpp
/* stblit ioctl 参数 */
typedef struct
{
    STGFB_BLIT_Operation_t Operation;     // Blit 类型，见下面，Blit、FillRect、DrawRect 都是用同一个 io 命令的
	union
	{
        STGFB_BLIT_FillRectangleParams_t    FillRectangle;      // Blit 类型为 FillRect 时的参数
        STGFB_BLIT_DrawRectangleParams_t    DrawRectangle;      // Blit 类型为 DrawRect 时的参数
        STGFB_BLIT_BlitParams_t             Blit;               // Blit 类型为 Blit 时的参数
    } Params;
    ST_ErrorCode_t Result;          // 错误代码

} STGFB_BLIT_Command_t;


/* Blit 操作类型 */
typedef enum
{
    STGFB_BLIT_UNKNOWN = 0,
    STGFB_BLIT_FILLRECTANGLE,
    STGFB_BLIT_DRAWRECTANGLE,
    STGFB_BLIT_BLIT,
    STGFB_BLIT_COPYRECTANGLE
} STGFB_BLIT_Operation_t;


/* Blit 操作命令参数 */
typedef struct
{
    STBLIT_Handle_t         Handle;  // blit handle，这个没什么特殊要求的话，可以设置成0，底层有自己的handle的

    /* source 2 */
    STBLIT_SourceType_t     Src2Type;  // 第二个源，这个支持2个源混合后 blit 到目标，但是 MiniGUI 不支持，没用到
    void*                   Src2Data_p;
    STGXOBJ_Rectangle_t     Src2Rectangle;


	/* source */
    STBLIT_SourceType_t     SrcType;   // 第一个源的类型，见下面的
    void*                   SrcData_p;   // 第一个源的数据指针，如果是位图的话（blit），就是指向 STGXOBJ_Bitmap_t 的指针
    STGXOBJ_Rectangle_t     SrcRectangle;  // 第一个源的区域大小

    /* destination */
    STGXOBJ_Bitmap_t        DestBitmap;    // 目标位图
    STGXOBJ_Rectangle_t     DestRectangle;  // 目标区域

    STBLIT_BlitContext_t    Context;  // blit 的上下文参数，见下面，主要是设置 colorkey，层alpha等信息
    STGFB_BLIT_Effect_t     Effect;   // 暂时没研究用处
    void                    *Palette_p;
} STGFB_BLIT_BlitParams_t;


/* blit 源类型 */
typedef enum STBLIT_SourceType_e
{
    STBLIT_SOURCE_TYPE_COLOR,   // 单色，用于 fill 或是 draw
    STBLIT_SOURCE_TYPE_BITMAP,  // 位图，用于 blit
    STBLIT_SOURCE_TYPE_COLOR_PREMULTIPLIED    // 这个暂时还没研究是干啥的
} STBLIT_SourceType_t;


/* blit 上下文参数 */
typedef struct STBLIT_BlitContext_s
{
    STBLIT_ColorKeyCopyMode_t       ColorKeyCopyMode;   // colorkey 标志，见下面
    STGXOBJ_ColorKey_t              ColorKey;    // colorkey 颜色，一个colorkey，ST你整那么复杂干啥，见下面
    STBLIT_AluMode_t                AluMode;    // alpha 混合标志（层alpha），见下面
    BOOL                            EnableMaskWord;  // 是否允许 Mask 值，MiniGUI 不支持
    U32                             MaskWord;      // Mask 值
    BOOL                            EnableMaskBitmap;    // 是否允许 Mask 位图，MiniGUI 不支持
    STGXOBJ_Bitmap_t*               MaskBitmap_p;    // Mask 位图指针
    STGXOBJ_Rectangle_t             MaskRectangle;  // Mask 位图区域
    void*                           WorkBuffer_p;                             
    BOOL                            EnableColorCorrection;
    STGXOBJ_Palette_t*              ColorCorrectionTable_p;
    STBLIT_Trigger_t                Trigger;
    U8                              GlobalAlpha;       // 层alpha数值，注意范围是 0～128！！
    BOOL                            EnableClipRectangle;    // 是否允许 blit 剪切域，这个 MiniGUI 处理了，可以禁掉
    STGXOBJ_Rectangle_t             ClipRectangle;    // blit 剪切域
    BOOL                            WriteInsideClipRectangle;
    BOOL                            EnableFlickerFilter;
#if defined(STBLIT_OBSOLETE_USE_RESIZE_IN_BLIT_CONTEXT)
    BOOL                            EnableResizeFilter;
#endif
    STBLIT_JobHandle_t              JobHandle;    // job handle，一般设置为0
    void*                           UserTag_p;       // 通知回调用户参数，这个是用来同步用的，具体的后面解释
    BOOL                            NotifyBlitCompletion;     // 是否允许完成 blit 调用通知回调
    STEVT_SubscriberID_t            EventSubscriberID;  // 通知回调订阅ID
} STBLIT_BlitContext_t;


/* blit colorkey 标志 */
typedef enum STBLIT_ColorKeyCopyMode_e
{
    STBLIT_COLOR_KEY_MODE_NONE,  // 不使用 colorkey
    STBLIT_COLOR_KEY_MODE_SRC,  // 使用源的 colorkey，MiniGUI 仅支持这种
    STBLIT_COLOR_KEY_MODE_DST   // 使用目标的 colorkey
} STBLIT_ColorKeyCopyMode_t;


/* colorkey 参数 */
typedef struct STGXOBJ_ColorKey_s
{
  STGXOBJ_ColorKeyType_t           Type;   // colorkey 类型
  STGXOBJ_ColorKeyValue_t          Value;  // colorkey 数值
} STGXOBJ_ColorKey_t;


/* colorkey 类型，ST 你不用分这么多种吧～～不就一个像素值么 -_-|| */
typedef enum STGXOBJ_ColorKeyType_e
{ 
  STGXOBJ_COLOR_KEY_TYPE_CLUT1,   // 1位调色板
  STGXOBJ_COLOR_KEY_TYPE_CLUT8,   // 8位调色板
  STGXOBJ_COLOR_KEY_TYPE_RGB888,  // RGB888
  STGXOBJ_COLOR_KEY_TYPE_YCbCr888_SIGNED,  // 这个好像是色差分量的颜色表示方式，反正 MiniGUI 之用调色板和RGB >_<
  STGXOBJ_COLOR_KEY_TYPE_YCbCr888_UNSIGNED,
  STGXOBJ_COLOR_KEY_TYPE_RGB565   // RGB565
} STGXOBJ_ColorKeyType_t;


/* colorkey 数值参数，上面那么多类型每个对应一个成员变量～～我晕咯 -_-|| */
typedef union STGXOBJ_ColorKeyValue_u
{
  STGXOBJ_ColorKeyCLUT_t           CLUT1;
  STGXOBJ_ColorKeyCLUT_t           CLUT8;
  STGXOBJ_ColorKeyRGB_t            RGB888;   // 这里我们只用关心 RGB 类型的 colorkey
  STGXOBJ_ColorKeySignedYCbCr_t    SignedYCbCr888;
  STGXOBJ_ColorKeyUnsignedYCbCr_t  UnsignedYCbCr888;
  STGXOBJ_ColorKeyRGB_t            RGB565;
} STGXOBJ_ColorKeyValue_t;


/* RGB 类型的 colorkey 参数，看到还有这么多参数我再次晕了～～ 
   应该是为了精准，所以设置了RGB每个分量的范围。但是我感觉这个更难用。
   上层 MiniGUI 传过来的是一个像素值，怎么转化这个范围。嘿嘿，这里我把 dfb 的代码复制了过来。
    dfb 不管 16位、还是32位，统一设置成 RGB888 类型的 colorkey。让每个分量的最小值等于最大值 */
typedef struct STGXOBJ_ColorKeyRGB_s
{ 
  U8      RMin;                   // R 分量的最小值
  U8      RMax;                  // R 分量的最大值
  BOOL    ROut;                // 是否允许 R 分量超出最小值和最大值的范围
  BOOL    REnable;           // 是否启动 R 分量
  
  U8      GMin;
  U8      GMax;
  BOOL    GOut;
  BOOL    GEnable;
  
  U8      BMin;
  U8      BMax;
  BOOL    BOut;
  BOOL    BEnable;
} STGXOBJ_ColorKeyRGB_t;


/* alpha 混合标志，那么多我目前用到的就2种 */
typedef enum  STBLIT_AluMode_e
{
    STBLIT_ALU_CLEAR            = 0,
    STBLIT_ALU_AND              = 1,
    STBLIT_ALU_AND_REV          = 2,
    STBLIT_ALU_COPY             = 3,            // copy alpha ，就是不使用alpha
    STBLIT_ALU_AND_INVERT       = 4,
    STBLIT_ALU_NOOP             = 5,
    STBLIT_ALU_XOR              = 6,
    STBLIT_ALU_OR               = 7,
    STBLIT_ALU_NOR              = 8,
    STBLIT_ALU_EQUIV            = 9,
    STBLIT_ALU_INVERT           = 10,
    STBLIT_ALU_OR_REVERSE       = 11,
    STBLIT_ALU_COPY_INVERT      = 12,
    STBLIT_ALU_OR_INVERT        = 13,
    STBLIT_ALU_NAND             = 14,
    STBLIT_ALU_SET              = 15,
    STBLIT_ALU_ALPHA_BLEND      = 16    // alpha 混合
} STBLIT_AluMode_t;

```

### FillRectangle
FillRectangle 可以用2种命令来实现（都是STGFB_IO_BLIT_COMMAND，只是参数不同而已）：

* blit 时使用 STGFB_BLIT_FillRectangleParams_t 参数。这个好像不能带 colorkey 和 alpha，不推荐用这种。
* blit 时使用 STGFB_BLIT_BlitParams_t 参数。这个可以带 colorkey 和 alpha，推荐用这种。

这里就说说推荐的第二种方法吧。这种方法， ioctl 的参数类型和 blit 是一样的，但是设置值不一样，主要是在 blit 上下文参数那里的 SrcType 设置成 STBLIT_SOURCE_TYPE_COLOR （这里可以参看上面的blit），然后 SrcData_p 这个指针指向一个 STGXOBJ_Color_t 的地址。在 STGXOBJ_Color_t 里设置你要填充的颜色。colorkey 和 alpha 的设置和 blit 是一样的。下面来看看 STGXOBJ_Color_t 参数吧：

```cpp
/* 颜色结构体 */
typedef struct STGXOBJ_Color_s
{
  STGXOBJ_ColorType_t            Type;      /* 颜色类型 */
  STGXOBJ_ColorValue_t           Value;     /* 颜色值 */
} STGXOBJ_Color_t;

/* 颜色类型 */
typedef enum STGXOBJ_ColorType_e
{
  STGXOBJ_COLOR_TYPE_ARGB8888,
  STGXOBJ_COLOR_TYPE_RGB888,
  STGXOBJ_COLOR_TYPE_ARGB8565,
  STGXOBJ_COLOR_TYPE_RGB565,
  STGXOBJ_COLOR_TYPE_ARGB1555,
  STGXOBJ_COLOR_TYPE_ARGB4444,

  STGXOBJ_COLOR_TYPE_CLUT8,
  STGXOBJ_COLOR_TYPE_CLUT4,
  STGXOBJ_COLOR_TYPE_CLUT2,
  STGXOBJ_COLOR_TYPE_CLUT1,
  STGXOBJ_COLOR_TYPE_ACLUT88,
  STGXOBJ_COLOR_TYPE_ACLUT44,

  STGXOBJ_COLOR_TYPE_SIGNED_YCBCR888_444,
  STGXOBJ_COLOR_TYPE_UNSIGNED_YCBCR888_444,
  STGXOBJ_COLOR_TYPE_SIGNED_YCBCR888_422,
  STGXOBJ_COLOR_TYPE_UNSIGNED_YCBCR888_422,
  STGXOBJ_COLOR_TYPE_SIGNED_YCBCR888_420,
  STGXOBJ_COLOR_TYPE_UNSIGNED_YCBCR888_420,
  STGXOBJ_COLOR_TYPE_UNSIGNED_AYCBCR6888_444,
  STGXOBJ_COLOR_TYPE_SIGNED_AYCBCR8888,
  STGXOBJ_COLOR_TYPE_UNSIGNED_AYCBCR8888,

  STGXOBJ_COLOR_TYPE_ALPHA1,
  STGXOBJ_COLOR_TYPE_ALPHA4,
  STGXOBJ_COLOR_TYPE_ALPHA8,
  STGXOBJ_COLOR_TYPE_BYTE,

  STGXOBJ_COLOR_TYPE_ARGB8888_255,
  STGXOBJ_COLOR_TYPE_ARGB8565_255,
  STGXOBJ_COLOR_TYPE_ACLUT88_255,
  STGXOBJ_COLOR_TYPE_ALPHA8_255

} STGXOBJ_ColorType_t;

/* 颜色值 */
typedef union STGXOBJ_ColorValue_u
{
  STGXOBJ_ColorARGB_t           ARGB8888;
  STGXOBJ_ColorRGB_t            RGB888;
  STGXOBJ_ColorARGB_t           ARGB8565;
  STGXOBJ_ColorRGB_t            RGB565;
  STGXOBJ_ColorARGB_t           ARGB1555;
  STGXOBJ_ColorARGB_t           ARGB4444;

  U8                            CLUT8;
  U8                            CLUT4;
  U8                            CLUT2;
  U8                            CLUT1;
  STGXOBJ_ColorACLUT_t          ACLUT88 ;
  STGXOBJ_ColorACLUT_t          ACLUT44 ;

  STGXOBJ_ColorSignedYCbCr_t    SignedYCbCr888_444;
  STGXOBJ_ColorUnsignedYCbCr_t  UnsignedYCbCr888_444;
  STGXOBJ_ColorSignedYCbCr_t    SignedYCbCr888_422;
  STGXOBJ_ColorUnsignedYCbCr_t  UnsignedYCbCr888_422;
  STGXOBJ_ColorSignedYCbCr_t    SignedYCbCr888_420;
  STGXOBJ_ColorUnsignedYCbCr_t  UnsignedYCbCr888_420;
  STGXOBJ_ColorUnsignedAYCbCr_t UnsignedAYCbCr6888_444;
  STGXOBJ_ColorSignedAYCbCr_t   SignedAYCbCr8888;
  STGXOBJ_ColorUnsignedAYCbCr_t UnsignedAYCbCr8888;

  U8                            ALPHA1;
  U8                            ALPHA4;
  U8                            ALPHA8;
  U8                            Byte;

} STGXOBJ_ColorValue_t;


/* ARGB 颜色值 */
typedef struct STGXOBJ_ColorARGB_s
{
  U8 Alpha;    // 这里的alpha值范围依旧是： 0 ~ 128
  U8 R;
  U8 G;
  U8 B;
} STGXOBJ_ColorARGB_t;

/* RGB 颜色值 */
typedef struct STGXOBJ_ColorRGB_s
{
  U8 R;
  U8 G;
  U8 B;
} STGXOBJ_ColorRGB_t;

```

### 同步
同步的意思就是说，等待底层 blit 命令完成后，再执行后面的操作。如果不同步的话，就会遇到我日志里的问题。这里同步的方法理论上有2种，但是目前我只弄成功了一种：

* 使用 ST 的通知回调。前面看到过，blit 的上下文参数里有一个就是设置回调参数的。这个回调是在 blit 完成后，底层调用的。不过要用这个比较麻烦。流程好像是：
    * 先要打开 stevt_ioctl 这个设备，取得stevt设置handle。
    * 然后注册通知设备（这个好像是说哪个blit设备吧，这里没太搞清楚）。
    * 通过得到的 handle 在 blit 上下文参数设置回调函数（通过 stevt 设备的 ioctl 命令）。
    * 使用完后，删除注册通知设备。
    * 退出时，关闭 stevt 设备。
具体的可以看看 ST 那个控制台的代码。不过我弄了下，发现不行，在 blit 上下文那些一设置回调就包内核错误了 -_-||。估计是哪里没弄对。算了，不管了，下面有比这个更好用的～～嘿嘿（这个就算回调弄成功了，回调里锁这、锁那的也麻烦）。

* 使用 stgfb 提供的blit设备同步 ioctl 命令。上面列出的的 stgfb io 接口里有这个一个接口：STGFB_IO_SYNC_BLITTER 。用这个就可以了。这个命令其实是没有参数的，ioctl 最后那个参数直接赋值0就可以了。在 blit ioctl 后面调用这个，能保证 blit 完成后，ioctl 才返回。这个正是我们需要的功能。用这个简单的 ioctl 命令就OK了，不用使用 ST 的底层事件功能了：

代码：

```cpp
/* sync the blitter, no need any params */
if (wait_sync) {
    if (ioctl(fd, STGFB_IO_SYNC_BLITTER, 0) != 0) { 
        perror(strerror(errno));
        fprintf(stderr, "STAPI_Blit >> Sync Engine failed\n");
    }
}
```

### 显存管理
板子上可用来分配给 MiniGUI memdc（离屏surface） 的显存一般是除去了屏幕暂用的显存（pitch x 屏幕高度）之后所剩余的（一般板子上的显存都会比屏幕大，用总显存减去屏幕所暂用的显存就可以得到剩下的显存大小了）。这样就需要一定的管理策略来有效的管理这些显存（要求分配到的显存要是连续的）。目前的 GAL 采用的办法是：

1. 将显存分块（block）。用双向链表（借用linux内核的链表实现）来管理这些块（bucket）。块中保存了块的pitch、高度、相对于显存首地址的偏移、以及是已经使用的标志。初始化时，将所有的显存创建成一个标记未使用过的大块。

2. 当新申请一块显存时（传入所需块的pitch和高度）：通过传入的参数计算所需要申请的显存大小。从链表头开始查找没有使用过的并且足够大的块。如果找到了就把这个块拆成2块：一块是申请的大小（新插入到这个块的前面），一块是原来剩余，并且把新申请到的块标记为使用的；如果没找到就返回失败。

3. 当释放一块显存时（传入block的指针）：分别向这块显存的前一个和后一个块检查。看临近的这2块的标记是否是未使用的，如果是则合并这2块（在链表中删掉其中一块，并把这一块的大小加入到未删除块上，修改未删除块相应的偏移信息），并设置标志为未使用。这样坐的目的是为了尽可能的保持未使用的显存的连续性，从而尽量保证申请显存的成功率。

下面举个简单的例子：显存有10M，初始化时，分配屏幕显存（1920x1080-16bpp），需要一个 640x480-16bpp 的 surface （图中没写偏移地址）：

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/sti7167-gal/3.png)

然后使用完成后，就要释放。会先查找前面一个块，这个块是正在使用的，不合并；然后查找到后面一个块，这个块是没有使用的，所以就要合并。如果是前、后都在使用的，那就简单的把这个块标记为未使用即可（从图中可以看出，这种策略的特点：第一个块一般是屏幕的显存，最后一个块一般是最大的未使用的块；链表中间有可能会出现一些小的未使用的块，但是随着临近块的释放，这些块会慢慢的合并到最后的大块中）：

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/sti7167-gal/4.png)

贴贴代码，对着看下呗：

```cpp
int vmbucket_init(vmbucket_t *bucket, unsigned char *start, int size) {
    bucket->start = start;
    bucket->size = size;
    INIT_LIST_HEAD(&bucket->block_head);

    {
        vmblock_t *block;
        block = (vmblock_t *)calloc(1, sizeof(*block));
        block->offset = 0;
        block->height = 1;
        block->pitch = size;
        list_add(&block->list, &bucket->block_head);
    }
    return 0;
}

void vmbucket_destroy(vmbucket_t *bucket) {
    while (! list_empty(&bucket->block_head)) {
        vmblock_t *block = (vmblock_t *)bucket->block_head.prev;
        list_del(bucket->block_head.prev);
        free(block);
    }
}

#define TEST_BIT(a, bit) ((a) & (bit))
#define SET_BIT(a, bit) ((a) |= (bit))
#define UNSET_BIT(a, bit) ((a) &= ~(bit))
#define BLOCK_SIZE(block) ((block)->height * (block)->pitch)

vmblock_t *vmbucket_alloc(vmbucket_t *bucket, int pitch, int height) {
    struct list_head *i;
    int required_size = height * pitch;

    list_for_each(i, &bucket->block_head) {
        vmblock_t *block;
        block = (vmblock_t *)i;
        if (! TEST_BIT(block->flag, VMBLOCK_FLAG_USED)) {
            int block_size = BLOCK_SIZE(block);
            if (required_size > block_size) {
                continue;
            }else if (required_size < block_size) {
                vmblock_t *new_block;
                new_block = (vmblock_t *)calloc(1, sizeof(*new_block));
                new_block->offset = block->offset + required_size;
                new_block->pitch = block_size - required_size;
                new_block->height = 1;
                list_add(&new_block->list, &block->list);
            }
            block->height = height;
            block->pitch = pitch;
            SET_BIT(block->flag, VMBLOCK_FLAG_USED);
            return block;
        }
    }
    return NULL;
}

void vmbucket_free(vmbucket_t *bucket, vmblock_t *block)
{
    int size = BLOCK_SIZE(block);
    vmblock_t *near_block;

    if (!TEST_BIT(block->flag, VMBLOCK_FLAG_USED)) {
        fprintf(stderr, "videomem-bucket: can't free an unuse block!\n");
        return;
    }
    UNSET_BIT(block->flag, VMBLOCK_FLAG_USED);

    /* look for prev */
    if (block->list.prev != &(bucket->block_head)) {
        near_block = (vmblock_t *)block->list.prev;
        if (!TEST_BIT(near_block->flag, VMBLOCK_FLAG_USED)) {
            size += BLOCK_SIZE(near_block);
            block->offset = near_block->offset;
            block->height = 1;
            block->pitch = size;
            list_del(&near_block->list);
            free(near_block);
        }
    }

    /* look for next */
    if (block->list.next != &(bucket->block_head)) {
        near_block = (vmblock_t *)block->list.next;
        if (!TEST_BIT(near_block->flag, VMBLOCK_FLAG_USED)) {
            size += BLOCK_SIZE(near_block);
            block->height = 1;
            block->pitch = size;
            list_del(&near_block->list);
            free(near_block);
        }
    }
}
```

### overlapped blit
overlapped blit 的意思就是说：源数据地址与目的数据地址有重叠，见下面的图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/minigui/sti7167-gal/5.png)

如果是软件 blit 的话（使用 framebuffer 映射到的内存来 memcpy），是不需要考虑这种情况的。或是有些硬件处理了这种情况，也不需要考虑。7167 底层没处理，所以就要自己处理。overlapped 造成的问题是：一般来说底层的复制是逐行、逐行的复制的，所以上面的这种情况，上面的数据会把重叠的地方的给覆盖了，当复制到重叠的地方（就是源数据的下面部分）就会把被覆盖后的数据复制到目标区域，这样就会原来的图像就不正确了。解决办法有2个：

1. 创建一个和源一样大的临时显存块，把源先 blit 到临时显存块，然后再把临时显存块 blit 到目标区域，最后释放临时显存块。

2. 把源区域分块 blit ，保证不重叠。

第一种方法不许要计算分块，blit 次数固定为2；但是需要创建临时显存块，如果源比较大，并且显存使用比较多的情况下，会造成 blit 失败。所以采用第2种方法。分块的策略为（这里以图中的垂直向下为例，垂直向上，水平向左、水平向右是类似的）：

1. 计算出相交的区域矩形大小（图中绿色的部分）。

2. 用相交区域的高与源区域的高相减得到未相交区域的大小（delta）。

3. 将源区域每次按 delta 大小从下向上 blit，当最后区域不足 delta 时就取最后剩下区域的大小（这样做的目的就是为了每次 blit 都不相交）。

这里讨论了水平、垂直的情况（4种），还有剩下4种斜的情况。因为斜的情况，水平、垂直的策略都适用，但是不同的情况使用不同的策略速度会不同。仔细看下相交区域：当相交区域的高比较大的时候，垂直分块策略会比较慢；当相交区域的宽比较大的情况下，水平分块策略会比较慢。所以这里如果是斜的情况，可以根据相交区域来选择最快的分块策略。贴一下代码吧，这样对着看会比较清晰些：

```cpp
int own_overlapped_bitblit(GAL_blit real_blit, struct GAL_Surface *src, GAL_Rect *srcrect,
        struct GAL_Surface *dst, GAL_Rect *dstrect) {
    int w, W, x;
    int h, H, y;
    int delta;
    int ret = 0;
    RECT intersect, src_rc, dst_rc;

    assert(srcrect->w == dstrect->w || srcrect->h == dstrect->h);

    galrect_2_rect(srcrect, &src_rc);
    galrect_2_rect(dstrect, &dst_rc);

    /* don't intersect or horizontal left or up or left-up overlapped blit directly */
    if (! IntersectRect(&intersect, &src_rc, &dst_rc)
            || (dst_rc.top <= src_rc.top && dst_rc.left <= src_rc.left)) {
        return real_blit(src, srcrect, dst, dstrect);
    }

    GAL_Rect src1, dst1;

    /* vertical overlapped. 
     * if left-up right-up left-down right-down, horizontal is faster than vertical than use horizonatl mode. */
    if ( (dst_rc.left == src_rc.left) 
            || ((dst_rc.top != src_rc.top) && (RECTH(intersect) <= RECTW(intersect))) ) {
        h = RECTH(intersect);
        H = RECTH(src_rc);
        delta = H - h;

        /* down or left-down or right-down overlapped, separate into per not-Intersect rect to blit,
         * from bottom to up */
        if (dst_rc.top > src_rc.top) {
            for (y = H; y > 0; y -= delta) {
                if (y < delta) {
                    delta = y;
                }
                src1.x = src_rc.left;
                src1.y = src_rc.top + y - delta;
            
                dst1.x = dst_rc.left;
                dst1.y = dst_rc.top + y - delta;
            
                src1.w = dst1.w = srcrect->w;
                src1.h = dst1.h = delta;

                ret |= real_blit(src, &src1, dst, &dst1);
            }
        }
        /* up or left-up or right-up overlapped, just the same down except from up to bottom */
        else {
            for (y = 0; y < H; y += delta) {
                if (y + delta > H) {
                    delta = H - y;
                }
                src1.x = src_rc.left;
                src1.y = src_rc.top + y;
            
                dst1.x = dst_rc.left;
                dst1.y = dst_rc.top + y;
            
                src1.w = dst1.w = srcrect->w;
                src1.h = dst1.h = delta;

                ret |= real_blit(src, &src1, dst, &dst1);
            }
        }
    }
    /* horizontal overlapped */
    else {
        w = RECTW(intersect);
        W = RECTW(src_rc);
        delta = W - w;
        
        /* horizontal right overlapped, separate into per not-Intersect rect to blit,
         * from right to left */
        if (dst_rc.left > src_rc.left) {
            for (x = W; x > 0; x -= delta) {
                if (x < delta) {
                    delta = x;
                }
                src1.x = src_rc.left + x - delta;
                src1.y = src_rc.top;
            
                dst1.x = dst_rc.left + x - delta;
                dst1.y = dst_rc.top;
            
                src1.w = dst1.w = delta;
                src1.h = dst1.h = srcrect->h;

                ret |= real_blit(src, &src1, dst, &dst1);
            }
        }
        /* horizontal left overlapped. just the same right except from left to right.
         * this just for optimization overlapped blit speed. */
        else {
            for (x = 0; x < W; x += delta) {
                if (x + delta > W) {
                    delta = W - x;
                }
                src1.x = src_rc.left + x;
                src1.y = src_rc.top;
            
                dst1.x = dst_rc.left + x;
                dst1.y = dst_rc.top;
            
                src1.w = dst1.w = delta;
                src1.h = dst1.h = srcrect->h;

                ret |= real_blit(src, &src1, dst, &dst1);
            }
        }
    }

    return ret;
}
```

### 进程版
MiniGUI 的多进程间通信通过 Unix socket ，采用 C/S 模型（桌面是服务器，其它的应用进程是客户端）。 在线程版的基础修改以下的地方：

* 初始化的时候，需要的设备（framebuffer、stlayer等）每个进程都要打开。frambuffer 的内存映射（mmap）每个进程要做，因为 mmap 是映射到当前进程的地址空间，所以如果只有服务器映射的话，就算给了客户端地址，客户端也无法访问。每个进程都要计算屏幕相应的内存映射地址偏移（这个是用户空间的地址，每个进程的都不一样的，因为之前各自映射各自的，所以每个客户端都要做）。只在服务器初始化显存管理链表（之前说的bucket），同时申请屏幕显存（分配硬件显存地址，一般只能做一次，所以只在服务器进程分配）。

* 退出时，每个进程都要关闭打开的设备（因为之前每个进程都打开了）。每个进程都要卸载 framebuffer 的内存映射（munmap）。只在服务器销毁显存管理链表（因为只在服务器创建了）。

* 客户端向服务器发申请显存请求；服务器响应客户端请求并分配显存。客户端向服务器发送释放显存请求；服务器响应客户端请求并释放显存。这里要注意一点：一个进程是不允许访问另外一个进程的内存地址空间的（这里的是指用户态的，虚拟的内存地址），除非是创建共享内存（这个好像稍微麻烦点）。这样不管是服务器向客户端发送数据分配的显存数据（MiniGUI 里一般是一个 void* 类型，保证的是数据的地址），还是客户端向服务器发送要释放的显存数据；如果像多线程那样直接访问的话，是要出信号失败的错误的。解决办法一个是创建共享内存（这个说了会麻烦点）；还有一个就是不要访问地址（指针），只访问数值。这里的技巧就是：

    * 显存申请时：服务器分配显存是会申请一个 block（这是个结构体），这里把 block 指针、block 的 offset 和 pitch 发送给客户端。客户端得到数据后，把服务器发过来的 block 指针保存下来，这里注意仅仅是保存下这个指针（地址）就行了，不要去访问（原因前面说了）。然后根据得到的 block 的 offset 和 pitch 计算出物理显存地址和映射的framebuffer地址（客户端需要的 block 数据仅仅是 offset 和 pitch）。这样显存申请就 OK 了。

    * 显存释放时：客户端把之前服务器发过来的 block 地址发送给服务器。服务器就可以拿着这个 block 地址调用 bucket 的释放函数释放显存了。这里的 block 是服务器的进程创建的，所以服务器端就可以访问。

### 使用 stapi 的注意事项

* 要包括 ST 的一些头文件（我把这些复制到 MiniGUI 源码树里去了）：
<pre config="brush:bash;toolbar:false;">
    * src/newgal/stgfb/st_include/stblit_ioctl.h
    * src/newgal/stgfb/st_include/stddefs.h
    * src/newgal/stgfb/st_include/stcommon.h
    * src/newgal/stgfb/st_include/stevt_ioctl.h
    * src/newgal/stgfb/st_include/linuxwrapper.h
    * src/newgal/stgfb/st_include/stblit.h
    * src/newgal/stgfb/st_include/stlite.h
    * src/newgal/stgfb/st_include/stevt.h
    * src/newgal/stgfb/st_include/stgfb.h
    * src/newgal/stgfb/st_include/stdevice.h
    * src/newgal/stgfb/st_include/linuxcommon.h
    * src/newgal/stgfb/st_include/stavmem.h
    * src/newgal/stgfb/st_include/stlayer_ioctl.h
    * src/newgal/stgfb/st_include/stgxobj.h
    * src/newgal/stgfb/st_include/stlayer.h
    * src/newgal/stgfb/st_include/stsys.h
    * src/newgal/stgfb/st_include/layer_rev.h
    * src/newgal/stgfb/st_include/stos.h
</pre>

* 要加上几个编译宏：
<pre>
CFLAGS="$CFLAGS -DST_OSLINUX -DST_7105 -DARCHITECTURE_ST40 -DDEFINED_BOOL"
</pre>

* 编译环境要包括 linux 内核头文件。这个在 ST 的根文件系统的包里有，我默认安装就在这：
<pre>
/opt/STM/STLinux-2.3/devkit/sources/kernel/linux-sh4
</pre>

## 编译、运行 MiniGUI （DirectFB）
... ...

## 编译、运行 MiniGUI （stgfb）

### MiniGUI source
rel-3-0 分支 svn 13675 以上： URL: svn+ssh://devsrv/home/projects/svn/minigui/branches/rel-3-0

### 编译环境设置
按上面写的安装好 STLinux。这里假设你安装的路径是 /opt/STM/STLinux-2.3，那么：

* 交叉工具链路径是： /opt/STM/STLinux-2.3/devkit/sh4/bin
* 根文件系统路径是： /opt/STM/STLinux-2.3/devkit/sh4/target
* kernel源码路径是： /opt/STM/STLinux-2.3/devkit/sources/kernel/linux-sh4

要编译基于 stgfb GAL 的 MiniGUI 的话要设置如下的 CFLAGS 和 LDFLAGS：

<pre config="brush:bash;toolbar:false;">

/* 设置安装路径 */
export PREFIX="/home/mingming/nfs/st7167_procs"

/* 设置根文件系统的头文件路径  */
export SH4_ROOT=/opt/STM/STLinux-2.3/devkit/sh4
export KERNEL_ROOT=/opt/STM/STLinux-2.3/devkit/sources/kernel/linux-sh4

/* 要特别设置下 c++ 的头文件路径，编译 mgplus、mdolphin、mdtv 用 */
cxx_path="${SH4_ROOT}/target/usr/include/c++/4.2.1"

/* 设置 CFLAGS，包括根文件系统自带的一些头文件、和内核头文件 */
export CFLAGS="-I. -I.. -I${PREFIX}/include  -I${SH4_ROOT}/include -I${SH4_ROOT}/target/usr/include -I${SH4_ROOT}/taget/usr/local/include -I${KERNEL_ROOT}/include"

/* 设置 C++ flag，除了包括 CFLAGS 的一些头文件路径还要包括 c++ 文件路径 */
export CPPFLAGS="${CFLAGS} -I${cxx_path} -I${cxx_path}/sh4-linux"
export CXXFLAGS=$CPPFLAGS

/* 设置 pkg 路径，包括编译安装路径和根文件系统的，这个不设置的话，编译会报错 */
export PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig:$SH4_ROOT/target/usr/lib/pkgconfig

/* 设置 LDFLAGS，注意如果 MiniGUI 打开了 freetype，要自己编译个高版本的 freetype（用 2.3.9 版本的），根文件系统自带的不行。
 * 所以设置的路径要把安装路径设置到根文件系统的前面 */
export libs_base="-L${PREFIX}/lib -Wl,-rpath-link=${PREFIX}/lib -L${SH4_ROOT}/target/lib -L${SH4_ROOT}/target/usr/lib -L${SH4_ROOT}/target/usr/local/lib -Wl,-rpath-link=${SH4_ROOT}/target/lib -Wl,-rpath-link=${SH4_ROOT}/target/usr/lib -Wl,-rpath-link=${SH4_ROOT}/target/usr/local/lib"
export extra_libs="-lm -lpthread -ljpeg -lpng -lfreetype"
export LDFLAGS="${libs_base} ${extra_libs}"

/* 设置 bin 路径，这里主要设置交叉编译链路径和一些库的 config 路径，注意先后顺序 */
extra_path="${SH4_ROOT}/bin"
export PATH=$PREFIX/bin:$extra_path:/home/mingming/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

</pre>

### MiniGUI 编译配置
<pre>
configure --prefix=$PREFIX \
    --build=i386-linux \
    --host=sh4-linux \
    --enable-videofbcon \
    --enable-videostgfb \
    --enable-procs \
    --disable-videoqvfb \
    --disable-pcxvfb \
    --disable-qvfbial \
    --with-ttfsupport=ft2 \
    --enable-ttfcache \
    --enable-jpgsupport \
    --enable-pngsupport \
    --disable-splash \
    --disable-screensaver
</pre>

### MiniGUI 运行时配置
可以使用不带加速的 fbcon 来验证下带加速的 stgfb 。如果鼠标设备位置不对，可以改为 mdev=/dev/mouse0。framebuffer 的颜色格式设置可以参看上面写的改 load_fb.sh 里的设置。
<pre>
[system]
# GAL engine and default options
#gal_engine=fbcon
gal_engine=stgfb
defaultmode=1920x1080-16bpp

# IAL engine
ial_engine=console
mdev=/dev/input/mice
mtype=IMPS2
</pre>

最后还要设置下系统运行时环境，这里提供一个脚本（要先安装howto文档里，先把 ST 的控制台先跑来， 然后另外一个console telnet登录上去运行该脚本；注意脚本的运行方式：soucre xx.sh 或是 . ./xx.sh）：
<pre config="brush:bash;toolbar:false;">
/* howto 文档里写的加载 framebuffer 设备，分辨率 1920x1080 */
source /root/load_fb.sh FULLHD PAL 

/* 挂载 sawp，没有的话，可以删掉这个 */
mkswap /dev/sda
swapon /dev/sda

/* 设置动态库路径，非重重要。这里我将 MiniGUI 的安装路径设上去了。 */
export LD_LIBRARY_PATH=/root/nfs_ths/lib:$LD_LIBRARY_PATH
export MG_CFG_PATH=/root/nfs_ths/myetc
export MG_RES_PATH=/root/nfs_ths/share/minigui/res

/* 挂载 nfs，上面的 MiniGUI 安装路径就是这里挂载上去的 */
umount /root/nfs_ths
mount 192.168.1.102:/home/mingming/nfs/st7167 /root/nfs_ths -o nolock

/* 设置 ST 环境，这个可有、可无，最好还是加上 */
source modules/load_env.sh
</pre>

### 注意事项
* 链接时，会出现乱链库的情况。这个时候要去把根文件系统下把那些自带库的 .la 文件里的路径改对；还有一些 bin 下的 config 文件，也要改对。

* 板子出现无法解析host的情况，修改 /etc/resolv.conf 文件，设置正确的 nameserver ，例如，目前深圳这边的设置为 192.168.1.1


