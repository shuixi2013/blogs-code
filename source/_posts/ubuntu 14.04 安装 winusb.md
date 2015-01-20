title: ubuntu 14.04 安装 winusb
date: 2015-01-19 10:25:16
tags: [linux]
---

今天 win8 突然蓝屏启动不了，说什么 boot File:\BCD 损坏了，我擦，搞什么飞机。只好切换到 ubuntu 下想办法制作个 win8.1 的启动盘修复一下。

度娘了一下，发现 ubuntu 下有个叫 winusb 的软件可以制作 win 的启动盘，但是 14.04 没有对应的源（或者话说被屏蔽掉了）。然后 google 了一个老外的办法，手动安装：

64 bit 的：
<pre config="brush:bash;toolbar:false;">
wget https://launchpad.net/~colingille/+archive/freshlight/+files/winusb_1.0.11+saucy1_amd64.deb
</pre>

32 bit 的：
<pre config="brush:bash;toolbar:false;">
wget https://launchpad.net/~colingille/+archive/freshlight/+files/winusb_1.0.11+saucy1_i386.deb
</pre>

然后：
<pre config="brush:bash;toolbar:false;">
sudo dpkg -i winusb_1.0.11+saucy1*
</pre>

然后好像会出错的，然后再修复下依赖关系：
<pre config="brush:bash;toolbar:false;">
sudo apt-get -f install
</pre>

最后修复依赖关系那会让你装一堆东西，装就行，中间好像让你装 GRUB，然后让你选安装的地方，按 esc 退出去，然后选 yes 不安装 GRUB，这个东西别乱选，不然把 ubuntu 的引导都搞坏了就麻烦了（我的 window 已经歇菜了）。

然后命令行 winusbgui 就可以启动带 gui 的 winusb 了，用起来挺简单的：

![](http://7u2hy4.com1.z0.glb.clouddn.com/linux/winusb/1.png)


最后说一句，NND，电脑上多装一个系统还是保险点，无缘无故给老子启动文件损坏，操，操~~ >_<

