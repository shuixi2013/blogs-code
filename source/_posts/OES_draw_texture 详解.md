title: OES_draw_texture 详解
date: 2015-01-31 16:58:16
categories: [Other]
tags: [opengl]
---

昨天在网上找了头天，找到关于这个函数的一部分信息，好像网上对这个函数的信息不是很多，可以在这里看到对该函数的详细解释：[OES_draw_texture](http://www.khronos.org/registry/gles/extensions/OES/OES_draw_texture.txt "OES_draw_texture")

这个函数有什么用呢？它的作用是把纹理的指定片断，当然这个片断是个矩形，直接贴到你指定的视图坐标上的一个矩形上面。加快了2D图形的渲染速度。该函数是在opengl es 1.1引入，opengl 1.4引入 。

为什么需要这个函数呢，也前我们如果想要在屏幕上显示一幅图片，需要绑定一个纹理，然后指顶点坐标，以及对应的纹理坐标，指定好后，就可以把纹理显示出来 ，这样做的一个缺点是需要经过顶点变换，纹理坐标变换，然后执行片断处理，性能上不如该函数来的快，这个函数不需要执行顶点变换和纹理变换，直接进行片断处理。现在说一下如何操作吧，首先你需要绑定对应的纹理，然后调用下面函数指定需要用纹理的哪一部分。

```java
public void glTexParameterfv(int target,
                             int pname,
                             float[] params,
                             int offset)
```

这个函数大家应该很熟悉，pname 为GL_TEXTURE_CROP_RECT_OES， params是纹理的左边界，下边界，宽，高，宽和高可以为负数，这样渲染出来的效果是图片时行了左右颠倒，或者上下颠倒。然后执行下面这个函数指定屏幕坐标：

```java
void DrawTex{sifx}OES(T X, T Y, T Z, T W, T H);
void DrawTex{sifx}vOES(T* coords);
```

上面函数指定了屏幕上的一个矩形，最终效果是纹理被填充到这个矩形中，我们可以看到矩形的大小 和纹理大小没有关系，也就是说纹理可能会被放大或者缩小。Z值用于计算纹理距离屏幕的远近,从下面公式可以看出z处于0.1之间会被计算到最远最近平面中间的一点，否则会在最远或者最近平面上。

```cpp
     { n,                 if z <= 0 }
Zw = { f,                 if z >= 1 }
     { n + z * (f - n),   otherwise }
```

指定矩形上的每一个点所对应的纹理坐标是如何算出来的呢？下面是计算纹理坐标的方法。

```java
s = (Ucr + (X - Xs)*(Wcr/Ws)) / Wt
t = (Vcr + (Y - Ys)*(Hcr/Hs)) / Ht
r = 0
q = 1
```

式中，带有CR下标的是纹理截取时的参数，X,Y是某个点的屏幕坐标， 带有S下标的是屏幕截取时的参数。这个公式简单说就是把屏幕上的一点换算到纹理图片上，看它在纹理上的坐标是什么。


