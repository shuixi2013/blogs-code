title: GL 中的纹理坐标
date: 2015-01-31 17:07:16
updated: 2015-01-31 17:07:16
categories: [Other]
tags: [opengl]
---

GL 中的纹理坐标是左上（0，0）、右上（1，0）、右下（1，1）、左下（0，1），来几张图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/opengl/texture-coords/1.jpeg)

![](http://7u2hy4.com1.z0.glb.clouddn.com/opengl/texture-coords/2.jpeg)

还有纹理坐标可以为负的，会有很有趣的现象（反过来贴），范围是 0 ～ 1，超过 1 现象会非常奇怪（负数范围也要在 -1 ～ 0）。这里记一下，免得每次都忘记坐标系统。


