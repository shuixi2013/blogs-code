title: 使用 pkg-config
date: 2015-01-19 09:51:16
updated: 2015-01-19 09:51:16
categories: [Linux]
tags: [linux, shell]
---

有的使用了共享库的程序，在编译和连接时都很顺利，但是在运行时却发生了找不到共享库的问题，其原因就是库的搜索路径没有设置，或者设置不正确。一般来说，如果库的头文件不在 /usr/include 目录中，那么在编译的时候需要用 -I 参数指定其路径。由于同一个库在不同系统上可能位于不同的目录下，用户安装库的时候也可以将库安装在不同的目录下，所以即使使用同一个库，由于库的路径的 不同，造成了用 -I 参数指定的头文件的路径也可能不同，其结果就是造成了编译命令界面的不统一。如果使用 -L 参数，也会造成连接界面的不统一。编译和连接界面不统一会为库的使用带来麻烦。

为了解决编译和连接界面不统一的问题，人们找到了一些解决办法。其基本思想就是：事先把库的位置信息等保存起来，需要的时候再通过特定的工具将其中 有用的信息提取出来供编译和连接使用。这样，就可以做到编译和连接界面的一致性。

## pkg-config

其中，目前最为常用的库信息提取工具就是下面介绍的 pkg-config。pkg-config 是通过库提供的一个 .pc 文件获得库的各种必要信息的，包括版本信息、编译和连接需要的参数等。这些信息可以通过 pkg-config 提供的参数单独提取出来直接供编译器和连接器使用。

在默认情况下，每个支持 pkg-config 的库对应的 .pc 文件在安装后都位于安装目录中的 lib/pkgconfig 目录下。例如，我们在上面已经将 gstreamer 安装在 /usr/lib/gstreamer 目录下了，那么这个 gstreamer 库对应的 .pc 文件是 /usr/lib/pkgconfig 目录下一个叫 gtreamer-0.10.pc 的文件。

<pre config="brush:bash;toolbar:false;">

prefix=/usr
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include/gstreamer-0.10
toolsdir=${exec_prefix}/bin
pluginsdir=${exec_prefix}/lib/gstreamer-0.10
gstcontrol_libs=-lgstcontrol-0.10

Name: GStreamer
Description: Streaming media framework
Requires: glib-2.0, gobject-2.0, gmodule-no-export-2.0, gthread-2.0, libxml-2.0
Version: 0.10.18
Libs: -L${libdir} -lgstreamer-0.10
Cflags: -I${includedir}

</pre>

大概可以看得出是怎么回事了。

## 使用方法

使用 pkg-config 的 --cflags 参数可以给出在编译时所需要的选项，而 --libs 参数可以给出链接时的选项。例如，假设一个 sample.c 的程序用到了 gstreamer 库，就可以这样编译：

<pre config="brush:bash;toolbar:false;">

//先这样编译（注意是键盘上数字键1旁边的`，不是单引号'）：
$ gcc -c `pkg-config --cflags gstreamer-0.10` sample.c

//然后这样链接：
$ gcc sample.o -o sample `pkg-config --libs gstreamer-0.10`

//或者上面两步也可以合并为以下一步：
$ gcc sample.c -o sample `pkg-config --cflags --libs gstreamer-0.10`

</pre>

可以看到：由于使用了 pkg-config 工具来获得库的选项，所以不论库安装在什么目录下，都可以使用相同的编译和连接命令，带来了编译和连接界面的统一。使用 pkg-config 工具提取库的编译和连接参数有两个基本的前提：库本身在安装的时候必须提供一个相应的 .pc 文件。不这样做的库说明不支持 pkg-config 工具的使用。

pkg-config 必须知道要到哪里去寻找此 .pc 文件。一般来所某些库会指定一个环境变量来寻找 .pc 文件，例如gstreamer就是哟你 PKG_CONFIG_PATH 来寻找的。可以用 export PKG_CONFIG_PATH=XX 设置 .pc 文件的路径。

## 我的makefile

这样一来我之前用的makefile模板就可以改成这样啦：

<pre config="brush:bash;toolbar:false;">
#This Makefile is created by mingming-killer

SRCS := $(wildcard *.c)
DEPS := $(SRCS:.c=.d)
OBJS := $(SRCS:.c=.o)

APP = app


CC = gcc
LDFLAGS = `pkg-config --libs gstreamer-0.10` -L$(LIB_PATH) $(LIBS)
CFLAGS = `pkg-config --cflags gstreamer-0.10` -I$(INCLUDE_PATH)
LIBS = -lpthread -lminigui_ths -lm -ljpeg -lpng
LIB_PATH = /usr/local/lib
INCLUDE_PATH = /usr/local/include


# if "make clean" do not create the depend file
ifneq ($(MAKECMDGOALS), clean)

%d: %c
	$(CC) -MM $< > $@

endif



$(APP): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)


-include $(DEPS)




.PHONY: clean 
clean:
	-rm -f debug $(APP) $(wildcard *.o ./tmp/*.d *.d *~)
	@echo "it is cleaning."
</pre>

autoconf、automake的工具暂时还不会用，又不像mg-sample那样可以替换里面的文件来直接使用，而且又只是小程序，所以还是用个makefile来搞定吧 ^_^ 。

