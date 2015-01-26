title: Linux 命令备忘
date: 2015-01-10 23:55:16
categories: [Linux]
tags: [linux]
---

百度、google都能搜得到，但是很杂。这里记录下我记不清的，但是又比较常用的一些linux命令用法。

## mount
挂载设备命令，一般挂载存储设备就用这个了（硬盘、U盘等）。

<pre>
mount [-t vfstype] [-o options] device dir
</pre>

其中：

* -t vfstype 指定文件系统的类型，通常不必指定。mount 会自动选择正确的类型。常用类型有：
    * 光盘或光盘镜像：iso9660
    * DOS fat16文件系统：msdos
    * Windows 9x fat32文件系统：vfat （一般U盘的是这种文件系统）
    * Windows NT ntfs文件系统：ntfs
    * Mount Windows文件网络共享：smbfs
    * UNIX(LINUX) 文件网络共享：nfs

* -o options 主要用来描述设备或档案的挂接方式。常用的参数有：
    * loop：用来把一个文件当成硬盘分区挂接上系统
    * ro：采用只读方式挂接设备
    * rw：采用读写方式挂接设备
    * iocharset：指定访问文件系统所用字符集

* device 要挂接(mount)的设备。

* dir设备在系统上的挂接点(mount point)。

具体的可以去看 man mount 。在挂载 samba 的时候如果说啥 mount 格式错误，一般是没有装 smbfs ， apt-get install 装一个就好。
查看电脑上的文件系统：fdisk -l 或 more /proc/partitions （一般要有root权限）。一般的用法是：

<pre config="brush:bash;toolbar:false;">
// /mnt/windows 必须存在
// 用完了卸载用 sudo umount /mnt/windows
sudo mount -t ntfs /dev/sda1 /mnt/windows 

// 挂载 samba 网络设备（以 192.168.0.8 为例）
sudo mount -t smbfs -o username=****,password="****" 192.168.0.8:/xx /mnt/samba
</pre>

要想在一开机就让linux自己挂载某个硬盘分区可以这样：编辑/etc/fstab，加入以下一行：

<pre>
/dev/sda1 /mnt/windows ntfs defaults 0 0
</pre>

卸载命令是：umount 

## find
查找文件命令。常用的形式： find path -type f -name filename -depth 。

* path: 路径，一般当前路径可以用 "." 表示。
* -type：查找的文件类型，f 表示普通文件。其他的可以看 man find。
* -name：查找文件的名字，可以用正则表达式（不过我基本上只会用*而已）。
* -depth：表示递归查找（查找子目录），好像可以设置深度的，具体的看 man 吧。

## grep
查找文件内容命令。常用的形式： grep -Irn findstrings filenames。

* I：表示忽略小写。
* r: 表示查找子目录。
* n：表示显示行号。
* v：这个表示方向查找，就是显示不包含查找内容的文件。例如 grep -Irn xx . | grep -v svn 就可以不去查找可恶的svn目录下的的东西。
* l：表示只显示文件名字，例如 grep -lr xx . 搜索结果就只显示文件名。
* findstrings：要查找的字符串，同样支持正则表达式。字符串可以加“”这样遇到要查找一些特殊字符不要加转意字符(\)，否则一些字符是命令的需要加转意字符。例如 grep -Irn "path\list" .
* filenames：要查找的文件。

## sed
流编辑器，功能十分强大，但是我目前就会用它的一点点功能而已，简单的说它能将输入文件一行一行的做处理。这样它和 grep , find 配合起来做替换的话，就十分方便了。常用形式：sed 's/old/new/g' -i files 。

* 's'：这个是替换命令，和vim的替换命令差不多。
* -i：将输出写入原文件，也就是修改原文件的内容；如果不加的话就会输出到标准输出（一般是终端）。
* files：输入的文件，可以是多个。

这里与 grep 配合一个就可以在一个工程项目里做全局的替换了：（在工程的根目录弄） 

<pre>
# grep -r old * | sed 's/old/new/g' -i
</pre>

## apt-get install
从网上下载并安装软件包。用法很简单，一般就是 sudo apt-get install xx ，但是一般会记不住包的名字。可以使用 aptitude search 关键字 （apt-cache search xx）来搜索。软件源可以修改 /etc/apt/source.list （不过一般用图形界面比较方便点），然后使用 sudo apt-get update 来更新。还有 apt-get install xx 的下载包的缓存地址在 /var/cache/apt/archives 里，这里其实可以备份起来，下次重装系统的时候就不用再重新下载了。

aptitude dist-upgrade xx 可以单独更新某个包。

## chmod 和 chown
chmod 是修改文件权限的，例如添加所属组的读写权限、添加其它用户的读写权限等。chown 是修改文件的拥有者，拥有者对该文件拥有所有权限。

## mkswap, swapon
用来创建和挂载swap分区（linux下的虚拟内存），一般在板子上开发时，可以拿U盘做swap分区（事先要格式化成swap分区格式的）。一般用法是： mkswap /dev/sda 创建swap分区（假设/dev/sda是你的U盘），然后 swapon /dev/sda 挂载上就可以了。然后可以用 free 查看结果。

