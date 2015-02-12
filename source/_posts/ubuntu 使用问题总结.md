title: ubuntu 使用问题总结
date: 2015-01-19 09:54:16
updated: 2015-01-19 09:54:16
categories: [Linux]
tags: [linux]
---

## 激活 root 账号

默认 root 账号是不能切换和登陆的，只能用 sudo 来了临时获取一个命令的 root 执行权限。有些时候为了方便可以激活 root

账号。 sudo passwd root ，设置一个密码就可以了。

## 中文输入安装之后无法调出

安装了 ibus 无法调出，一般是 ibus demon 没启动，然后输入法也没设置为中文。ibus-setup 可以调出输入法设置，然后把

中文输入法设置好。不过要想个办法让 ibus demon 自动启动（没启动的，ibus-setup 守护进程就启动了）。可以装一个 im-swtich 然后设置默认输入法为 ibus

## 自动挂载 ntfs 分区

一般来说安装了双系统（window、ubuntu），window 的 ntfs 分区，ubuntu 会默认挂载。可读写，但是无法使用 git 和 repo，因为挂载的权限不太对。同理，移动硬盘的如果也是使用 ntfs 分区的话，也会有这个问题（拿移动硬盘存 android mirror 就会遇到这些问题了）。这个时候可以安装 ntfs-config 这个软件来，然后 sudo 运行，勾选 mount 成可读、写的，然后 auto-config 一下就行了。

然后在 /etc/fstab 会记录 mount 的路径。可以自己编辑的，把自己的 window 分区挂载到自己指定的目录（这个目录要自己提前 mkdir 建好，不然无法挂载的），同理移动硬盘的分区可以自己指定路径的，像下面这样（这样 mount 路径固定，写某些脚本会方便点，不然时不时变一下 mount 路径，烦得很）：

```shell
# /etc/fstab: static file system information.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>

#Entry for /dev/sdc7 :
UUID=0cc75462-fe83-4928-bdd9-514750e91f7f   /   ext4    errors=remount-ro   0   1
#Entry for /dev/mapper/isw_ebgeghbdjj_ssd_gpt3 :
UUID=5f0a43bf-72d5-45f1-9c66-c5a355ca454a   /boot   ext4    defaults    0   2
#Entry for /dev/mapper/isw_ebgeghbdjj_ssd_gpt1 :
UUID=6483-74C9  /boot/efi   vfat    defaults    0   1
#Entry for /dev/sdc8 :
UUID=b2317adb-c3a3-4608-af58-78b9d8a836dd   /home   ext4    defaults    0   2
#Entry for /dev/sdc6 :
#UUID=F084F63784F60040  /media/BIOS_RVY ntfs-3g defaults,locale=en_US.UTF-8 0   0
#Entry for /dev/sdc2 :
UUID=5046ED5746ED3DFA   /media/local/collect    ntfs-3g defaults,locale=en_US.UTF-8 0   0
#Entry for /dev/sdc4 :
UUID=64C626E5C626B768   /media/local/game   ntfs-3g defaults,locale=en_US.UTF-8 0   0
#Entry for /dev/mapper/isw_ebgeghbdjj_ssd_data2 :
UUID=32D67187D6714BDB   /media/local/ssd    ntfs-3g defaults,locale=en_US.UTF-8 0   0
#Entry for /dev/mapper/isw_ebgeghbdjj_ssd_gpt4 :
#UUID=E6C0F9E8C0F9BF3D  /media/local/win    ntfs-3g defaults,locale=en_US.UTF-8 0   0
#Entry for /dev/sdc3 :
UUID=BAC60DBFC60D7D3F   /media/local/work   ntfs-3g defaults,locale=en_US.UTF-8 0   0

#Entry for /dev/sdd3 :
UUID=E214EC8C14EC64CF   /media/removable/media  ntfs-3g defaults,nosuid,nodev,locale=en_US.UTF-8    0   0
#Entry for /dev/sdd2 :
UUID=27E4DC5E2CC9645D   /media/removable/other  ntfs-3g defaults,nosuid,nodev,locale=en_US.UTF-8    0   0
#Entry for /dev/sdd1 :
UUID=66118DF048E7A1EE   /media/removable/work   ntfs-3g defaults,nosuid,nodev,locale=en_US.UTF-8    0   0

#Entry for /dev/sdc5 :
UUID=d1143252-352a-4c36-8560-d35c8661089a   none    swap    sw  0   0

#UUID=F084F63784F60040  /media/BIOS_RVY ntfs-3g defaults,locale=en_US.UTF-8 0   0
```

然后注意下 UUID 不要去动 ntfs-config 自动生成的（UUID 是每一个硬盘分区的识别号），当然如果生成有错误，可以用下面2个命令查看：

<pre>
blkid -s UUID

ls -l /dev/disk/by-uuid
</pre>

然后有些时候 window 的 ntfs 分区会挂载失败，然后出来下面的错误：

![](http://7u2hy4.com1.z0.glb.clouddn.com/linux/ubuntu-memos/unable-mount.png "挂载错误")

这个时候可以用 sudo ntfsfix -d xx（xx 就是图中报错的那个 device： /dev/sdc3，这个用上面那个2个命令也可以看得到的）。一般来说输出修复成功就可以挂载了。如果还是实在不行，网上有人话说是因为 win8 的快速启动导致 ntfs 分区没有被 window 完全卸载导致的。实在不行，尝试把 win8 的快速启动关掉吧。

## vpn 无法连接

ubuntu 一安装好就可以直接新建 PPTP 方式的 vpn 的（点右上角的网络连接的 vpn 配置那里）。除了记得设置 Advanced 那里按下面的设置：

![](http://7u2hy4.com1.z0.glb.clouddn.com/linux/ubuntu-memos/vpn-config-adv.png)

把 IP、用户名和密码设置好后，一般就能连上了。不过有些时候有点蛋疼，怎么也连不上，刚开始以为是什么 vpn PPTP 的什么软件没装，搞了半天还是没用。后面无意发现设置 General 那里有一个 **允许所有用户连接** 的选项（默认一般是勾上的），不知道什么时候没勾上，勾上后就可以连了。

![](http://7u2hy4.com1.z0.glb.clouddn.com/linux/ubuntu-memos/vpn-all-users.png)




