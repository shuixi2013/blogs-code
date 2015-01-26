title: Android 坑爹大全 —— Paint mask filter 无效
date: 2015-01-26 23:53:16
categories: [Android Development]
tags: [android]
---

android 的 Paint 提供2种边缘效果：一种是 BlurMaskFilter，一种是 EmbossMaskFilter。BlurMaskFilter 是模糊效果，EmbossMaskFilter 是浮雕效果。这里注意下，Paint 提供的只是边缘效果而已（egde），不是整体的效果，例如这个模糊效果不是类似 window vista 那种的整体的背景毛玻璃效果，边缘效果是这样的：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/suck-problem-paint-mask-filter/1.jpeg) 

然后用法很简单：

```java
// 创建一个 BlurMaskFilter（设置下参数）设置给 Paint
paint.setMaskFilter(new BlurMaskFilter(radius, BlurMaskFilter.Blur.NORMAL));
// 然后 draw 的时候使用这个 Paint 去画就行了
canvas.drawBitmap(bmp, null, rect, paint);
```

BlurMaskFilter 有几个参数： radius 是模糊半径（值越大，模糊范围就越大），mode 是模式，有几个选择：

* NORMAL： 内外边缘都模糊
* INNER： 内边缘模糊
* OUTER： 外模糊，（4.4 和 之前的版本效果不一样，有点坑爹）
* SOLID： 外边缘模糊

从左到右依次是： NORML、INNER、OUTER、SOLID：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/suck-problem-paint-mask-filter/2.jpeg)

上面 OUTER 是在 4.2 上的效果，在 4.4 上是这样的，可能内部实现变了：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/suck-problem-paint-mask-filter/3.jpeg)

Emboss 效果没贴上来了，可以自己试验。从上面图上可得出，这个东西的算法对于某些图像还是有点缺陷，要使用的话，尽量保证真正的图片内容不要紧靠着边缘（那个兔子的一部贴着边缘了，效果不是很好）。


但是今天说的是坑爹的一点：之前在 4.2 和 4.4 上怎么跑都效果，还以为是代码哪里弄错了，跑去源码看了一下，发现其实是 skia 库只代的 effect 而已，在 java 层拉了几个接口，skia 的 SkBlurMaskFilter 和 SkEmbossMaskFilter（代码在 skia 的 effect 目标下面，在 skia 的 samples 里有示例怎么使用）。

后来想了一下是不是因为开了**硬件加速**的原因，果然把硬件加速关了，效果就出来了（4.0 之后，默认开硬件加速的）。可以在 activity 级别关，也可以在使用的 view 上使用 setLayerType(View.LAYER_TYPE_SOFTWARE, null) 关硬件加速。

想想看，官方文档里说硬件加速（使用 openGL）只支持点、画线、画多边形、贴图、旋转、平移、缩放、aplha渐变这几个吧，这种稍微高级点的东西不支持正常。不过一开始没想到而已。

网上看到有人也发现 4.0 之后这几个效果不生效了，他们的办法是把 targetSDKVersion 改低，但是其实是没什么意义的。主要源头在于硬件加速。

