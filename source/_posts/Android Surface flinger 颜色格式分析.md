title: Android Surface flinger 颜色格式分析
date: 2015-01-28 19:51:16
updated: 2015-01-28 19:51:16
categories: [Android Framework]
tags: [android]
---

其实是前段时间移植 2.3 的 blur 效果到 4.2 上，然后记录下一些需要注意点的地方。改的地方是 surfacefliner（代码在 framework/native/services/surfaceflinger 下面，2.3 的是在 framework/base/services/surfaceflinger 下）。这里主要说下颜色格式的问题（2.3 的 blur 效果是拿软件做的）。

## 大小端问题
这个过了一段时间之后老是忘记，这里要记录一下：大、小端可以理解为，高地址存储的是高位数据还是低位数据。大端存储是高地址存储是高位数据，小端存储是高地址存储的是低位数据（还有一个更取巧的记法：大端存储不要转化，和内存顺序是一致的，小端存储正好反过来）。图文讲解可以看 wiki ，很详细： [Endianness](http://en.wikipedia.org/wiki/Big-endian "Endianness") 

## 颜色格式
2.3 的 blur 效果，是通过从 GL 的 buffer 中读出 pixel 数据（glReadPixels，相当于截图），然后对这些数据进行软件的模糊算法（类高斯模糊，速度比较快），然后再把模糊后的 pixel 数据生成 texure，再贴回到 GL 上去。

所以这里就涉及到 surface flinger 的 pixel format 以及 GL 中的 pixel format。这里先说下 GL 中的 pixel format。glReadPixels 表示格式的有2个参数：一个是 format，一个是 type。这2个参数的类型可以去查红宝书的第8章， android 的代码里用的 format 和 type 就2种：

```cpp
    // surfaceflinger/LayerBlur.cpp --- onDraw()
    if (mTextureName == -1U) {
        // create the texture name the first time
        // can't do that in the ctor, because it runs in another thread.
        glGenTextures(1, &mTextureName);
        glGetIntegerv(GL_IMPLEMENTATION_COLOR_READ_FORMAT_OES, &mReadFormat);
        glGetIntegerv(GL_IMPLEMENTATION_COLOR_READ_TYPE_OES, &mReadType);
        if (mReadFormat != GL_RGB || mReadType != GL_UNSIGNED_SHORT_5_6_5) {
            mReadFormat = GL_RGBA;
            mReadType = GL_UNSIGNED_BYTE;
            mBlurFormat = GGL_PIXEL_FORMAT_RGBX_8888;
        }   
    }
```

代码里面去取 OES 所使用的 format 和 type，如果 format 不是 GL_RGB 或者 type 不是 `GL_UNSIGNED_SHORT_5_6_5` 就强制设置为 `GL_RGBA` 和 `GL_UNSIGNED_BYTE` ，顺带可以看到把 surface 的颜色格式设置为 `GGL_PIXEL_FORMAT_RGBX_8888` 了（默认是 `GGL_PIXEL_FORMAT_RGB_565`）。

GL format `GL_RGB` 就是 RGB 3通道颜色格式， `GL_RGBA` 就是 RGBA 4通道颜色格式。而 type 就是存储格式， `GL_UNSIGNED_SHORT_5_6_5` 是打包的存储方式（GL 中这种 `5_6_5`, `4_4_4_4` 之类的都是打包的存储方式）。所谓打包的存储方式，就是把前面的 RGB 3通道数据打包到 16 位的无符号整数中。特别注意一点，GL 的打包数据的存储格式总是将颜色分量的高位打包到高位，和 host 的存储方式无关。就是说 `GL_UNSIGNED_SHORT_5_6_5` with `GL_RGB` 就是 R（16，11）--G（11，5）--B（5，0） 的存储顺序。但是不打包的存储格式（例如 `GL_UNSIGNED_BYTE`），就是和存储格式相关的。看看代码的转化：

```cpp
// surfaceflinger/BlurFliter.cpp 2.3 的 blur
#if BYTE_ORDER == LITTLE_ENDIAN
inline uint32_t BLUR_RGBA_TO_HOST(uint32_t v) {
    return v;
}
inline uint32_t BLUR_HOST_TO_RGBA(uint32_t v) {
    return v;
}
#else
inline uint32_t BLUR_RGBA_TO_HOST(uint32_t v) {
    return (v<<24) | (v>>24) | ((v<<8)&0xff0000) | ((v>>8)&0xff00);
}
inline uint32_t BLUR_HOST_TO_RGBA(uint32_t v) {
    return (v<<24) | (v>>24) | ((v<<8)&0xff0000) | ((v>>8)&0xff00);
}
#endif

struct BlurColor888X
{
    typedef uint32_t type;
    int r, g, b;
    inline BlurColor888X() { }
    inline BlurColor888X(uint32_t v) {
        v = BLUR_RGBA_TO_HOST(v);
        r = v & 0xFF;
        g = (v >>  8) & 0xFF;
        b = (v >> 16) & 0xFF;
    }
    inline void clear() { r=g=b=0; }
    inline uint32_t to(int shift, int last, int dither) const {
        int R = r;
        int G = g;
        int B = b;
        if  (UNLIKELY(last)) {
            if (FACTOR>0) {
                int L = (R+G+G+B)>>2;
                R += ((L - R) * FACTOR) >> 8;
                G += ((L - G) * FACTOR) >> 8;
                B += ((L - B) * FACTOR) >> 8;
            }
        }
        R >>= shift;
        G >>= shift;
        B >>= shift;
        return BLUR_HOST_TO_RGBA((0xFF<<24) | (B<<16) | (G<<8) | R);
    }
    inline BlurColor888X& operator += (const BlurColor888X& rhs) {
        r += rhs.r;
        g += rhs.g;
        b += rhs.b;
        return *this;
    }
    inline BlurColor888X& operator -= (const BlurColor888X& rhs) {
        r -= rhs.r;
        g -= rhs.g;
        b -= rhs.b;
        return *this;
    }
};

struct BlurColor565
{
    typedef uint16_t type;
    int r, g, b;
    inline BlurColor565() { }
    inline BlurColor565(uint16_t v) {
        r = v >> 11;
        g = (v >> 5) & 0x3E;
        b = v & 0x1F;
    }
    inline void clear() { r=g=b=0; }
    inline uint16_t to(int shift, int last, int dither) const {
        int R = r;
        int G = g;
        int B = b;
        if  (UNLIKELY(last)) {
            if (FACTOR>0) {
                int L = (R+G+B)>>1;
                R += (((L>>1) - R) * FACTOR) >> 8;
                G += (((L   ) - G) * FACTOR) >> 8;
                B += (((L>>1) - B) * FACTOR) >> 8;
            }
            R += (dither << shift) >> BLUR_DITHER_BITS;
            G += (dither << shift) >> BLUR_DITHER_BITS;
            B += (dither << shift) >> BLUR_DITHER_BITS;
        }
        R >>= shift;
        G >>= shift;
        B >>= shift;
        return (R<<11) | (G<<5) | B;
    }
    inline BlurColor565& operator += (const BlurColor565& rhs) {
        r += rhs.r;
        g += rhs.g;
        b += rhs.b;
        return *this;
    }
    inline BlurColor565& operator -= (const BlurColor565& rhs) {
        r -= rhs.r;
        g -= rhs.g;
        b -= rhs.b;
        return *this;
    }
};
```

可以看的 android 只会转化 `GGL_PIXEL_FORMAT_RGBX_8888` 格式的， `GGL_PIXEL_FORMAT_RGB_565` 的不需要转化（因为前面说了 GL 打包的格式是固定是小端存储的）。

然后再看下 android surface 的颜色格式定义。surface 用的颜色格式定义在 pixelflinger 中。这个 pixelflinger 是 android 自带的 GL 软件实现，现在一般机器上的都不用这个了，但是颜色格式定义还是用这个的（代码在 system/core/libpixelflinger/）。

```cpp
// system/core/include/pixelflinger/format.h
typedef struct {
#ifdef __cplusplus
    enum {
        ALPHA   = GGL_INDEX_ALPHA,
        RED     = GGL_INDEX_RED,
        GREEN   = GGL_INDEX_GREEN,
        BLUE    = GGL_INDEX_BLUE,
        STENCIL = GGL_INDEX_STENCIL,
        DEPTH   = GGL_INDEX_DEPTH,
        LUMA    = GGL_INDEX_Y,
        CHROMAB = GGL_INDEX_CB,
        CHROMAR = GGL_INDEX_CR,
    };
    inline uint32_t mask(int i) const {
            return ((1<<(c[i].h-c[i].l))-1)<<c[i].l;
    }
    inline uint32_t bits(int i) const {
            return c[i].h - c[i].l;
    }
#endif
    uint8_t     size;   // bytes per pixel
    uint8_t     bitsPerPixel;
    union {
        struct {
            uint8_t     ah;     // alpha high bit position + 1
            uint8_t     al;     // alpha low bit position
            uint8_t     rh;     // red high bit position + 1
            uint8_t     rl;     // red low bit position
            uint8_t     gh;     // green high bit position + 1
            uint8_t     gl;     // green low bit position
            uint8_t     bh;     // blue high bit position + 1
            uint8_t     bl;     // blue low bit position
        };  
        struct {
            uint8_t h;
            uint8_t l;
        } __attribute__((__packed__)) c[4];    
    } __attribute__((__packed__));
    uint16_t    components; // GGLFormatComponents
} GGLFormat;

// system/core/libpixelflinger/format.cpp
static GGLFormat const gPixelFormatInfos[] =
{   //          Alpha    Red     Green   Blue
    // 下面 markdwon 的渲染有点问题，里面的代码 )) 应该是 }} 的
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  4, 32, {{32,24,   8, 0,  16, 8,  24,16 )), GGL_RGBA },   // PIXEL_FORMAT_RGBA_8888
    {  4, 24, {{ 0, 0,   8, 0,  16, 8,  24,16 )), GGL_RGB  },   // PIXEL_FORMAT_RGBX_8888
    {  3, 24, {{ 0, 0,   8, 0,  16, 8,  24,16 )), GGL_RGB  },   // PIXEL_FORMAT_RGB_888
    {  2, 16, {{ 0, 0,  16,11,  11, 5,   5, 0 )), GGL_RGB  },   // PIXEL_FORMAT_RGB_565
    {  4, 32, {{32,24,  24,16,  16, 8,   8, 0 )), GGL_RGBA },   // PIXEL_FORMAT_BGRA_8888
    {  2, 16, {{ 1, 0,  16,11,  11, 6,   6, 1 )), GGL_RGBA },   // PIXEL_FORMAT_RGBA_5551
    {  2, 16, {{ 4, 0,  16,12,  12, 8,   8, 4 )), GGL_RGBA },   // PIXEL_FORMAT_RGBA_4444
    {  1,  8, {{ 8, 0,   0, 0,   0, 0,   0, 0 )), GGL_ALPHA},   // PIXEL_FORMAT_A8
    {  1,  8, {{ 0, 0,   8, 0,   8, 0,   8, 0 )), GGL_LUMINANCE},//PIXEL_FORMAT_L8
    {  2, 16, {{16, 8,   8, 0,   8, 0,   8, 0 )), GGL_LUMINANCE_ALPHA},// PIXEL_FORMAT_LA_88
    {  1,  8, {{ 0, 0,   8, 5,   5, 2,   2, 0 )), GGL_RGB  },   // PIXEL_FORMAT_RGB_332
        
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE

    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
            
    {  2, 16, {{  0, 0, 16, 0,   0, 0,   0, 0 )), GGL_DEPTH_COMPONENT},
    {  1,  8, {{  8, 0,  0, 0,   0, 0,   0, 0 )), GGL_STENCIL_INDEX  },
    {  4, 24, {{  0, 0, 24, 0,   0, 0,   0, 0 )), GGL_DEPTH_COMPONENT},
    {  4,  8, {{ 32,24,  0, 0,   0, 0,   0, 0 )), GGL_STENCIL_INDEX  },

    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE

    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
    {  0,  0, {{ 0, 0,   0, 0,   0, 0,   0, 0 )),        0 },   // PIXEL_FORMAT_NONE
};
```

从上面的定义可以看得到 `GGL_PIXEL_FORMAT_RGBX_8888` 是 ABGR 的小端存储格式，所以上面如果 host 是小端格式，就不用转化，是大端的话就需要转化一下。这里的 RGBA 定义感觉挺奇怪的，和 java 层的 Bitmap ARGB 的定义顺序不一样，不知道为什么要这么定义。

`GGL_PIXEL_FORMAT_RGB_565` 正好和 `GL_UNSIGNED_SHORT_5_6_5` with `GL_RGB` 存储顺序是一样的，所以 `5_6_5` 的颜色格式也不需要转化。

颜色格式搞清楚了，软件算法用现成的就好啦。android 2.3 的 blur 效果不算很好，但是速度很快，网上有很多软件的 blur 算法，效果比 2.3 的好，但是速度慢不少。高斯模糊好像就是对这些像素点搞个啥加权求均值，具体的以后再研究了，这次只是把颜色格式搞清楚。

哎，这样吐槽下，以前移植、改的 blur 软件算法的 32位格式 `RGBX_8888` 格式是错的，只不过一般机器上的 OES 都是 565 的，所以没暴露出来而已，呵呵，以后有空在改吧。


