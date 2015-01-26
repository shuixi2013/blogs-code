title: MiniGUI 源码分析笔记
date: 2015-01-21 21:12:16
categories: [MiniGUI]
tags: [minigui]
---

随手记记，免得忘记。

## 线程版启动的线程

线程版的 MiniGUI 在 InitGUI 后，会启动额外3个线程：

* __mg_desktop（pthread_t）：桌面线程，线程函数 DesktopMain。这个线程用于运行桌面处理函数，处理各种桌面消息，以及分发各种消息给所需要的窗口。

* __mg_parsor：事件循环线程，线程函数 EventLoop。这个线程用于不停的接受底层驱动事件（鼠标、键盘等），然后转化为 MiniGUI 消息放入消息队列。

* __mg_timer：全局计数线程，线程函数 TimerEntry。这个线程用于提供 MiniGUI 最小计数单位（定时器用）。

所以，加上 MiniGUI 应用程序本身，如果不再额外创建线程的话，线程版的 MiniGUI 一共有4个线程。

## 有层 alpha 的时候会忽略逐点 alpha

目前的 MiniGUI 是这样的，如果既要有层 alpha 有要逐点 alpha，只能分2次混合，一次使用逐点 alpha，一次使用层 alpha，中间使用 memdc 倒一次。这个可以看看 GAL_UpperBlit、GAL_LowerBlit、GAL_MapSurface、GAL_CalculateBlit、GAL_CalculateBlitN 等等这些函数。有空最好把目前 MiniGUI BitBlt 的流程总结下。哎～～以前写 GAL 的时候有点印象的，隔了段时间又忘记了。

