title: Linux Shell 进阶环境脚本分析
date: 2015-01-18 23:44:16
tags: [linux]
---

之前编译环境脚本是用候哥的，简单方便。不过有个缺点就是如果要特别指定一些配置的话，不太方便，需要每次都自己去设置。最近从其志那弄了个高级点的，据说源头是万哥那个的 -_-|| 。来看下吧。

##  用法

在分析脚本之前先说说用法吧，很简单：

<pre>
/* 切换编译环境 */
sw xx

/* autogen */
cfg
make
make install

/* cmake */
cm
make
make install
</pre>

然后终端显示会相应的变成 xx 。这就表示已经设置成 xx 的编译环境了。

## sw.sh

<pre config="brush:bash;toolbar:false;">
#! /bin/bash

#check weather exe
if [ "$1" = "" -o $# -ne 1 ]; then
echo "Usage: $0 <target_name>"
echo "  target_name: build target name"
exit 1
else
cd /home/mingming/bin/inc

INCFILE="$1.inc"

#check file
INCFILE=`find . -iname $INCFILE`
if [ "$INCFILE" = "" ]; then
echo "Can't find the path: $INCFILE."
exit 1
fi
echo "Found match path: $INCFILE"
echo "Setting $1 env ..."
echo 

export TARGET_NAME=$1

#set env
export PS1="\[\033[01;35m\]$1\[\033[01;34m\] \w \$\[\033[00m\] "
source $INCFILE
fi

#/bin/bash
</pre>

这个脚本很短，也比较容易理解。它会把输入的第一个参数（就是 sw xx 的 xx），作为这个编译环境的名字。然后去 /home/mingming/bin/inc 下去找对应的环境配置文件： xx.inc 。这个路径可以自己指定。如果找不到就报错。找到后就把这个名字 export 一下，后面的配置文件会用到的。然后把终端显示相应的改成编译环境的名字。最后读入配置文件的信息。

## inc

我的配置文件放在 /home/mingming/bin/inc 下，一般新建一个编译环境，就只要新写一个配置文件就可以了。现在来看看一个比较简单的 MiniGUI PC 上的线程版的配置文件吧：

<pre config="brush:bash;toolbar:false;">
#!/bin/sh

#cfg
export minigui_configure_flags="\
--with-ttfsupport=ft2 \
--enable-ttfcache \
--enable-jpgsupport \
--enable-pngsupport \
--disable-dlcustomial \
--disable-splash \
--disable-screensaver"

export cmake_minigui_configure_flags="\
-Dwith_osname:STRING=linux \
-Dimage_pngsupport:BOOL=ON \
-Dimage_jpegsupport:BOOL=ON \
-Dwith_fontttfsupport:STRING=ft2 \
-Dfont_ttfenablecache:BOOL=ON \
-Dlicense_splash:BOOL=OFF \
-Dlicense_screensaver:BOOL=OFF \
-Dial_dlcustom:BOOL=OFF"

export mdolphin_configure_flags="\
--enable-focusring_tv"


export minigui_src="${ss}/minigui/rel-3-0"
export mgplus_src="${ss}/mgplus/rel-3-0"
export mgeff_src="${ss}/mgeff/rel-0-8"
export gvfb_src="${ss}/gvfb/trunk"


#env
export PREFIX="${tt}/${TARGET_NAME}"
export pp=$PREFIX
export BUILDPATH="${bb}/${TARGET_NAME}"

export MG_RES_PATH=$ss/res-trunk
export MG_CFG_PATH=$PREFIX/etc
export MG_CFG_FILE=$PREFIX/etc/MiniGUI.cfg
export libs_base="-L$PREFIX/lib -L/usr/local/lib -Wl,-rpath-link=$PREFIX/lib"
export extra_libs="-lpthread -lfreetype -lpng -ljpeg"
export CFLAGS="-g -Wall -I$PREFIX/include -I/usr/local/include"
export CPPFLAGS=$CFLAGS
export CXXFLAGS=$CFLAGS
export LDFLAGS="${libs_base}"
export PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig
export PATH=$extra_path:$PREFIX/bin:/home/mingming/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export LD_LIBRARY_PATH="${tt}/${TARGET_NAME}/lib":$LD_LIBRARY_PATH

export CC=gcc
export AR=ar
export RANLIB=ranlib
export HOST=


cd $BUILDPATH
</pre>

这个文件其实很一目了然了。开始的那个几个 flags 是相应设置 MiniGUI configure, MiniGUI cmake 的。如果有别的库需要编译的话，可以多加。后面就是设置安装路径（PREFIX）、CFLAGS、CXXFLAGS、LDFLAGS 等。这里注意下后面几个编译变量：

* PKG_CONFIG_PATH：这个是设置 pkgconfig 的文件路径，编译一些库必须要指定这个才能正确编译。
* PATH：这个是指定一些库的配置脚本的，例如 freetype-config 等，有些库如果要链接另外一个库的话，必须要有这些配置脚本才能正确编译。注意会优先找放在前面的
* LD_LIBRARY_PATH：这个是设置链接时候的路径。也是优先找放在前面的。不设置这个的话，很容易老去链接 /usr/lib 或是 /usr/local/lib 下的，交叉编译特别有用。

## cfg

<pre config="brush:bash;toolbar:false;">
#!/bin/sh

#. common_flags.inc

echo $PWD |grep minigui > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$minigui_configure_flags $common_minigui_configure_flags"
fi

echo $PWD |grep mgplus > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$mgplus_configure_flags $common_mgplus_configure_flags"
fi

echo $PWD |grep mgeff > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$mgeff_configure_flags $common_mgeff_configure_flags"
fi

echo $PWD |grep mgutils > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$mgutils_configure_flags $common_mgutils_configure_flags"
fi

echo $PWD |grep mgncs > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$ncs_configure_flags $common_mgncs_configure_flags"
fi

echo $PWD |grep mginit > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$mginit_configure_flags $common_mginit_configure_flags"
fi

echo $PWD |grep core > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$mdolphin_configure_flags $common_mdolphin_core_configure_flags"
fi

echo $PWD |grep mdtv > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$mdtv_configure_flags $common_mdtv_configure_flags"
fi

echo $PWD |grep curl > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$curl_configure_flags $common_curl_configure_flags"
fi

echo $PWD |grep xml2 > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$xml2_configure_flags $common_libxml_configure_flags"
fi

echo $PWD |grep xslt > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$xslt_configure_flags $common_libxslt_configure_flags"
fi

echo $PWD |grep dfb > /dev/null
if [ $? -eq 0 ]; then
FLAGS="$dfb_configure_flags $common_dfb_core_configure_flags"
fi

if [ ! -e configure ]; then
echo autogen.sh
./autogen.sh
fi

cmd="./configure $FLAGS --prefix=$PREFIX $common_configure_flags $*"

echo $cmd
$cmd
</pre>

这个配合前面的 inc 文件也很简单了。它就是把之前 inc 文件里 export 的那些配置变量用到不同的库的 configure 上（如果没有 configure 的话，会先执行 autogen.sh）。不过这里可以看到局限性：就是这个脚本是根据当前的文件名判断应该使用哪个配置变量的。例如你的当前编译目录是： /home/mingming/build/pc-ths/minigui 的话，它就会使用 minigui_configure_flags 。所以编译库的文件夹名字不能随便乱取，要和这个脚本的对应。如果要新加的库的话，在这个脚本和加上那几句就可以，还有 inc 文件里也要加上相应的配置变量（不过如果你需要特别的配置的话，可以不加）。这个是简单的，复杂的可以参看附件里交叉编译 ST7167 的 ^_^。

## cm

<pre config="brush:bash;toolbar:false;">
#!/bin/sh

#. common_flags.inc

echo $PWD |grep cmake_minigui > /dev/null
if [ $? -eq 0 ]; then
SOURCE="$minigui_src"
FLAGS="$cmake_minigui_configure_flags $common_cmake_minigui_configure_flags"
fi

echo $PWD |grep cmake_mgplus > /dev/null
if [ $? -eq 0 ]; then
SOURCE="$mgplus_src"
FLAGS="$cmake_mgplus_configure_flags $common_cmake_mgplus_configure_flags"
fi

echo $PWD |grep cmake_mgeff > /dev/null
if [ $? -eq 0 ]; then
SOURCE="$mgeff_src"
FLAGS="$cmake_mgeff_configure_flags $common_cmake_mgeff_configure_flags"
fi

echo $PWD |grep cmake_mgutils > /dev/null
if [ $? -eq 0 ]; then
SOURCE="$mgutils_src"
FLAGS="$cmake_mgutils_configure_flags $common_cmake_mgutils_configure_flags"
fi

echo $PWD |grep cmake_mgncs > /dev/null
if [ $? -eq 0 ]; then
SOURCE="$mgncs_src"
FLAGS="$cmake_mgncs_configure_flags $common_cmake_mgncs_configure_flags"
fi

echo $PWD |grep cmake_gvfb > /dev/null
if [ $? -eq 0 ]; then
SOURCE="$gvfb_src"
FLAGS="$cmake_gvfb_configure_flags $common_cmake_gvfb_configure_flags"
fi


cmd="cmake $SOURCE -DCMAKE_INSTALL_PREFIX=$PREFIX $FLAGS $common_cmake_configure_flags $*"

echo $cmd
$cmd
</pre>

这个是和 cfg 类似的，只不过是 cmake 的而已。也就是说如果你的库同时支持 autogen 和 cmake，你 inc 文件里就要写2份配置。

## 注意事项

原来我的 sw.sh 是 sw 的，然后是放在 ～/bin/ 下的，不过后来发现这样，终端显示的名字改变不了。好像是改变后马上又变会 ~/.bashrc 里设置的默认的了。晕～～后来把 sw 改成 sw.sh ，然后放到 ~/build/ 下才算可以正常修改终端的显示名字。真是奇怪，估计和 .bashrc 里哪个冲突了吧，不过我暂时还发现是哪个。

## 附件

[set.zip](http://pan.baidu.com/s/1hq6BqWO)
