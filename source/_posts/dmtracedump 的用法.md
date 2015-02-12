title: dmtracedump 的用法
date: 2015-01-26 23:35:16
updated: 2015-01-26 23:35:16
categories: [Android Development]
tags: [android]
---

这个东西是 android sdk 中自带的一个开发工具，用来分析程序的性能的，和 traceview 配合使用。这个工具需要使用 Debug 中的 trace 方法来生成 .trace 文件（具体方法看 android 的开发文档）。

然后通过 .trace 可以生成一个函数调用栈的图像。用法为：

<pre config="brush:bash;toolbar:false;">
dmtracedump -g my.png my.trace
</pre>

会生成一个 png 图片。这个工具的官方文档说明真是水，居然连最基本的用法都没写。这玩意生成的图，怎么分析程序的瓶颈我好像不怎么看得懂，但是我发现用来分析 framework 的一些流程倒是挺不错的 -_-|| 。

