title: Android x86 在 X window system 上的调研
date: 2015-01-27 23:14:16
updated: 2015-01-27 23:14:16
categories: [Android Framework]
tags: [android]
---

这个是以前弄到一个项目：把 Android 应用程序跑到 Meego。那个时候 Android 官方还不支持 X86 平台，所以折腾，现在 Android 官方支持了，这个其实就没什么用了，权当纪念一下以前扯蛋的项目吧。

## 需求
我们的最终目的是为了让 Android 在 Meego 上（GUI 是 x86 的 X window system）能有接近原生的运行速度。这样的话就不能再采用 demo 里的 vfb 的做法。而应该是让 Android 直接在屏幕上进行绘制（显存中），并且充分的利用平台的硬件加速能力（Intel 945GME），例如使用硬件的 OpenGL 实现等。这样的话就需要利用 X window system 中的原生底层接口来实现 Android 的低层图形驱动模块。所以需要调研 X window system 的一些原理、用法、如何利用硬件加速的。

## 相关名词
先大概的说一下一些相关的名词吧：（为了方便起见，以下的 Meego 都是说的 netbook 1.1 版本）

* **X window system**
由于其主版本号是 11，因此又叫 X11 。X11 是 linux 上的一个窗口系统，采用 C/S 模型，因此它可以方便的使用于网络中（分布式结构）。但是这也给我们的项目提供了困难，因为上网本上的 Meego 是确定只在本地使用的。X11 会有一个服务端，每一显示设备都是它的一个客户端。客户端通过发送请求到服务端申请显示设备，通过发送请求让服务器绘制图像显示到指定的显示设备上（一般是自己申请的那个）。Meego 上使用的 Xlib 是 1.3.3 的，Xorg-server 是 1.9.0 的，好像在后面的源码路径上都找不到版本一样的，只能找一些接近的了。源码在： [X11源代码路径](http://www.x.org/releases/ "X11源代码路径")。

* **Xlib**
X11 有很多个模块。Xlib 是它的基本模块，提供了一些基本的图形访问接口。此外 X11 还可以有一些列的扩展接口，只要符合 X11 的扩展规范，就可以。X11 上的 OpenGL 就是通过 X11 的扩展来衔接的。

* **GLX**
OpenGL 是一个 3D 图形库，它是独立于窗口系统的。但是如果要在窗口系统中使用，这就必须要和窗口系统建立起联系，例如说如何在窗口中渲染 3D 图形等等。这样就需要一个桥梁来建立起 OpenGL 和本地窗口系统的衔接。在 X11 上这个就是 GLX 要做的事情（类似于嵌入式系统上的 EGL）。Meego 上使用的是 1.4 版本。其其源代码包括在 Mesa 里面。GLX 走的是 X11 的协议，是通过和 X server 打交道来进行渲染的，因此又叫作间接渲染（indirect render）。

* **DRM & DRI**
DRI 全称 Direct Rendering Infrastructure。由于 X11 是采用 C/S 架构的，客户端的任何操作都需要和服务器进行通讯，在实时的 3D 渲染上性能无法接受。因此就出现了 DRI 这种框架，在 X11 上能够允许直接访问硬件渲染器（显卡），从而直接将 3D 图形渲染到屏幕上，绕过 X11 ，提升性能，这种叫作直接渲染（direct render）。DRI 为上层 3D 库提供访问底层硬件的接口。DRM 全称 Direct Rendering Manager，从字面上理解为直接渲染管理器，是真正操作硬件的层次。各个硬件厂商负责提供各自硬件的 drm 模块（开源的提供源码、不开源的提供二进制文件）。DRI 通过调用 DRM 的接口来实现上层 3D 图形库的接口。HP mini 上使用的是 Intel 的 945GME 显卡，使用的 DRM 源码在：git://git.kernel.org/pub/scm/linux/kernel/git/ickle/drm-intel（git 路径）。DRI 的源码则在 Mesa 中。

* **Mesa**
OpenGL 只是一个规范，它的实现有很多种。Mesa 是 OpenGL 在 linux 上的一个实现。Mesa 自身提供一套纯软件的 OpenGL 实现，能在各个平台运行 OpenGL 程序，不过速度非常慢。其中 Mesa 会根据系统平台的 dri 和 drm 情况调用硬件的 OpenGL 实现，这时就是硬件的 3D 加速，速度会比纯软件的快得多。从上面来看流程应该是： 3D app --> Mesa(OpenGL API) --> dri --> drm --> hardware。Meego 上使用的是 Mesa 7.9 。源码在：[Mesa 源代码路径](ftp://ftp.freedesktop.org/pub/mesa/ "Mesa 源代码路径")。

* **SDL**
SDL 是一个开源跨平台的图形库，它小巧、简单、快速，提供在 linux 上简单的 GUI 接口。目前的 vfb 就是用 SDL 实现的，在这个方面它比 Qt 简介、方便。在 Meego 上使用的是 1.2.14 版本。源代码在 [SDL源代码路径](http://www.libsdl.org/download-1.2.php "SDL源代码路径")。

* **Mutter Window Manager**
X11 只是提供了一些基本的 GUI 元素，但是对于窗口如果显示、窗口移动了怎么办、窗口尺寸改变了又怎么办、多个窗口重叠在一起怎么显示等等，X11 是不会管的。这个些就需要一个叫做窗口管理系统的软件来管理。Meego 上使用的窗口管理系统就是 Mutter Window Manager 。一般 X11 上的窗口管理系统都是基于 Xlib 开发的。源代码在：[Mutter源代码路径](http://ftp.acc.umu.se/pub/GNOME/sources/mutter/ "Mutter源代码路径") （在 Meego 源代码里也有）。

* **Clutter**
Clutter是一个跨平台的图形库，能够支持 OpenGL。Mutter Window Manager 使用的图形库是就是 Clutter。Clutter 采用 glib 提供的 gobject 实现面向对象设计，采用多种 backend 的架构，实现了跨平台。在 linux 上使用的 backend 是 GLX 和 X11 。Meego 上使用的 Clutter 版本是 1.2.8。源码在：[Clutter源代码路径](http://source.clutter-project.org/sources "Clutter源代码路径")

* **Cogl**
在 Clutter 中使用的 OpenGL 接口。源代码包含在 Clutter 中。

## 实现思路
在 Android x86 的代码里，底层有一个叫 gralloc 的模块，是用来管理和分配显存的。我们只要能向 X11 拿到显存地址，就可以分配给上层 Android 程序，从而让 Android 直接绘制到屏幕上（相当于使用本地的一些接口实现 gralloc），这样可以去掉 vfb，这样可以让 Android 原生的跑在 X11 上。并且 Android 在 2D 的块传送上还有一个专门的硬件加速的 copyblt 的模块，我们如果通过 X11 的一些接口，或者是说直接通过显卡驱动的一些接口实现这个模块，也就能相应的完成 Android 2D 加速。至于 OpenGL 的 3D 模块，Android 完整的实现了一套软件的 OpenGL（好像是 ES 的），我们是不是可以用 Mesa 3D 的替换原来的。因此先需要了解 X11 上的绘图相关的东西，看看如果获取显存地址（像素地址）。

## Xlib 分析
Xlib 是 X11 上最基本的东西了，这个是肯定是要研究下的。

### Xlib 基本数据结构
* **Display**
若干个屏幕(Screen)以及一套输入设备（键盘和鼠标）构成一个 Display，Display 概念的关键就是有一套完整的输入输出。屏幕不一定必须是一个，可以有多个，各个屏幕可以用来显示相同的内容，也可以用来构成矩阵显示一个大屏幕的内容。X11 中客户端通过打开 Display 和服务器相连接。要在 X11 上创建一个窗口，第一步就是需要打开一个 Display。它的数据结构是：

代码：

```cpp
// 在 X11/Xlib.h 中的定义
typedef struct
#ifdef XLIB_ILLEGAL_ACCESS
_XDisplay
#endif
{
	XExtData *ext_data;	/* hook for extension to hang data */
	struct _XPrivate *private1;
	int fd;			/* Network socket. */
	int private2;
	int proto_major_version;/* major version of server's X protocol */
	int proto_minor_version;/* minor version of servers X protocol */
	char *vendor;		/* vendor of the server hardware */
        XID private3;
	XID private4;
	XID private5;
	int private6;
	XID (*resource_alloc)(	/* allocator function */
		struct _XDisplay*
	);
	int byte_order;		/* screen byte order, LSBFirst, MSBFirst */
	int bitmap_unit;	/* padding and data requirements */
	int bitmap_pad;		/* padding requirements on bitmaps */
	int bitmap_bit_order;	/* LeastSignificant or MostSignificant */
	int nformats;		/* number of pixmap formats in list */
	ScreenFormat *pixmap_format;	/* pixmap format list */
	int private8;
	int release;		/* release of the server */
	struct _XPrivate *private9, *private10;
	int qlen;		/* Length of input event queue */
	unsigned long last_request_read; /* seq number of last event read */
	unsigned long request;	/* sequence number of last request. */
	XPointer private11;
	XPointer private12;
	XPointer private13;
	XPointer private14;
	unsigned max_request_size; /* maximum number 32 bit words in request*/
	struct _XrmHashBucketRec *db;
	int (*private15)(
		struct _XDisplay*
		);
	char *display_name;	/* "host:display" string used on this connect*/
	int default_screen;	/* default screen for operations */
	int nscreens;		/* number of screens on this server*/
	Screen *screens;	/* pointer to list of screens */
	unsigned long motion_buffer;	/* size of motion buffer */
	unsigned long private16;
	int min_keycode;	/* minimum defined keycode */
	int max_keycode;	/* maximum defined keycode */
	XPointer private17;
	XPointer private18;
	int private19;
	char *xdefaults;	/* contents of defaults from server */
	/* there is more to this structure, but it is private to Xlib */
}

// 在 X11/Xlibint.h 中的定义
struct _XDisplay
{
	XExtData *ext_data;	/* hook for extension to hang data */
	struct _XFreeFuncs *free_funcs; /* internal free functions */
	int fd;			/* Network socket. */
	int conn_checker;         /* ugly thing used by _XEventsQueued */
	int proto_major_version;/* maj. version of server's X protocol */
	int proto_minor_version;/* minor version of server's X protocol */
	char *vendor;		/* vendor of the server hardware */
        XID resource_base;	/* resource ID base */
	XID resource_mask;	/* resource ID mask bits */
	XID resource_id;	/* allocator current ID */
	int resource_shift;	/* allocator shift to correct bits */
	XID (*resource_alloc)(	/* allocator function */
		struct _XDisplay*
		);
	int byte_order;		/* screen byte order, LSBFirst, MSBFirst */
	int bitmap_unit;	/* padding and data requirements */
	int bitmap_pad;		/* padding requirements on bitmaps */
	int bitmap_bit_order;	/* LeastSignificant or MostSignificant */
	int nformats;		/* number of pixmap formats in list */
	ScreenFormat *pixmap_format;	/* pixmap format list */
	int vnumber;		/* Xlib's X protocol version number. */
	int release;		/* release of the server */
	struct _XSQEvent *head, *tail;	/* Input event queue. */
	int qlen;		/* Length of input event queue */
	unsigned long last_request_read; /* seq number of last event read */
	unsigned long request;	/* sequence number of last request. */
	char *last_req;		/* beginning of last request, or dummy */
	char *buffer;		/* Output buffer starting address. */
	char *bufptr;		/* Output buffer index pointer. */
	char *bufmax;		/* Output buffer maximum+1 address. */
	unsigned max_request_size; /* maximum number 32 bit words in request*/
	struct _XrmHashBucketRec *db;
	int (*synchandler)(	/* Synchronization handler */
		struct _XDisplay*
		);
	char *display_name;	/* "host:display" string used on this connect*/
	int default_screen;	/* default screen for operations */
	int nscreens;		/* number of screens on this server*/
	Screen *screens;	/* pointer to list of screens */
	unsigned long motion_buffer;	/* size of motion buffer */
	volatile unsigned long flags;	   /* internal connection flags */
	int min_keycode;	/* minimum defined keycode */
	int max_keycode;	/* maximum defined keycode */
	KeySym *keysyms;	/* This server's keysyms */
	XModifierKeymap *modifiermap;	/* This server's modifier keymap */
	int keysyms_per_keycode;/* number of rows */
	char *xdefaults;	/* contents of defaults from server */
	char *scratch_buffer;	/* place to hang scratch buffer */
	unsigned long scratch_length;	/* length of scratch buffer */
	int ext_number;		/* extension number on this display */
	struct _XExten *ext_procs; /* extensions initialized on this display */
	/*
	 * the following can be fixed size, as the protocol defines how
	 * much address space is available.
	 * While this could be done using the extension vector, there
	 * may be MANY events processed, so a search through the extension
	 * list to find the right procedure for each event might be
	 * expensive if many extensions are being used.
	 */
	Bool (*event_vec[128])(	/* vector for wire to event */
		Display *	/* dpy */,
		XEvent *	/* re */,
		xEvent *	/* event */
		);
	Status (*wire_vec[128])( /* vector for event to wire */
		Display *	/* dpy */,
		XEvent *	/* re */,
		xEvent *	/* event */
		);
	KeySym lock_meaning;	   /* for XLookupString */
	struct _XLockInfo *lock;   /* multi-thread state, display lock */
	struct _XInternalAsync *async_handlers; /* for internal async */
	unsigned long bigreq_size; /* max size of big requests */
	struct _XLockPtrs *lock_fns; /* pointers to threads functions */
	void (*idlist_alloc)(	   /* XID list allocator function */
		Display *	/* dpy */,
		XID *		/* ids */,
		int		/* count */
		);
	/* things above this line should not move, for binary compatibility */
	struct _XKeytrans *key_bindings; /* for XLookupString */
	Font cursor_font;	   /* for XCreateFontCursor */
	struct _XDisplayAtoms *atoms; /* for XInternAtom */
	unsigned int mode_switch;  /* keyboard group modifiers */
	unsigned int num_lock;  /* keyboard numlock modifiers */
	struct _XContextDB *context_db; /* context database */
	Bool (**error_vec)(	/* vector for wire to error */
		Display     *	/* display */,
		XErrorEvent *	/* he */,
		xError      *	/* we */
		);
	/*
	 * Xcms information
	 */
	struct {
	   XPointer defaultCCCs;  /* pointer to an array of default XcmsCCC */
	   XPointer clientCmaps;  /* pointer to linked list of XcmsCmapRec */
	   XPointer perVisualIntensityMaps;
				  /* linked list of XcmsIntensityMap */
	} cms;
	struct _XIMFilter *im_filters;
	struct _XSQEvent *qfree; /* unallocated event queue elements */
	unsigned long next_event_serial_num; /* inserted into next queue elt */
	struct _XExten *flushes; /* Flush hooks */
	struct _XConnectionInfo *im_fd_info; /* _XRegisterInternalConnection */
	int im_fd_length;	/* number of im_fd_info */
	struct _XConnWatchInfo *conn_watchers; /* XAddConnectionWatch */
	int watcher_count;	/* number of conn_watchers */
	XPointer filedes;	/* struct pollfd cache for _XWaitForReadable */
	int (*savedsynchandler)( /* user synchandler when Xlib usurps */
		Display *	/* dpy */
		);
	XID resource_max;	/* allocator max ID */
	int xcmisc_opcode;	/* major opcode for XC-MISC */
	struct _XkbInfoRec *xkb_info; /* XKB info */
	struct _XtransConnInfo *trans_conn; /* transport connection object */
	struct _X11XCBPrivate *xcb; /* XCB glue private data */

	/* Generic event cookie handling */
	unsigned int next_cookie; /* next event cookie */
	/* vector for wire to generic event, index is (extension - 128) */
	Bool (*generic_event_vec[128])(
		Display *	/* dpy */,
		XGenericEventCookie *	/* Xlib event */,
		xEvent *	/* wire event */);
	/* vector for event copy, index is (extension - 128) */
	Bool (*generic_event_copy_vec[128])(
		Display *	/* dpy */,
		XGenericEventCookie *	/* in */,
		XGenericEventCookie *   /* out*/);
	void *cookiejar;  /* cookie events returned but not claimed */
};
```

其中第一个定义是暴露给外部应用程序使用的，第二个定义在 X11 内部使用的。可以看到第一个里面的一些 private* ，在第二个中都有了相应的定义，并且它们所占用的存储空间都是一样的，而且第一个在最后还提示了，后面的部分是 Xlib 私有的。

* **Screen**
Screen 的层次在 Display 之下，是 X server 显示管理的次级单位。一个 Screen 对应一个根窗口（root window），根窗口的大小与 Screen 相同。如果在命令行执行 "X" 的话，启动了 X server，这时在屏幕上看到一个单调的桌面，以及一个 "X" 形的鼠标，不过因为没有启动 window manager，所以什么都不能做，只能动动鼠标。这时你看到的这个单调的“桌面”正是根窗口。它的数据结构如下：

代码：

```cpp
// 在 X11/xlib.h 中的定义
typedef struct {
	XExtData *ext_data;	/* hook for extension to hang data */
	struct _XDisplay *display;/* back pointer to display structure */
	Window root;		/* Root window id. */
	int width, height;	/* width and height of screen */
	int mwidth, mheight;	/* width and height of  in millimeters */
	int ndepths;		/* number of depths possible */
	Depth *depths;		/* list of allowable depths on the screen */
	int root_depth;		/* bits per pixel */
	Visual *root_visual;	/* root visual */
	GC default_gc;		/* GC for the root root visual */
	Colormap cmap;		/* default color map */
	unsigned long white_pixel;
	unsigned long black_pixel;	/* White and Black pixel values */
	int max_maps, min_maps;	/* max and min color maps */
	int backing_store;	/* Never, WhenMapped, Always */
	Bool save_unders;
	long root_input_mask;	/* initial root input mask */
} Screen;
```

其中包括的一些看看注释也就知道是什么了。

* **GC**
Graphic Context：X11 上的绘图上下文，类似于 MiniGUI 里的 DC 概念。它的数据结构是：

代码：

```cpp
// 在 X11/xlib.h 中的定义
/*
 * Data structure for setting graphics context.
 */
typedef struct {
	int function;		/* logical operation */
	unsigned long plane_mask;/* plane mask */
	unsigned long foreground;/* foreground pixel */
	unsigned long background;/* background pixel */
	int line_width;		/* line width */
	int line_style;	 	/* LineSolid, LineOnOffDash, LineDoubleDash */
	int cap_style;	  	/* CapNotLast, CapButt,
				   CapRound, CapProjecting */
	int join_style;	 	/* JoinMiter, JoinRound, JoinBevel */
	int fill_style;	 	/* FillSolid, FillTiled,
				   FillStippled, FillOpaeueStippled */
	int fill_rule;	  	/* EvenOddRule, WindingRule */
	int arc_mode;		/* ArcChord, ArcPieSlice */
	Pixmap tile;		/* tile pixmap for tiling operations */
	Pixmap stipple;		/* stipple 1 plane pixmap for stipping */
	int ts_x_origin;	/* offset for tile or stipple operations */
	int ts_y_origin;
        Font font;	        /* default text font for text operations */
	int subwindow_mode;     /* ClipByChildren, IncludeInferiors */
	Bool graphics_exposures;/* boolean, should exposures be generated */
	int clip_x_origin;	/* origin for clipping */
	int clip_y_origin;
	Pixmap clip_mask;	/* bitmap clipping; other calls for rects */
	int dash_offset;	/* patterned/dashed line information */
	char dashes;
} XGCValues;

/*
 * Graphics context.  The contents of this structure are implementation
 * dependent.  A GC should be treated as opaque by application code.
 */

typedef struct _XGC
#ifdef XLIB_ILLEGAL_ACCESS
{
    XExtData *ext_data;	/* hook for extension to hang data */
    GContext gid;	/* protocol ID for graphics context */
    /* there is more to this structure, but it is private to Xlib */
}
#endif
*GC;

// 在 X11/xlibint.h 中的定义
struct _XGC
{
    XExtData *ext_data;	/* hook for extension to hang data */
    GContext gid;	/* protocol ID for graphics context */
    Bool rects;		/* boolean: TRUE if clipmask is list of rectangles */
    Bool dashes;	/* boolean: TRUE if dash-list is really a list */
    unsigned long dirty;/* cache dirty bits */
    XGCValues values;	/* shadow structure of values */
};
```

和 Display 的定义类似的，暴露的结构数据比较少。

* **Window**
这个就是 X11 里的窗口的概念了。不过它的定义和别的 GUI 不太一样。它在 X11/X.h 里被简单的定义成： 

代码：

```cpp
// 在 X11/X.h 中定义
typedef CARD32 XID;
typedef XID Window;

// 在 X11/Xmd.h 中定义
typedef unsigned long CARD32;
```

可以看得到这个其实是一个 32位 的 ID 号。而不像其它 GUI 中包括一些什么窗口的标题、矩形区域大小、绘图上下文等等。这些可能保存在 X 服务器那一端吧，因为 X11 是 C/S 架构的，所以通过一个唯一的 ID 号来进行传送效率会比较高吧。这里需要说明下，这个就已经是 X11 上能够绘制图像的东西了，可以说类似 MiniGUI Surface 一样的东西了，只不过你无法通过这个变量直接拿到里面的数据。要想获得一些相关的数据，好像需要通过 X11 的协议向 X 服务器发送相应的请求。

* **Pixmap**
这个是 X11 上另一个可以绘制图像的东西。它和上面 Window 的区别在于：Window 的绘制是直接在屏幕上的，而 Pixmap 的绘制是离屏的（类似 MiniGUI 的 on-screen surface 和 off-screen surface）。它在 X11/X.h 中也被简单的定义成：typedef XID Pixmap; 。同样是一个 ID 号。

* **Drawable**
这个也是一个 ID 号： typedef XID Drawable; 。上面说了 X11 中可以绘制的东西分为2种：on-screen 的 Window 和 off-sreen 的 Pixmap，这个 Drawable 就是指向其中的某一种的。也就是说一个 Drawable 不是 Window 就是 Pixmap，这里可能是为了方便某些地方的统一处理吧。

### 基本使用流程
下面说下在 X11 上创建一个窗口的基本流程：

* **XOpenDisplay**
首先需要调用 XOpenDisplay  打开显示设备。这个函数会返回一个 Display 指针。一般传入 NULL 会使用环境变量中 DISPLAY 设定的值。

* **DefaultScreen**
然后调用 DefaultScreen 得到默认的 Screen。一般是传入 XOpenDisplay 得到的 display。

* **XCreateWindow**
然后是通过 XCreateWindow 创建窗口。这个 API 需要传入大量的参数：例如 display 指针、父窗口 ID 号（第一个窗口的父窗口是 root window）、矩形区域、色深、颜色格式、窗口属性数组等等。这个具体的看附件的教程资料吧，东西太多了。有个简单的接口叫 XCreateSimpleWindow 。

* **XMapWindow**
创建了窗口后，不会马上显示，需要用这个 API 之后才会显示。

* **XCreateGC**
通过之前创建的 display 和 drawable（window）创建 GC，通过这个 GC 就可以在创建的 window 上进行绘图了。 附件里有个小例子。

### Xlib 线索
查找了下 Xlib 的 API 手册，没发现有可以直接使用的 API 能拿到像素地址的。不过想想也是，应该是没有哪个 GUI 库会直接提供 API 让应用程序来直接读写 surface 的像素地址的，因为这样对于普通的应用程序来说还是比较危险的，很容易造成显示上的问题。特别是对于 C/S 架构的 X11 来说。并且通过前面的讨论也无法直接从一些核心的图形数据结构上直接拿到（像 Window 这样的类型直接就是一个 ID 号）。API 手册上有几个 API 倒是可以把 Drawable 中的像素数据复制出来，但是是复制出来的，不是 Drawable 原始的像素数据。但是既然它能够从 Drawable 中复制出数据，那就是说明它肯定是可以访问得到 Drawable 原始的像素数据的，至少可以拿得到它的地址。所以我觉得我们可以参照下这些 API 的实现，自己改装下，应该也可以拿得到像素数据地址。其中的一个 API 是：

* **XGetImage**

代码：

```cpp
// 这个函数的流程是先通过 GetReq 填写向 X 服务器要发送的请求。
// 然后调用 _XReply 向 X 服务器发送获取给定窗口图像的请求，并且等待服务器返回数据。
// 然后当 _XReply 正确返回后，调用 _XReadPad 通过 X 服务器返回过来的偏移量计算出图像的像素地址。
// 然后根据这个像素地址调用 XCreateImage 创建一个新的 XImage ，最后返回。
// 这里返回的是复制过的，所以我们可以参考它是如果像 X11 拿像素地址的
// _XReply 和 _XReadPad 的代码就不贴了，都是在 Xlib 库的源代码里的
XImage *XGetImage (
     register Display *dpy,
     Drawable d,
     int x,
     int y,
     unsigned int width,
     unsigned int height,
     unsigned long plane_mask,
     int format)	/* either XYPixmap or ZPixmap */
{
	xGetImageReply rep;
	register xGetImageReq *req;
	char *data;
	long nbytes;
	XImage *image;
	LockDisplay(dpy);
	GetReq (GetImage, req);
	/*
	 * first set up the standard stuff in the request
	 */
	req->drawable = d;
	req->x = x;
	req->y = y;
	req->width = width;
	req->height = height;
	req->planeMask = plane_mask;
	req->format = format;

	if (_XReply (dpy, (xReply *) &rep, 0, xFalse) == 0 ||
	    rep.length == 0) {
		UnlockDisplay(dpy);
		SyncHandle();
		return (XImage *)NULL;
	}

	nbytes = (long)rep.length << 2;
	data = (char *) Xmalloc((unsigned) nbytes);
	if (! data) {
	    _XEatData(dpy, (unsigned long) nbytes);
	    UnlockDisplay(dpy);
	    SyncHandle();
	    return (XImage *) NULL;
	}
        _XReadPad (dpy, data, nbytes);
        if (format == XYPixmap)
	   image = XCreateImage(dpy, _XVIDtoVisual(dpy, rep.visual),
		  Ones (plane_mask &
			(((unsigned long)0xFFFFFFFF) >> (32 - rep.depth))),
		  format, 0, data, width, height, dpy->bitmap_pad, 0);
	else /* format == ZPixmap */
           image = XCreateImage (dpy, _XVIDtoVisual(dpy, rep.visual),
		 rep.depth, ZPixmap, 0, data, width, height,
		  _XGetScanlinePad(dpy, (int) rep.depth), 0);

	if (!image)
	    Xfree(data);
	UnlockDisplay(dpy);
	SyncHandle();
	return (image);
}
```

不过这个函数需要调用一些 Xlib 内部使用的函数。不知道能不能方便的扣出来使用。并且估计这个拿到的应该是虚拟地址，而不是物理地址。

## X server 分析
X11 很多东西都是通过向 X server 发送请求来完成的。所有其实有很多核心的东西是在 X server 这边的。包括低层的打开 framebuffer 设备等等，应该都是在 X server 这边实现的。

### X server 基本数据结构
之前在 Xlib 那边说过很多结构都是一个 ID 号，其实在 X server 这边都对应一个数据结构。

* **Screen**

代码：

```cpp
typedef struct _Screen {
    int			myNum;	/* index of this instance in Screens[] */
    ATOM		id;
    short		x, y, width, height;
    short		mmWidth, mmHeight;
    short		numDepths;
    unsigned char      	rootDepth;
    DepthPtr       	allowedDepths;
    unsigned long      	rootVisual;
    unsigned long	defColormap;
    short		minInstalledCmaps, maxInstalledCmaps;
    char                backingStoreSupport, saveUnderSupport;
    unsigned long	whitePixel, blackPixel;
    GCPtr		GCperDepth[MAXFORMATS+1];
			/* next field is a stipple to use as default in
			   a GC.  we don't build default tiles of all depths
			   because they are likely to be of a color
			   different from the default fg pixel, so
			   we don't win anything by building
			   a standard one.
			*/
    PixmapPtr		PixmapPerDepth[1];
    pointer		devPrivate;
    short       	numVisuals;
    VisualPtr		visuals;
    WindowPtr		root;
    ScreenSaverStuffRec screensaver;

    /* Random screen procedures */

    CloseScreenProcPtr		CloseScreen;
    QueryBestSizeProcPtr	QueryBestSize;
    SaveScreenProcPtr		SaveScreen;
    GetImageProcPtr		GetImage;
    GetSpansProcPtr		GetSpans;
    SourceValidateProcPtr	SourceValidate;

    /* Window Procedures */

    CreateWindowProcPtr		CreateWindow;
    DestroyWindowProcPtr	DestroyWindow;
    PositionWindowProcPtr	PositionWindow;
    ChangeWindowAttributesProcPtr ChangeWindowAttributes;
    RealizeWindowProcPtr	RealizeWindow;
    UnrealizeWindowProcPtr	UnrealizeWindow;
    ValidateTreeProcPtr		ValidateTree;
    PostValidateTreeProcPtr	PostValidateTree;
    WindowExposuresProcPtr	WindowExposures;
    CopyWindowProcPtr		CopyWindow;
    ClearToBackgroundProcPtr	ClearToBackground;
    ClipNotifyProcPtr		ClipNotify;
    RestackWindowProcPtr	RestackWindow;

    /* Pixmap procedures */

    CreatePixmapProcPtr		CreatePixmap;
    DestroyPixmapProcPtr	DestroyPixmap;

    /* Backing store procedures */

    SaveDoomedAreasProcPtr	SaveDoomedAreas;
    RestoreAreasProcPtr		RestoreAreas;
    ExposeCopyProcPtr		ExposeCopy;
    TranslateBackingStoreProcPtr TranslateBackingStore;
    ClearBackingStoreProcPtr	ClearBackingStore;
    DrawGuaranteeProcPtr	DrawGuarantee;
    /*
     * A read/write copy of the lower level backing store vector is needed now
     * that the functions can be wrapped.
     */
    BSFuncRec			BackingStoreFuncs;
    
    /* Font procedures */

    RealizeFontProcPtr		RealizeFont;
    UnrealizeFontProcPtr	UnrealizeFont;

    /* Cursor Procedures */

    ConstrainCursorProcPtr	ConstrainCursor;
    CursorLimitsProcPtr		CursorLimits;
    DisplayCursorProcPtr	DisplayCursor;
    RealizeCursorProcPtr	RealizeCursor;
    UnrealizeCursorProcPtr	UnrealizeCursor;
    RecolorCursorProcPtr	RecolorCursor;
    SetCursorPositionProcPtr	SetCursorPosition;

    /* GC procedures */

    CreateGCProcPtr		CreateGC;

    /* Colormap procedures */

    CreateColormapProcPtr	CreateColormap;
    DestroyColormapProcPtr	DestroyColormap;
    InstallColormapProcPtr	InstallColormap;
    UninstallColormapProcPtr	UninstallColormap;
    ListInstalledColormapsProcPtr ListInstalledColormaps;
    StoreColorsProcPtr		StoreColors;
    ResolveColorProcPtr		ResolveColor;

    /* Region procedures */

    BitmapToRegionProcPtr	BitmapToRegion;
    SendGraphicsExposeProcPtr	SendGraphicsExpose;

    /* os layer procedures */

    ScreenBlockHandlerProcPtr	BlockHandler;
    ScreenWakeupHandlerProcPtr	WakeupHandler;

    pointer blockData;
    pointer wakeupData;

    /* anybody can get a piece of this array */
    PrivateRec	*devPrivates;

    CreateScreenResourcesProcPtr CreateScreenResources;
    ModifyPixmapHeaderProcPtr	ModifyPixmapHeader;

    GetWindowPixmapProcPtr	GetWindowPixmap;
    SetWindowPixmapProcPtr	SetWindowPixmap;
    GetScreenPixmapProcPtr	GetScreenPixmap;
    SetScreenPixmapProcPtr	SetScreenPixmap;

    PixmapPtr pScratchPixmap;		/* scratch pixmap "pool" */

    unsigned int		totalPixmapSize;

    MarkWindowProcPtr		MarkWindow;
    MarkOverlappedWindowsProcPtr MarkOverlappedWindows;
    ChangeSaveUnderProcPtr	ChangeSaveUnder;
    PostChangeSaveUnderProcPtr	PostChangeSaveUnder;
    ConfigNotifyProcPtr		ConfigNotify;
    MoveWindowProcPtr		MoveWindow;
    ResizeWindowProcPtr		ResizeWindow;
    GetLayerWindowProcPtr	GetLayerWindow;
    HandleExposuresProcPtr	HandleExposures;
    ReparentWindowProcPtr	ReparentWindow;

    SetShapeProcPtr		SetShape;

    ChangeBorderWidthProcPtr	ChangeBorderWidth;
    MarkUnrealizedWindowProcPtr	MarkUnrealizedWindow;

    /* Device cursor procedures */
    DeviceCursorInitializeProcPtr DeviceCursorInitialize;
    DeviceCursorCleanupProcPtr    DeviceCursorCleanup;
} ScreenRec;
```

这个 Screen 是 X server 这边使用的，和 Xlib 那边的不一样。其中有很多回调函数指针，从名字可以看到很多都是一些核心的图形功能的，例如创建窗口、创建GC 等等。这些回调函数在初始化时被设置成和 framebuffer 操作相关的函数。这个在 xserver 的代码目录的 fb/fbscreen.c 的 fbSetupScreen 函数里可以看得到，而这个函数会在初始化的时候被调用，具体的大家可以自己去跟下代码。

* **GC**

代码：

```cpp
/* there is padding in the bit fields because the Sun compiler doesn't
 * force alignment to 32-bit boundaries.  losers.
 */
typedef struct _GC {
    ScreenPtr		pScreen;
    unsigned char	depth;
    unsigned char	alu;
    unsigned short	lineWidth;
    unsigned short	dashOffset;
    unsigned short	numInDashList;
    unsigned char	*dash;
    unsigned int	lineStyle : 2;
    unsigned int	capStyle : 2;
    unsigned int	joinStyle : 2;
    unsigned int	fillStyle : 2;
    unsigned int	fillRule : 1;
    unsigned int 	arcMode : 1;
    unsigned int	subWindowMode : 1;
    unsigned int	graphicsExposures : 1;
    unsigned int	clientClipType : 2; /* CT_<kind> */
    unsigned int	miTranslate:1; /* should mi things translate? */
    unsigned int	tileIsPixel:1; /* tile is solid pixel */
    unsigned int	fExpose:1;     /* Call exposure handling */
    unsigned int	freeCompClip:1;  /* Free composite clip */
    unsigned int	scratch_inuse:1; /* is this GC in a pool for reuse? */
    unsigned int	unused:13; /* see comment above */
    unsigned long	planemask;
    unsigned long	fgPixel;
    unsigned long	bgPixel;
    /*
     * alas -- both tile and stipple must be here as they
     * are independently specifiable
     */
    PixUnion		tile;
    PixmapPtr		stipple;
    DDXPointRec		patOrg;		/* origin for (tile, stipple) */
    struct _Font	*font;
    DDXPointRec		clipOrg;
    DDXPointRec		lastWinOrg;	/* position of window last validated */
    pointer		clientClip;
    unsigned long	stateChanges;	/* masked with GC_<kind> */
    unsigned long       serialNumber;
    GCFuncs		*funcs;
    GCOps		*ops;
    PrivateRec		*devPrivates;
    /*
     * The following were moved here from private storage to allow device-
     * independent access to them from screen wrappers.
     * --- 1997.11.03  Marc Aurele La France (tsi@xfree86.org)
     */
    PixmapPtr		pRotatedPixmap; /* tile/stipple rotated for alignment */
    RegionPtr		pCompositeClip;
    /* fExpose & freeCompClip defined above */
} GC;
```

同样是 X server 这边使用的 GC。其中 ops 这个结构里是一大堆类似 GDI 的回调函数指针（），也是在初始化的时候被设置的。具体的还没怎么仔细看。

* **Window**

代码：

```cpp
typedef struct _Window {
    DrawableRec	drawable;
    PrivateRec		*devPrivates;
    WindowPtr		parent;		/* ancestor chain */
    WindowPtr		nextSib;	/* next lower sibling */
    WindowPtr		prevSib;	/* next higher sibling */
    WindowPtr		firstChild;	/* top-most child */
    WindowPtr		lastChild;	/* bottom-most child */
    RegionRec		clipList;	/* clipping rectangle for output */
    RegionRec		borderClip;	/* NotClippedByChildren + border */
    union _Validate	*valdata;
    RegionRec		winSize;
    RegionRec		borderSize;
    DDXPointRec		origin;		/* position relative to parent */
    unsigned short	borderWidth;
    unsigned short	deliverableEvents; /* all masks from all clients */
    Mask		eventMask;      /* mask from the creating client */
    PixUnion		background;
    PixUnion		border;
    pointer		backStorage;	/* null when BS disabled */
    WindowOptPtr	optional;
    unsigned		backgroundState:2; /* None, Relative, Pixel, Pixmap */
    unsigned		borderIsPixel:1;
    unsigned		cursorIsNone:1;	/* else real cursor (might inherit) */
    unsigned		backingStore:2;
    unsigned		saveUnder:1;
    unsigned		DIXsaveUnder:1;
    unsigned		bitGravity:4;
    unsigned		winGravity:4;
    unsigned		overrideRedirect:1;
    unsigned		visibility:2;
    unsigned		mapped:1;
    unsigned		realized:1;	/* ancestors are all mapped */
    unsigned		viewable:1;	/* realized && InputOutput */
    unsigned		dontPropagate:3;/* index into DontPropagateMasks */
    unsigned		forcedBS:1;	/* system-supplied backingStore */
    unsigned		redirectDraw:2;	/* COMPOSITE rendering redirect */
    unsigned		forcedBG:1;	/* must have an opaque background */
#ifdef ROOTLESS
    unsigned		rootlessUnhittable:1;	/* doesn't hit-test */
#endif
} WindowRec;
```

这个就有点类似 MiniGUI 里的 MainWnd 结构了，有剪切域、父子指针、私有结构、DC（Drawable）等这些东西。

* **Pixmap**

代码：

```cpp
typedef struct _Pixmap {
    DrawableRec		drawable;
    PrivateRec		*devPrivates;
    int			refcnt;
    int			devKind; /* This is the pitch of the pixmap, typically width*bpp/8. */
    DevUnion		devPrivate; /* When !NULL, devPrivate.ptr points to the raw pixel data. */
#ifdef COMPOSITE
    short		screen_x;
    short		screen_y;
#endif
    unsigned		usage_hint; /* see CREATE_PIXMAP_USAGE_* */
} PixmapRec;
```

其实从 X server 的代码里看，X 中的 surface 都应该是 Pixmap 类型来的。就算是之前说的 Window 类型，它也会从它的结构里取出 Pixmap 变量来进行操作。看代码，估计 pixels 数据就在 devPrivates 这个指针指向的结构里面。不过这个类型的定义我还没看到在哪，不知道被藏到哪个地方去了。

* **Drawable**

代码：

```cpp
typedef struct _Drawable {
    unsigned char	type;	/* DRAWABLE_<type> */
    unsigned char	class;	/* specific to type */
    unsigned char	depth;
    unsigned char	bitsPerPixel;
    XID			id;	/* resource id */
    short		x;	/* window: screen absolute, pixmap: 0 */
    short		y;	/* window: screen absolute, pixmap: 0 */
    unsigned short	width;
    unsigned short	height;
    ScreenPtr		pScreen;
    unsigned long	serialNumber;
} DrawableRec;
```

## GLX 分析
GLX 是衔接 OpenGL 和 X11 的桥梁，OpenGL 是通过 GLX 来获取渲染的 Buffer 地址的。也许我们可以通过 GLX 来获取像素地址。

### GLX 基本数据结构
* **GLXContext**
这个是 GLX 的绘图上下文，类似 X11 的 GC，MiniGUI 的 DC。在 Mesa-7.9 里的定义是：

代码：

```cpp
typedef struct __GLXcontextRec *GLXContext;
typedef XID GLXPixmap;
typedef XID GLXDrawable;
/* GLX 1.3 and later */
typedef struct __GLXFBConfigRec *GLXFBConfig;
typedef XID GLXFBConfigID;
typedef XID GLXContextID;
typedef XID GLXWindow;
typedef XID GLXPbuffer;

/* The GLX API dispatcher (i.e. this code) is being built into stand-alone
 * Mesa.  We don't know anything about XFree86 or real GLX so we define a
 * minimal __GLXContextRec here so some of the functions in this file can
 * work properly.
 */
typedef struct __GLXcontextRec {
   Display *currentDpy;
   GLboolean isDirect;
   GLXDrawable currentDrawable;
   GLXDrawable currentReadable;
   XID xid;
} __GLXcontext;
```

从注视可以看得到，这个其实并不是 HP mini 上 Meego 使用的 GLX 的定义，因为这个应该是和具体的硬件驱动有关的（drm-intel ？？），但是从上面也看得到这个 GLXContext 的最小结构是什么。包括有 X11 的显示设备指针 Display*、是否是直接渲染的标志（DRI）、当前的 Drawable （这个应该就是 surface）、最后一个是一个 ID 号。

* **GLXDrawable**
这个的定义也在上面，基本上就是等同于 X11 的 Drawable，并且在 OpenGL 的官方文档中也是这么直接了当的说的（官方的 glx 1.3 说明文档在附件中）。它也是可以指向 GLXWindow 或是 GLXPixmap。

* **GLXWindow**
上面已经说的很清楚了，等于 X11 的 Window。

* **GLXPixmap**
上面也说清楚了，等于 X11 的 Pixmap。

### GLX 线索
从这里看来从 GLX 的核心数据结构里也无法直接拿到像素地址。最后还是走到 X11 那里去了。不过也许可以通过 GLX 访问 DRI 的一些接口，从而拿到物理的显存地址。不过这个我还没研究过。

## SDL 分析
SDL 是之前用于 vfb 的一个图形库，它比较简单，接近底层，应该是会有可能从它这里拿到像素地址，并且它已经把 OpenGL 封装好了。

### SDL 基本数据结构
* **SDL_Surface**

代码：

```cpp
typedef struct SDL_Surface {
	Uint32 flags;				/**< Read-only */
	SDL_PixelFormat *format;		/**< Read-only */
	int w, h;				/**< Read-only */
	Uint16 pitch;				/**< Read-only */
	void *pixels;				/**< Read-write */
	int offset;				/**< Private */

	/** Hardware-specific surface info */
	struct private_hwdata *hwdata;

	/** clipping information */
	SDL_Rect clip_rect;			/**< Read-only */
	Uint32 unused1;				/**< for binary compatibility */

	/** Allow recursive locks */
	Uint32 locked;				/**< Private */

	/** info for fast blit mapping to other surfaces */
	struct SDL_BlitMap *map;		/**< Private */

	/** format version, bumped at every change to invalidate blit maps */
	unsigned int format_version;		/**< Private */

	/** Reference count -- used when freeing surface */
	int refcount;				/**< Read-mostly */
} SDL_Surface;
```

这个是很直接的一个数据的结构，和 MiniGUI 的 Surface 很像（好像 MiniGUI 也借鉴了 SDL 的一些东西）。其中的 pixels 确实就是像素地址。但是不同的 video （MiniGUI 的 gal 概念），这个值的含义确实不一样的。在 X11 的 video 中这个地址不是直接的窗口的像素地址。而是一个 SDL 自己创建的基于 Display 的一个共享 XImage 对象的内存的地址。SDL 自己创建这个 XImage，然后 SDL 所有的绘制接口都在这个 XImage 上进行，最后在 SDL_UpdateRect 的时候再讲这个共享的 XImage 送到 X11 上，然后屏幕上才会显示更新的。我曾经试过了直接写这个地址，屏幕是不会变化的，必须要调用 SDL_UpdateRect 后，屏幕才会有变化。而且 SDL 的代码里就是这样写的，可以自己去看：在 /src/video/x11/SDL_x11image.c 里。而且这还是要 SDL 能够创建硬件的 surface 才行，要是颜色格式不对，导致 SDL 使用软件 surface 的话，那就完全软件来实现这些画图工作了。并且还有一点，这还不能开启 SDL 的 OpenGL 窗口，如果开启了 OpenGL 窗口的话，这个 surface 就基本没啥用了（这个时候 surface 里的一些相关数据基本上全是0），都必须要用 OpenGL 的接口才能画得出东西来了。如果你想继续画 2D 的东西的话，按照 SDL 官网的建议，要自己创建一个软件的 SDL_Surface ，画好后，把它转化成 OpenGL texture，然后贴到 OpenGL 的模型上。X11 video 的私有结构数据里包括了一些 X11 显示相关的变量：Display、Window 等，代码就不贴了，在 src/video/SDL_x11video.h 里。X11 video 的 gl 私有数据则包括了 GLX 绘图上下文：GLXContext 等，同样不贴代码了，在 src/video/SDL_x11gl_c.h 里。

### SDL 线索
从上面可以看得出 2D 的 SDL 的话，要获取窗口的像素地址，还是需要走 X11 的路线。而 OpenGL 的 SDL，则需要走 GLX 的路线。看样子从这里得不到什么好处呢。

## Mutter（Clutter）分析
做为窗口管理系统，它会负责处理当多个窗口重叠在一起时如果混合显示。现在某些桌面开启了某些特效之后，一些窗口会透过底下的窗口显示。因为有些功能，Mutter 这也许可以拿得到窗口的 surface 像素地址。

### Clutter 绘制流程
和上面的分析不一样，先介绍下 Mutter（Clutter）的绘制流程（我是跟踪 Mutter 显示一个窗口的流程的）。现在是在 Mutter 这边：

1. 在 Mutter 中显示窗口的 API 是 meta_compositor_show_window ，这个函数里会调用 meta_compositor_show_window 。
2. meta_compositor_show_window 最后回调用 Clutter 的 clutter_actor_show_all 。

现在是到 Clutter 这边来了，先说 Clutter 中的一些基本概念。在 Clutter 中有一个叫 Stage 的东西，这个类似 Window 的概念；然后还有一个叫 Actor 的东西，这个类似于 Window 里面的控件、widget 之类的。所有的 Actor 都在 Stage 中，然后各自负责自己的东西（绘制、移动等等），然后在 Stage 上展示出来。从概念上来说将 GUI 类比成演员在舞台上表演、然后开发人员是导演，指挥这些演员演出，感觉还是挺形象的。Clutter 也是采用 backend 的设计来实现跨平台的，在 linux 上的 backend 是 GLX 和 X11 （代码里是只有 GLX 这个 backend 的，但是 GLX 继承自 X11）。

1. clutter_actor_show_all 会调用 ClutterActorClass 的 show_all 函数指针，这个在 ClutterActorClass 类初始化的时候（之前说了 Clutter 是采用 gobject 的东西来设计的）是指向 clutter_actor_show 的。
2. clutter_actor_show 会调用 clutter_actor_queue_redraw 。
3. clutter_actor_queue_redraw 会调用 clutter_actor_queue_redraw_with_origin。
4. clutter_queue_redraw_with_origin 里会发射一个 gtk 的信号，看注释说默认的处理函数是 clutter_redraw。
5. clutter_redraw 会将一个重绘的标志置为 TRUE，然后开启一个时钟（定时器？？）。
6. 在时钟的回调函数里会调用 _clutter_stage_do_update。
7. _clutter_stage_do_update 会调用 _clutter_do_redraw。
8. _clutter_do_redraw 会调用 _clutter_bakcend_redraw。
9. _clutter_backend_redraw 会调用当前 backend 的 redraw，这里是 clutter_backend_glx_redraw 。
10. clutter_backend_glx_redraw 会调用 clutter_stage_glx_redraw 。
11. 在 clutter_stage_glx_redraw 会调用 clutter_actor_paint 让 actor 重画（这个函数好像是让 actor 真正的渲染到 display 上，这个 display 应该是 X11 里的那个 display 结构）。然后会根据当前是否支持 copy_sub_buffer 这个操作（这个是根据是否能取得到 glXCopySubBufferMESA 这个函数的实现来决定的），如果支持则调用 copy_sub_buffer，如果不支持则调用 glXSwapBuffers。最后的这个操作感觉像是将渲染好的 buffer 输出到最后的屏幕上显示。Clutter 也不是直接操作屏幕的么？？这个目前还没搞清楚。

到这里绘制流程就走得差不多了，这里可以看得到最后还是通过各个 backend 来进行绘制的，和 SDL 差不多。GLX 的 backend 会调用 Cogl 的 API 进行一些渲染（好像不光是 3D 的）。这里 GLX backend 感觉是带 OpenGL 支持的，X11 则是普通的 2D 的。因为从代码里可以看到 Clutter 专门区分了2个窗口：一个是 glxwin，一个是 xwin。

### Clutter 基本数据结构
我们来看看 Clutter backend 的基本数据结构吧：

* **ClutterBackendGLX、ClutterBackendX11**

代码：

```cpp
struct _ClutterBackendGLX
{
  ClutterBackendX11 parent_instance;

  int                    error_base;
  int                    event_base;

  /* Single context for all wins */
  gboolean               found_fbconfig;
  GLXFBConfig            fbconfig;
  GLXContext             gl_context;
  Window                 dummy_xwin;
  GLXWindow              dummy_glxwin;

  /* Vblank stuff */
  GetVideoSyncProc       get_video_sync;
  WaitVideoSyncProc      wait_video_sync;
  SwapIntervalProc       swap_interval;
  gint                   dri_fd;
  ClutterGLXVBlankType   vblank_type;

  CopySubBufferProc      copy_sub_buffer;

  /* props */
  Atom atom_WM_STATE;
  Atom atom_WM_STATE_FULLSCREEN;
};

struct _ClutterBackendX11
{
  ClutterBackend parent_instance;

  Display *xdpy;
  Window   xwin_root;
  Screen  *xscreen;
  int      xscreen_num;
  gchar   *display_name;

  /* event source */
  GSource *event_source;
  GSList  *event_filters;

  /* props */
  Atom atom_NET_WM_PID;
  Atom atom_NET_WM_PING;
  Atom atom_NET_WM_STATE;
  Atom atom_NET_WM_STATE_FULLSCREEN;
  Atom atom_NET_WM_USER_TIME;
  Atom atom_WM_PROTOCOLS;
  Atom atom_WM_DELETE_WINDOW;
  Atom atom_XEMBED;
  Atom atom_XEMBED_INFO;
  Atom atom_NET_WM_NAME;
  Atom atom_UTF8_STRING;

  int xi_event_base;
  int event_types[CLUTTER_X11_XINPUT_LAST_EVENT];
  gboolean have_xinput;

  Time last_event_time;

  ClutterDeviceManager *device_manager;
};
```

X11 里的是 X11 显示的一些基本变量：Display、Window、Screen 等；而 GLX 的则是 GLX 里的基本东西：GLXContext、GLXWindow 等。

* **ClutterStageGLX、ClutterStageX11**

代码：

```cpp
struct _ClutterStageGLX
{
  ClutterStageX11 parent_instance;

  int pending_swaps;

  GLXPixmap glxpixmap;
  GLXWindow glxwin;

  gboolean initialized_redraw_clip;
  ClutterGeometry bounding_redraw_clip;
};

struct _ClutterStageX11
{
  ClutterGroup parent_instance;

  guint        is_foreign_xwin      : 1;
  guint        fullscreening        : 1;
  guint        is_cursor_visible    : 1;
  guint        viewport_initialized : 1;

  Window       xwin;
  gint         xwin_width;
  gint         xwin_height; /* FIXME target_width / height */
  gchar       *title;

  ClutterStageState  state;

  ClutterStageX11State wm_state;

  int event_types[CLUTTER_X11_XINPUT_LAST_EVENT];
  GList *devices;

  ClutterStage *wrapper;
};
```

这里也是和上面的差不多的，至于是不是和上面的是重复的，我还没仔细看。

### Clutter 线索
这里也和 SDL 差不多了，最后都是 X11 底层的东西来完成绘制的。不过这里用的是 GLX 的接口（ GLX 也算是用 X11 的了吧）。不过根据这里的线索到 Mesa 的 GLX 去看看，好像能看到像 glXCopySubBufferMESA 这个函数的实现，应该是 src/glx/glxcmds.c 里的 __glXCopySubBufferMESA 吧，这个函数好像类似硬件的 blit 的感觉。这个函数里如果是走 DRI 的话，就直接调用 DRI 的回调函数；如果不是 DRI 的话，就只能通过向 X 服务器发请求的方式来实现。这个也许就是之前说的 GLX 中的线索吧。

## Cogl 分析
这个目前还没什么仔细看。

## 总结
从上面的讨论看到，如果抛开 OpenGL 的支持不管的话，就光要 Android 能直接画东西到 X11 的屏幕上，估计还是得要通过 Xlib 的接口，自己向 X 服务器发送一些申请，然后计算出像素地址。

## 附件
[GLX 1.3 官方说明文档](http://s.yunio.com/7xUt2x "GLX 1.3 官方说明文档")
[Xlib编程指南](http://s.yunio.com/AYxGTn "Xlib编程指南")
[x11_sample](http://s.yunio.com/1LPFWz "x11_sample")

