title: 反编译 Android apk
date: 2015-01-25 22:50:16
updated: 2015-01-25 22:50:16
categories: [Android Development]
tags: [android]
---

如果 java 程序编译的时候没有混淆代码的话，就可以比较容易的反编译。但是反编译不一定就能 100% 的得到原始的代码，不过已经可以拿来做参考了。

## apk --> xml

从 google code ([apktool下载](http://code.google.com/p/android-apktool/ "apktool下载")) 下载 apktool 工具，按照 google code 上的说明：

1. Download apktool-install-linux-* file
2. Download apktool-* file
3. Unpack both to /usr/local/bin directory (you must have root permissions)

设置好 PATH 路径，让系统能找到 apktool。然后 apktool d -f xx.apk /home/mingming/tmp/apk 解压出 apk 完成的 xml 和 资源文件。

## class --> jar

从 google code ([dex2jar 下载](http://code.google.com/p/dex2jar/downloads/list "dex2jar 下载")) 下载 dex2jar 工具。然后把 apk 文件改后缀为 .zip ，然后解压出 zip。在里面找到 class.dex ，然后把 class.dex 复制到 dex2jar 目录下。然后执行 dex2jar.sh class.dex (可能之前需要把 dex2jar.sh 增加可执行属性)。成功的话会在本目录下生成 classes.dex.dex2jar.jar 。

## jar --> java

从 这里 ([jd-gui 下载](http://jd.benow.ca/ "jd-gui 下载")) 下载 jd-gui 工具。然后运行 jd-gui 打开上一步生成的 jar 包，然后就可以在 jd-gui 里看到 jar 里的源代码里。可以使用 jd-gui 的 file 菜单里的 save all source 命令，把源代码导出成一个 zip 包里。

## 总结

结合第一步得到的 xml 文件 和 res 文件，以及第三步得到的 java 源代码，就差不多可以还原 apk 程序了。

## 重新打包

参考： http://nitinzzz.blogspot.com/ (注：这个被墙了 !!=_=!!)

* 首先，准备工具
    * apktool apk_manager ， [点这里下载](http://u.115.com/file/aq38jnat "点这里下载")
    * zip 的 管理工具 ，这个 ubuntu 底下默认有了。
    * jdk 的 jarsigner ， 我这里路径为 /home/nxliao/tool/android/jvm/java/jdk1.6.0_25/bin/jarsigner
    * android sdk 的 debug.keystore ，在ubuntu下为 ~/.android/debug.keystore

* 准备实验对象
Fishing Joy ， [点这里下载](http://u.115.com/file/bh5nw429 "点这里下载")

* 改装
    * 用 zip 管理工具打开这个 apk，删除里面的 META-INF 目录
    * 用 apktool 解压处理过的 apk 
<pre>
 $ ./apktool d ~/tmp/jianjiuhongchenfengha_V1.0_mumayi_85342.apk ~/tmp/jianjiuhongchenfengha
</pre>
    * 用 vi 打开目标代码 
<pre>
$ vi ~/tmp/jianjiuhongchenfengha/smali/com/sg/android/fish/FishActivity.smali
</pre>
    * 转到第 330行（在 .method private init()V 内），将 const/16 v6, 0xc8 修改成 const/16 v6, 0x647d （也可以设置成其它数值），即可将初始金钱改成 0x647d =25752
    * 保存退出
    * 用 apktool 重新打包 apk 
<pre>
$ ./apktool b ~/tmp/jianjiuhongchenfengha ~/tmp/jian.apk
</pre>
    * 这时候新的apk还不能直接安装，需要打上签名。用jdk的 jarsigner 打上签名 
<pre>
   $ jarsigner -verbose -storepass android -keystore ~/.android/debug.keystore ~/tmp/jian.apk androiddebugkey
</pre>

## odex 转 dex

odex文件无法直接使用dex2jar进行直接反编译成jar，必须先转为dex，才能继续反编译。用到的工具 baksmali smali

下载地址：http://code.google.com/p/smali/downloads/list 

步骤：

* 1，分解odex文件 java -jar baksmali-1.2.4.jar -x ../TEST.odex 这时候出现问题：

<pre>
Error occured while loading boot class path files. Aborting. 
org.jf.dexlib.Util.ExceptionWithContext: Cannot locate boot class path file core.odex 
at org.jf.dexlib.Code.Analysis.ClassPath.loadBootClassPath(ClassPath.java:237) 
at org.jf.dexlib.Code.Analysis.ClassPath.initClassPath(ClassPath.java:145) 
at org.jf.dexlib.Code.Analysis.ClassPath.InitializeClassPathFromOdex(ClassPath.java:110) 
at org.jf.baksmali.baksmali.disassembleDexFile(baksmali.java:96) 
at org.jf.baksmali.main.main(main.java:278) 
</pre>

这是由于缺少core.odex, ext.odex, framework.odex, android.policy.odex, services.odex, bouncycastle.odex, core-junit.odex, 这7个文件的问题，将framework下的这5个odex文件一并考到同级目录下，在运行命令即可。

* 2，生成classes.dex

java -Xmx512M -jar smali-1.2.4.jar out -o classes.dex


