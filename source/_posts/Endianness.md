title: Endianness
date: 2015-01-19 20:18:16
categories: [Basics Knowledge]
tags: [basics]
---

大、小端这个东西，每隔一段时间我就会忘记，发现维基上有2张图太形象了，记不得的时候看下这2张图就能想起来了：

![](http://7u2hy4.com1.z0.glb.clouddn.com/basics/Endianness/1.png)

![](http://7u2hy4.com1.z0.glb.clouddn.com/basics/Endianness/2.png)

然后总结下，有个简单的记得方法：如果是大端存储的话，那么就是大的数值（专有名词叫 The most significant byte (MSB），俗称高位，就是好说个、十、百、千中的千）是存放在存储的低位的（就是最开始地方）。所以大端存储格式直接按顺序从存储中读出来的位数就是对的。

小端就正好反过来， MSB 是存在存储的高位的（就是后面），所以小端存储的话，从存储中读出来，要把位数反过来。

最后贴个 wiki 地址： [Endianness](http://en.wikipedia.org/wiki/Endianness "Endianness")

