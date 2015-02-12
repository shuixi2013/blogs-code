title: ubuntu minicom 使用问题总结
date: 2015-01-19 09:57:16
updated: 2015-01-19 09:57:16
categories: [Linux]
tags: [linux]
---

## 键盘无法输入

一般来说 每秒位数/奇偶校验/位数 设置成 xxxx 8N1 就可以了的，但是有些时候还是不能输入命令。这个时候可以尝试把 硬件流控制（Hardware Flow Control），软件流控制（Software Flow Control） 关掉（设置成 off），就可以输入了。

