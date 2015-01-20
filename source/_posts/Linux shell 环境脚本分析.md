title: Linux shell 环境脚本分析
date: 2015-01-13 20:50:16
tags: [linux]
---

侯哥之前帮我在编译服务器上弄了个工作环境，vim啊，调试不同的minigui版本都挺好用的。不过总想弄明白这些脚本是咋回事。在不停的提问、百度、google之下，稍微明白了点；赶紧记下来吧，免得又忘记了。

## bashrc

目前的工作环境的目录安排是这样的（以下目录分别是在/home/xx/下）：

* source：源代码存放目录.
* build：编译产生的中间文件目录.
* target：minigui的库路径.

每个不同的minigui版本在source下存放不同的源代码（例如rel－3－0）；在build下用mkdir建立相应的目录（build/rel-3-0），然后build/rel-3-0下有可以建立minigui目录（用于编译此版本的minigui），mg－sample（编译此版本的实例程序）；在target也建立相应的目录（target/rel-3-0）。以上目录建立完成后就可以用命令：

<pre>
envbuild rel－3－0
</pre>

来自动切换到rel－3－0版本的编译环境。此时会自动进入到build/rel－3－0目下。然后可以用lndir命令把source下rel－3－0的源代码链接过来，这样就能保证源代码目录的“干净”（编译都在build下）。把源码链接过来后，就可以用acfg命令配置生成makefile文件；然后用mi命令就编译，把编译好的库安装到相应的target/rel-3-0目录下。之后在build可以把mg－sample的链接过来，然后配置自动生成makefile文件，编译、运行实例程序了（这个时候会链接正确的minigui库哦）。

以上这些功能是怎样通过脚本来实现的呢，那从.bashrc看起吧。

<pre config="brush:bash;toolbar:false;">
ss=/home/mingming/source
bb=/home/mingming/build
tt=/home/mingming/target
ftp=/home/ftp/pub
alias make='make -j5'
alias mc='make clean'
alias mi='make -j5 install'
alias mci='make clean; make -j5 install'
alias cfg='./configure --prefix=${TARGET_CFG}'
alias acfg='./autogen.sh;./configure --prefix=${TARGET_CFG}'
alias acmci='./autogen.sh;./configure --prefix=${TARGET_CFG}; mci'
#alias envbuild=". ${bb}/env.rc $*; cd ${bb}/${TARGET}"
#alias envsource="$ss/cd.sh $*"
alias buildtarget3="${ss}/build-target-3.sh $*"
alias envbuild=". ${bb}/env.sh $*"
alias envtarget=". ${tt}/set-env.sh $*"
alias cpexe="${bb}/cpexe1.sh $*"
export ss bb tt ftp
</pre>

.bashrc这个脚本其实它的很具体的一些的东西我还是不太清楚，但是我目前知道的一点就是它是bash shell的环境配置文件（ubuntu上默认的shell是bash）。在它里面设置的一些环境信息会在用户登录的时候读入到shell中。我们先来看看上面这段.bashrc里的内容。ss、bb、tt分别就是上面我说的建立的source、build、target目录。后面的几个alias就能知道前面的acfg、mi的真正面目啦（alias这个命令看网上的说法应该是定义一个命令的别名，可以说是相当于简写？）。后面就定义了envbuild和envtarget命令啦。这里可以看到envbuild和envtarget还分别用到了build/env.sh和target/set-env.sh，这个2个脚本我们后面再看。这里可以看到这2个命令最后还有一个 $* 的符号。我虽然没确切的明白这个符号什么意思，但是我猜应该是传递envbuild xx这个命令后接的所有参数的意思（例如envbuild rel－3－0，那 $* 应该就只有一个参数：rel－3－0）。我可是根据后面的脚本这样猜的咧，应该是对的吧。

## env.sh

<pre config="brush:bash;toolbar:false;">
#!/bin/sh

if test $# -eq 0; then
echo "target is not specify!"
echo "configure failed!"
exit 1
fi

export TARGET_NAME=$1
export TARGET=$1
TT=$tt
BB=$bb
TARGET_HOME=$TT
DEBUG=$2

if test -z $DEBUG; then
DEBUG="-O2 -Wall"

else
shift
a=""
for i in $@; do
a="$a $i"
echo $a
done
DEBUG=$a
fi

CFLAGS="-I${TARGET_HOME}/${TARGET_NAME}/include $DEBUG" 
CPPFLAGS="-I${TARGET_HOME}/${TARGET_NAME}/include $DEBUG" 
LDFLAGS="-L${TARGET_HOME}/${TARGET_NAME}/lib" 

export CFLAGS CPPFLAGS LDFLAGS
</pre>

env.sh脚本刚开始是一个测试命令。test这个命令，会检测后面的表达式，若条件成立，则由$?返回0值，否则返回非0值（一般是1）；然后if由$?的返回值决定执行流程，若$?为0则执行then后面的语句。这里就可以看得到出这个检测命令的意思就是如果evnbuild后面什么参数（就是你的环境目录啦）都没给的话（$# -eq 0）就会提示错误信息，然后退出。

然后是又定义了几个变量，其中有用到 $1、$2 的值的。这2个我网上查到的信息是第1个和第2个参数的值。这个是env.sh的参数哦，可是我们好像没有显示的给env.sh什么参数啊；但是结合前面envbuild的定义就可以看得出了：env.sh $* 。所以我前面猜这个 $* 是后面所有参数的意思，这里看来应该是这样了。这里把TARGET_NAME＝$1（我们假设之前的envbuild命令后面接的是rel－3－0），TARGET_HOME＝$TT（TT=$tt）然后后面的：

