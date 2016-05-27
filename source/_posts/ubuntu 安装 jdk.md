title: ubuntu 安装 jdk
date: 2015-01-19 09:59:16
updated: 2015-01-19 09:59:16
categories: [Linux]
tags: [linux, install]
---


## 10.10

* 在 /etc/apt/sources.list 里新建一个源文件：

<pre>
deb  http://archive.canonical.com/ubuntu maverick partner
</pre>

* 然后 apt-get update --> apt-get install sun-java6-jdk 。

* 值得注意的是，默认ubuntu10.10已经安装了open-JDK 可以使用java -version命令查看，当前默认使用的JDK是哪个类型的。如果直接卸载了openJDK软件包，再使用java -version命令，系统还是使用的open-JDK，这时需要修改ubuntu默认使用JDK的配置，配置默认Java使用哪个

<pre>
sudo update-alternatives –-config java
</pre>

选择“2”，再查看系统使用java的版本。注意：wordpress不好的地方就是在写代码时，经常会修改代码里的内容，上面在sudo update-alternatives –config java这句，config前面是两个“-”,wordpress会显示为一个长“-”。


## 13.04

更新下 13.04 安装 sun jdk 的方法：
 
* 如果安装了 open jdk 的，要先把 open jdk 卸掉： apt-get purge openjdk*

* 添加 sun java 源： add-apt-repository ppa:webupd8team/java （如果有问题可以先安装： apt-get install software-properties-common ），然后更新源列表： apt-get update

* 然后就可以安装 jdk 了： apt-get install oracle-java7-installer（这里装的是7，也可以安装6或是8 oracle-java6-installer, oralce-java8-installer）

* 如果要卸载之前安装的 sun jdk，用 apt-get purge oracle-java* 就行了，不要用 apt-get purge java* ，这样能删掉很多别的东西的。

## 13.10

ubuntu 13.10 之后软件源里面就没 oracle-java6-installer 了，但是如果是编译 android 的话，必须要 jdk6 才行。所以只能去 oracle 官网去下载： [oracle jdk6](http://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase6-419409.html#jdk-6u45-oth-JPR)。 我选的是 *.rpm.bin 下载的（下载还必须要注册 oracle 的帐号才能下，恶心）。然后就是手动安装了。

* 先是 chmod+x *.rpm.bin ，给这个玩意加上执行权限，然后执行一下，会解压出很多 *.rpm 包出来，可能这个东西会自动安装的吧，但是在我到机子上报错了，所以我只好手动安装这票 rpm 包了。

* 先要装 alien (apt-get install alien)，然后用 alien 把这些 rpm 包转化为 deb 包： alien --scripts --keep-version -d *.rpm 。耐心等一下，就会转化好 deb 包。

* 安装转化好到 deb 包： dpkg -i *.deb 。就终于装好 oracle-jdk-6 了。

## 14.04

android 5.0 以后就必须要 jdk7 才能编译通过了，然后好像其实 open-jdk 也能用了，所以如果一开始什么都没转的话用 apt-get install 装 open-jdk 比较好，啥路径都不要自己设置。但是如果以前就已经装了 sun jdk6 的话就有点的麻烦，话说我搞了半天也没发现要怎么删掉这个东西，其实要删掉很简单，which java 一下就能知道 jdk 的路径，直接过去删掉就没了，但是 /etc/alternatives/ 下面那个链接还在，后来实在不行，就直接把 /etc/alternatives/ 下那些 java* 的软件链接删掉了。然后去上面 oracle jdk 的下载页面把 jdk 下来，然后转化为 deb 包装一下。不过 14.04 上 jdk7 的 deb 包装好后，不会自己设置路径了。

唉，蛋疼。jdk7 装在 /usr/java 下面，有一个 jdk1.7.0_75 ，此外这个目录下还有一个 default 的软链接文件夹。看到这里就知道要怎么搞了吧。在 .bashrc 中增加：

```bash
export JAVA_HOME=/usr/java/default/
export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib 
export PATH=$PATH:$JAVA_HOME:$JAVA_HOME/bin
```

然后命令行下 java 就能用了。不用还有一件蛋疼的事，eclipse 还不认这个，本来我还想怎么设一下，让 eclipse 认的，后面发现 eclipse 报的错是这样的：

<pre>
A Java Runtime Environment (JRE) or Java Development Kit (JDK)
must be available in order to run Eclipse. No Java virtual machine
was found after searching the following locations:
/home/mingming/eclipse/jre/bin/java
java in your current PATH
</pre>

这就好办了么，去 eclipse 目录下（上面那个 /home/mingming/eclipse 就是我 eclipse 的目录），创建一个 jre 的软件链接指向 /user/java/default 就行了。

可以通过 android source 的官网装 open-jdk，其实不用 jdk-8，jdk-7 就可以： sudo apt-get install openjdk-7-jdk





