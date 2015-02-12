title: cygwin 使用备忘
date: 2015-01-31 11:31:16
updated: 2015-01-31 11:31:16
categories: [Window]
tags: [window]
---

## 乱码

有些时候中文或是别的字符集会乱码。可以右键单击标题栏 ——> Options ——> Text 修改字符集还有编码，但是注意一下，有些如果改了对应的字符集和编码，记得上面的 Font 要选择相应支持的字体才行。例如说，如果改的是**GBK**的，字体要选择**宋体**之类的才行。有些时候改了，一些命令的输出对了，但是正常使用又不行了，可以临时改了，用完了命令后再改回去。


## 清屏

cygwin 好像没带 clear 命令，可以使用 ctrl + L 代替。

## win8.1 提示 ssh 私钥权限错误

有些时候复制 ssh 私钥的时候会出现权限问题，一般 cygwin 会提示：

<pre>
Permissions 0660 for '/home/***/.ssh/id_rsa' are too open. It is required that your private key files are NOT accessible by others. 
</pre>

一般按照提示去把 `id_rsa` chmod 600 就行了。但是在 win8.1 下有个蛋疼的问题。就是你 chmod 600 会发现还是 660 ，就是 group 和 other 的权限怎么都收不回来。后来发现是因为 win8.1 的 cygwin 有个 None 的用户组，这个组很奇怪。`id_rsa` 被分到那个组去了。解决办法很简单，cat /etc/group 把 `id_rsa` 分配到不是 None 的组就行了，例如说我分配到 Users 组去了（`chgrp Users id_rsa` 就行了）。

![](http://7u2hy4.com1.z0.glb.clouddn.com/window/cygwin-note/1.jpeg)


