title: EFI 安装系统
date: 2015-01-31 11:55:16
categories: [Other]
tags: [window, linux, install]
---

啥叫 EFI 自己度娘、google 去，这里不解释。下面直接进入主题：

## EFI 安装 Win8（8.1）

### 制作启动安装U盘

在EFI下安装系统，是不能像以前那样分区滴，必须重新分区，需要GPT格式。找个大点的 U盘（至少4G以上），格成 fat32 格式的。用 [Universal-USB-Installer](http://pan.baidu.com/s/1c0ckax6 "Universal-USB-Installer") 把 win8 iso 制作成启动安装U盘。然后把 [EFI(x64)](http://pan.baidu.com/s/1dDILxJz "EFI(x64)") 下的文件全部 copy 到U盘根目录下。

![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/1.jpeg)

### 进入 EFI

这个比较简单，只要进入bois设置一下即可。启动项里，将UEFI这一项放到第一个：

![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/2.jpeg)

### 创建启动分区

进入EFI后，实际上就直接进入安装界面了，和正常安装相同，这里就不赘述了。

![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/3.jpeg)

到了选择分区的那一步停止（因为下一步是灰的）按shift+F10，打开命令提示行：

#### 1、把MBR磁盘转换为GPT磁盘
键入 diskpart ，打开 diskpart 工具，选择目标磁盘：

   * list disk--------------------列出系统拥有的磁盘
   * select disk 0 --------------选择0号磁盘（请根据磁盘大小，自行判断你的目标磁盘

![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/4.jpeg)

#### 2、清空目标磁盘，并转换为GPT格式
   * clean-------------------------清除磁盘,该命令会抹去磁盘上所有数据（注意备份以前的重要数据）
![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/5.jpeg)
   * convert gpt------------------将磁盘转换为GPT格式
![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/6.jpeg)
   * list partition-----------------列出磁盘上的分区，因为我们刚转换成GPT格式，因此，分区为空
![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/7.jpeg)

#### 3、建立EFI分区及系统安装分区
   * create partition efi size=200---------------建立EFI分区，大小为200M
![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/8.jpeg)
   * create partition msr size=128--------------建立MSR分区，微软默认建立的话，大小是128M
   * create partition primary size=51200-------建立主分区，大小为50G，根据自己需求调整，该分区用来安装win8
   * list partition---------------------------------列出磁盘上的分区
![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/9.jpeg)

PS：其实，一个diskpart工具，几乎可以代替其他的第三方磁盘工具了，大部分硬盘分区工具是无法更改GPT格式磁盘的分区ID的，但是diskpart可以。

### 安装win8：

关闭命令提示符，点刷新:

![](hhttp://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/10.jpeg)

选择50G的主分区安装win8:

![](http://7u2hy4.com1.z0.glb.clouddn.com/other/EFI-install/11.jpeg)

然后等着安装完成就 OK 了。

[原始出处](http://benyouhui.it168.com/thread-2475714-1-1.html "原始出处")

## EFI 安装 ubuntu

EFI Ubuntu 安装参看官网的的文档吧：  [Ubuntu EFI](https://help.ubuntu.com/community/UEFI "Ubuntu EFI")


