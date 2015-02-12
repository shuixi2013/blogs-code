title: ubuntu libtool 错误
date: 2015-01-19 10:01:16
updated: 2015-01-19 10:01:16
categories: [Linux]
tags: [linux]
---

使用Ubuntu 8.04以上版本的同事在编译咱们的产品的时候可能会遇到类似如下错误：

<pre config="brush:bash;toolbar:false;">
../libtool: line 841: X--tag=CC: command not found
../libtool: line 874: libtool: ignoring unknown tag : command not found
../libtool: line 841: X--mode=link: command not found
../libtool: line 1008: *** Warning: inferring the mode of operation is deprecated.: command not found
../libtool: line 1009: *** Future versions of Libtool will require --mode=MODE be specified.: command not found
../libtool: line 2253: X-g: command not found
../libtool: line 2253: X-Wall: command not found
../libtool: line 2253: X-I/home/cos/target/pc-ths/include: No such file or directory
../libtool: line 2253: X-I/home/cos/build/pc-ths/include: No such file or directory
../libtool: line 2253: X-Wall: command not found
../libtool: line 2253: X-Wstrict-prototypes: command not found
../libtool: line 2253: X-pipe: command not found
../libtool: line 1967: X-L/home/cos/target/pc-ths/lib: No such file or directory
../libtool: line 1967: X-L/home/cos/build/pc-ths/lib: No such file or directory
../libtool: line 2216: X-Wl,-rpath-link=/home/cos/build/pc-ths/lib: No such file or directory
</pre>

我们知道有一种解决方法是在运行autogen.sh和configure后，修改configure生成的libtool文件，将其中的ECHO修改为echo，然后make;make install即可。

在网上找到另外一种方法， 就是用autoreconf代替autogen.sh，方法如下:
<pre>
autoreconf -i
configure
make
make install
</pre>

[参考网址](http://www.gossamer-threads.com/lists/clamav/devel/41623)

