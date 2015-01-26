title: ubuntu 10.10 安装 jdk
date: 2015-01-19 09:59:16
categories: [Linux]
tags: [linux, install]
---

* 在 /etc/apt/sources.list 里新建一个源文件：
<pre>
deb  http://archive.canonical.com/ubuntu maverick partner
</pre>

    * 然后 apt-get update --> apt-get install sun-java6-jdk 。

    * 值得注意的是，默认ubuntu10.10已经安装了open-JDK 可以使用java -version命令查看，当前默认使用的JDK是哪个类型的。如果直接卸载了openJDK软件包，再使用java -version命令，系统还是使用的open-JDK，这时需要修改ubuntu默认使用JDK的配置，配置默认Java使用哪个

<pre>
sudo update-alternatives –config java  
</pre>

选择“2”，再查看系统使用java的版本。注意：wordpress不好的地方就是在写代码时，经常会修改代码里的内容，上面在sudo update-alternatives –config java这句，config前面是两个“-”,wordpress会显示为一个长“-”。


* 更新下 13.04 安装 sun jdk 的方法： 
    * 如果安装了 open jdk 的，要先把 open jdk 卸掉： apt-get purge openjdk*

    * 添加 sun java 源： add-apt-repository ppa:webupd8team/java （如果有问题可以先安装： apt-get install software-properties-common ），然后更新源列表： apt-get update

    * 然后就可以安装 jdk 了： apt-get install oracle-java7-installer（这里装的是7，也可以安装6或是8 oracle-java6-installer, oralce-java8-installer）

    * 如果要卸载之前安装的 sun jdk，用 apt-get purge oracle-java* 就行了，不要用 apt-get purge java* ，这样能删掉很多别的东西的。


* ubuntu 13.10 之后软件源里面就没 oracle-java6-installer 了，但是如果是编译 android 的话，必须要 jdk6 才行。所以只能去 oracle 官网去下载： [oracle jdk6](http://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase6-419409.html#jdk-6u45-oth-JPR)。 我选的是 *.rpm.bin 下载的（下载还必须要注册 oracle 的帐号才能下，恶心）。然后就是手动安装了。

    * 先是 chmod+x *.rpm.bin ，给这个玩意加上执行权限，然后执行一下，会解压出很多 *.rpm 包出来，可能这个东西会自动安装的吧，但是在我到机子上报错了，所以我只好手动安装这票 rpm 包了。

    * 先要装 alien (apt-get install alien)，然后用 alien 把这些 rpm 包转化为 deb 包： alien --scripts --keep-version -d *.rpm 。耐心等一下，就会转化好 deb 包。

    * 安装转化好到 deb 包： dpkg -i *.deb 。就终于装好 oracle-jdk-6 了。

