title: Android 智能指针备忘
date: 2015-01-28 20:12:16
updated: 2015-01-28 20:12:16
categories: [Android Framework]
tags: [android]
---

在 framework 的 native 层一大票 sp<xx>, wp<xx> 之类的玩意。这个是 android 自己搞的一套智能指针技术，就是为了偷懒用的，利用引用计数和构造、析构函数实现内存自动释放的功能（其实我挺讨厌这套东西，刚看着头晕，自己手动 new、delete 对象有那么难么？）。网上罗升阳的一个分析挺不错的： [智能指针分析](http://blog.csdn.net/luoshengyang/article/details/6786239 "智能指针分析")。不过这个太长的，这里稍微记点总结，方便之后查看。

* **RefBase**
所有要支持智能指针的类的基类，如果你写的类想要支持智能使用，就要继承这个东西。支持强指针（sp）和弱指针（wp）。

* **LightRefBase**
轻量级指针基类，这个和上面的区别就是只支持强指针（sp）。

* **生命周期**
android 的智能指针有个 flag 可以设置指针智能的策略： 

<pre config="brush:bash;toolbar:false;">
//! Flags for extendObjectLifetime()
enum {
    OBJECT_LIFETIME_STRONG  = 0x0000,
    OBJECT_LIFETIME_WEAK    = 0x0001,
    OBJECT_LIFETIME_MASK    = 0x0001
};
</pre>

`OBJECT_LIFETIME_STRONG`: 默认策略，对象销毁只受强引用计数影响，如果强引用计数为0，就会 delete 这个对象。

`OBJECT_LIFETIME_WEAK`： 使用 extendObjectLifetime（`OBJECT_LIFETIME_WEAK`） 设置。设置后对象销毁同时受强引用计数和弱引用计数的影响。只有当强引用计数和弱引用同时为 0 的时候才会 delete 这个对象。

* **sp**
强指针，用这个指针指向对象会导致强引用计数 +1，当这个指针销毁（析构的）的时候强引用计数会 -1。这个指针能够直接操作指向的对象（有 -> 和 * 操作）。

* **wp**
弱指针，用这个指针指向对象会导致弱引用计数 +1，当这个指针销毁（析构的）的时候弱引用计数会 -1。弱指针和强指针的最大的区别在于，弱指针不能直接操作指向的对象（没有 -> 和 * 操作）。如果要操作指向的对象需要调用其的 promote() 方法将弱指针转化为强指针访问指向的对象。这个 promote() 如果成功的，会返回对应的强指针，同时导致强引用和弱引用都 +1。但是可能会失败，可能是对象已经被 delete 或是类不允许弱引用转化为强引用。