## 删除不需要的内核

* dpkg --get-selections | grep linux ：可以查看自己安装了多少内核。用 uname -a 可以看到自己目前用的是哪个内核。
* 然后用 apt-get remove xxxx 就可以把自己不要的内核删掉（删掉后会自动把 grub 中对应的启动项目也删掉）。
* 不过某些时候 grub 会检测到一些没有办法启动的启动项目，这个时候可以修改 grub 的菜单文件把无用的项目去掉：
    * grub1（ubuntu 9.10 之前）：/boot/grub/menu.lst
    * grub2（ubuntu 10.04 之后）：  /boot/grub/grub.cfg

## patch
linux打补丁命令。由 vim diff 等工具生成的 .patch 文件可由该命令对源代码打补丁。一般形式为： patch -p0 < xx.patch 。p0 代表是源代码目标的第几层（文件夹深度），0就代表是根目录。一般是把 .patch 文件复制到源代码包的根目录，然后用上面的形式进行打补丁。更多的用 man patch 自己看吧。

## ftp
用 ftp 上传东西的时候，如果发的不是文本文件，在上传之前要使用 binary 命令，转化为二进制传输模式，否则上传后的文件可能无法正常使用。

## 解压 rpm 包
rpm 包可以使用 rpm -i 直接安装，也可以使用 rpm2cpio xxx.rpm | cpio -div 进行解压。

## 修改用户权限
* useradd
添加一个新用户，一般新建立一个用户就会相应的建立这个用户同名的用户组。如果要新建立用户组的话，可以用 groupadd 。

* usermod
修改指定用户的信息。可以修改这个用户的用户目录（home目录）、shell 环境（bash 还是 sh）、所属于的用户组等。其中修改说属的用户组就可以赋予和删除用户相应的用户组的权限。使用 

<pre>
usermod -a -G group1 user1
</pre>

就可以将 user1 添加到 group1 组中。具体的用法可以看 man。其中

<pre>
// 可以看到有哪些组, 组里有哪些用户
cat /ect/groups

// 可以看到用户的一些信息
cat /ect/passwd
</pre>

* chgrp
chgrp -R file 可以改变这个文件的所属于组。

* 权限说明
一般我们最常用的也就是 777 755 644 这三种 Linux主机文件目录权限原理：

<pre config="brush:bash;toolbar:false;">
444 r--r--r--
600 rw-------
644 rw-r--r--
666 rw-rw-rw-
700 rwx------
744 rwxr--r--
755 rwxr-xr-x
777 rwxrwxrwx
</pre>

三位数字代表9位的权限，分成3部分，第一部分3位表示所有者的权限，第二部分3位表示同组用户权限，第三部分3位表示其他用户权限，r代表读取权限等于4，w代表写入权限等于2，x代表执行权限等于1

比如777，第一位7等于4+2+1，所以就是rwx，所有者有读取、写入、执行的权限，第二位7也是4+2+1，rwx，同组用户具有读取、写入、执行权限，第三位7，代表其他用户有读取、写入、执行的权限。
比如744，第一位7等于4+2+1，rwx，所有者具有读取、写入、执行权限，第二位4等于4+0+0，r--，同组用户只有读取权限、第三位4，也是r--，其他用户只有读取权限。

## 修改用户密码
passwd user 可以修改指定用户的登陆密码，当然如果修改别的用户要有 root 权限。

## 监视某个端口
<pre>
# 这里是监视网络流量
watch -n 1 "/sbin/ifconfig eth0 | grep bytes"
</pre>

## 查看磁盘空间
虽然有 fdisk 可用，但是 df -h 效果更好。

## wget
这个东西下 http 的挺好用的。一般用法比较简单，直接 wget url 就行了。

## 压缩
可以用 tar，也可以用 zip：
<pre>
// gz 格式的
tar -xzvf xx.tar.gz      // 解压
tar -zvf xx.tar          // 解压
tar -czvf xx.tar.gz xx   // 压缩

// bz2 格式的
bzip2 xx.tar.bz2  // 解压 

// 递归（文件夹）中的打包为 xx.zip，其中 -0 表示不压缩，仅仅是存储
// 不加 -0 则表示用默认压缩
zip -r -0 xx.zip xx 

// 将 xx 这个文件加入到 xx.zip 中，其中能保持 xx 的文件树结构
zip -g xx.zip xx
</pre>

## dpkg
* 安装 deb 包： dpkg -i package-file.deb
* 卸载 deb 包： dpkg -r package-name
* 包名可以通过 dpkg --info package-file.deb 查看

## df
查看磁盘分区大小，可以加 -h 显示单位。

## xargs
可以显示指定让上一个命令的输出作为下一个命令的输入参数。例如： 
<pre>
# 先搜索以 buildin 结尾的文件，然后再删掉。
find . -iname *.buildin | xargs rm 
</pre>
