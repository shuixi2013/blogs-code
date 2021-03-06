title: (转) 如何取得Android源代码
date: 2015-01-27 22:33:16
updated: 2015-01-27 22:33:16
categories: [Android Framework]
tags: [android, install]
---

Git 是 Linux Torvalds 为了帮助管理 Linux 内核开发而开发的一个开放源码的分布式版本控制软件，它不同于Subversion、CVS这样的集中式版本控制系统。在集中式版本控制系统中只有一个仓库（repository），许多个工作目录（working copy），每一个工作目录都包含一个完整仓库，它们可以支持离线工作，本地提交可以稍后提交到服务器上。分布式系统理论上也比集中式的单服务器系统更健壮，单服务器系统一旦服务器出现问题整个系统就不能运行了，分布式系统通常不会因为一两个节点而受到影响。

因为Android是由kernel、Dalvik、Bionic、prebuilt、build等多个Git项目组成，所以Android项目编写了一个名为Repo的Python的脚本来统一管理这些项目的仓库，使得Git的使用更加简单。这几天William为了拿Android最新的sourcecode，学习了一下git和repo的一些基本操作，整理了一个如何取得Android代码的How-To，今天把他贴上来。

## 1、Git的安装
在Ubuntu 8.04上安装git只要设定了正确的更新源，然后使用apt-get就可以了，有什么依赖问题，就让它自己解决吧。其中cURL是一个利用URL语法在命令行下工作的文件传输工具，会在后面安装Repo的时候用到。

<pre>
sudo apt-get install git-core curl
</pre>

## 2、安装Repo
首先确保在当前用户的主目录下创建一个/bin目录（如果没有的话），然后把它(~/bin)加到PATH环境变量中。接下来通过cURL来下载Repo脚本，保存到~/bin/repo文件中

<pre>
curl http://android.git.kernel.org/repo >~/bin/repo
</pre>

别忘了给repo可执行权限

<pre>
chmod a+x ~/bin/repo
</pre>

## 3、初始化版本库
如果是想把Android当前主线上最新版本的所有的sourcecode拿下来，我们需要repo的帮助。先建立一个目录，比如~/android，进去以后用repo init命令即可。

<pre>
repo init -u git://android.git.kernel.org/platform/manifest.git
</pre>

这个过程会持续很长的时间（至少可以好好睡一觉），具体要多少时间就取决于网络条件了。最后会看到 repo initialized in /android这样的提示，就说明本地的版本库已经初始化完毕，并且包含了当前最新的sourcecode。如果想拿某个branch而不是主线上的代码，我们需要用-b参数制定branch名字，比如：

<pre>
repo init -u git://android.git.kernel.org/platform/manifest.git -b cupcake
</pre>

另一种情况是，我们只需要某一个project的代码，比如kernel/common，就不需要repo了，直接用Git即可。

<pre>
git clone git://android.git.kernel.org/kernel/common.git
</pre>

这也需要不少的时间，因为它会把整个Linux Kernel的代码复制下来。如果需要某个branch的代码，用git checkout即可。比如我们刚刚拿了kernel/common.get的代码，那就先进入到common目录，然后用下面的命令：

<pre>
git checkout origin/android-goldfish-2.6.27 -b goldfish
</pre>

这样我们就在本地建立了一个名为goldfish的android-goldfish-2.6.27分支，代码则已经与android-goldgish-2.6.27同步。我们可以通过git branch来列出本地的所有分支。

## 4、同步版本库
使用repo sync命令，我们把整个Android代码树做同步到本地，同样，我们可以用类似

<pre>
repo sync project1 project2 … 
</pre>

这样的命令来同步某几个项目。如果是同步Android中的单个项目，只要在项目目录下执行简单的 git pull 即可。

## 5、通过GitWeb下载代码
另外，如果只是需要主线上某个项目的代码，也可以通过 GitWeb 下载，在shortlog利用关键字来搜索特定的版本，或者找几个比较新的tag来下载还是很容易的。

Git最初是为Linux内核开发而设计，所以对其他平台的支持并不好，尤其是Windows平台，必须要有Cygwin才可以。现在，得益于 [msysgit](http://code.google.com/p/msysgit/ "msysgit") 项目，我们已经可以不需要Cygwin而使用Git了。另外， [Git Extensions](http://sourceforge.net/projects/gitextensions/ "Git Extensions") 是一个非常好用的Windows Shell扩展，它能与资源管理器紧密集成，甚至提供了Visual Studio插件。它的官方网站上有一分不错的 说明文档，感兴趣的朋友可以看一看。

至于Git的参考文档，我推荐 [Git Magic](http://www-cs-students.stanford.edu/~blynn/gitmagic/ "Git Magic")，这里还有一个 [Git Magic的中文版](http://docs.google.com/View?id=dfwthj68_675gz3bw8kj#__07735763982479649 "Git Magic的中文版")。

其实最靠谱的是： [google的官方说明文档](http://source.android.com/source/downloading.html "google的官方说明文档")

原始出处： 哎，以前没写转的出处 -_-||


 