<pre>
LDFLAGS="-L${TARGET_HOME}/${TARGET_NAME}/lib" 
CFLAGS="-I${TARGET_HOME}/${TARGET_NAME}/include $DEBUG"
</pre>

就可以知道为什么编译mg－sample能够链接到正确的库路径了。这里解析得到的就是：/home/mingming/target/rel-3-0/lib；正好就是之前minigui库的安装路径。如果换成rel-3-0-arena的版本的话，也能正确的找到minigui的库路径。

这里还有个 if test -z $DEBUG 的测试命令。DEBUG＝$2，这个我猜应该是envbuild后接了minigui的版本名字，后面再接一些特定的调试开关命令用的。因为$1是minigui的版本名字。这样就很好理解了，如果你什么调试开关都不写的话（这样test -z $DEBUG就成立），就用默认调试开关DEBUG＝"-O2 -Wall"；否则就将你输入的调试开关赋给DEBUG。然后可以看到CFLAGS最后将DEBUG给拼接上去了。


<pre config="brush:bash;toolbar:false;">
#change TARGET for copy file.
TARGET_CFG=$TT/$TARGET_NAME
export TARGET_CFG
export TARGET_HOME

export PS1="\[\033[01;31m\]$TARGET_NAME\[\033[01;34m\] \w \$\[\033[00m\] "
#echo "Target is $1..."
export PATH=$TARGET_CFG/bin:$PATH

target=$TARGET_NAME
export TARGET_NAME=$target

if test -d $BB/$target; then
cd $BB/$target
if test  -f self.sh; then
. ./self.sh
fi
fi

if test -d $TT/$target; then
envtarget $TARGET_NAME
cd $TT/$target
if test  -f self.sh; then
. ./self.sh
fi
fi
cd $BB/$target



#./configure --prefix=${TARGET_CFG}
</pre>

这里的PS1是根据当前的mingiui版本名字（也就是envbuild命令后面给的名字）改变shell的提示符。具体的代码含义可以去 其志 的个人wiki地盘去看 ^_^。PATH变量也增加了当前minigui库安装路径的bin目录。这里说下 export PATH=$TARGET_CFG/bin:$PATH 这样的用法，就是相当于在当前变量的基础上增加某些表达式，在设置某些查找库、二进制文件的路径上很有用。

接下来的 test -d $BB/$target 这个测试，应该是当/home/mingming/build/rel-3-0目录存在（假设是envbuild rel-3-0），并且该目录下有self.sh这个脚本文件，就去执行这个self.sh的脚本。这个应该估计是针对某些特殊minigui版本配置用的脚本，目前我还没用这个。

然后是 test -d $TT/$target 这个测试。这个测试是当/home/mingming/target/rel-3-0目录存在（同之前的假设），就会执行 envtarget $TARGET_NAME 这个命令。最后我们可以再看一下set-env.sh这个脚本了。（环环相扣啊）

## set-env.sh

<pre config="brush:bash;toolbar:false;">
#/bin/bash
if test $# -eq 0; then
echo "error:Target is not specic."
exit 0
fi

TT=$tt
SS=$ss
TT_DIR=$TT/$1

export MG_CFG_PATH=$TT_DIR/etc
export MG_RES_PATH=$SS/res-trunk/
export LD_LIBRARY_PATH=$TT_DIR/lib:$LD_LIBRAY_PATH
export PATH=$TT_DIR/bin:$PATH
export BOOTCLASSPATH=$TT_DIR/usr/local/share/jamvm/classes.zip:$TT_DIR/usr/share/classpath/glibj.zip:$TT_DIR/usr/local/share/mgjni/mgjni.zip
export CLASSPATH=.:$TT_DIR/usr/share/jamvm/classes.zip:$TT_DIR/usr/share/classpath/glibj.zip

echo "Target is $1..."
export PS1="\[\033[01;31m\]$1\[\033[01;34m\] \w \$\[\033[00m\] "

cd $TT_DIR
if test  -f ./self.sh; then
source self.sh
fi

cd $TT_DIR/bin
</pre>

哈哈，到这里就可以可以看到所有的minigui的一些环境信息都设置好了：

* MG_CFG_PATH：Minigui.cfg配置文件的路径
* MG_RES_PATH：minigui资源文件的路径
* LD_LIBRARY_PATH：minigui的链接库路径

有一些java运行时的，暂时没用到；其它的是和前面重复定义了的 -_-||。最后执行完set-env.sh（应该说是执行完envtarget命令）后，env.sh最后会 cd $BB/$target 进入到相应的环境build目录下。

现在我大致上知道我现在用的这些脚本在后面干了些什么事了。不过上次听万大师的介绍，他用的脚本更强了，可以根据环境的不同配置不同的mingiui的configure。不过这个好像更麻烦，以后有空再研究下了。

## 改进

有些项目用默认的 MiniGUI configure 不行了，所以就要手动改 configure 配置。这里可以在 .bashrc 里加2个映射命令：

<pre>
alias cb='cd ${bb}/${TARGET_NAME};./build.sh;cd ${bb}/${TARGET_NAME}/minigui'
alias acb='./autogen.sh;cd ${bb}/${TARGET_NAME};./build.sh;cd ${bb}/${TARGET_NAME}/minigui'
</pre>

然后在 /build/ 目录下放一个 build.sh 的 configure 脚本，MiniGUI 的编译源代码要放到 /build/minigui/ 下（一般我都是这么做的）。要用指定配置选项的时候就用 cb, acb 命令，要用默认配置选项的时候就用 cfg, acfg 命令。手法很简陋，但是自己用着还是蛮顺手的。

## 附件

[set.zip](http://pan.baidu.com/s/1hq6BqWO)

