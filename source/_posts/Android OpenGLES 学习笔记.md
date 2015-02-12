title: Android OpenGLES 学习笔记
date: 2015-01-25 23:28:16
updated: 2015-01-25 23:28:16
categories: [Android Development]
tags: [android, opengl]
---

## GL10

### 纹理问题

贴纹理的时候最好是要 2^n 字节对齐，这里说的是最后绑定到 GL 的那个图片（如果这个图片是由别的图片组合的，则组合的小图片没有这个要求）。还有纹理的大小不能超过 GL 最大纹理大小的限制。查询方法： （这里是 GL 标准的，应该还有些特定硬件的扩展的）

```java
// 最大绑定纹理大小, N x N , 应该是字节，我自己试验的结果。
glGetIntegerv(GL10.GL_MAX_TEXTURE_SIZE, statusBuffer) ;

// 最大绑定纹理的个数。
glGetIntegerv(GL10.GL_MAX_TEXTURE_UNITS, statusBuffer);
```

### 纹理渲染和颜色渲染冲突

开启了纹理渲染后，好像会对颜色渲染有影响，所以好的习惯是需要纹理渲染时，再打开，渲染完了后，马上关掉，不要让它一直开着。颜色渲染的同理。

## GL20



