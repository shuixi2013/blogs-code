title: GDB 命令备忘
date: 2015-01-11 00:25:16
tags: linux
---

一些常用的 gdb 使用命令备忘。help 可以查看命令，help xx 可以查看具体命令的所有参数。

## 编译带调试符号的程序
要想用 gdb 调试，编译的时候要带上 debug 符号，就是用 gcc 编译的时候带上 -g 参数编译（bu要带 -o 开启优化模式，这个会把所有的调试符号优化掉）。

## file 
file app，装载要调试的应用程序。

## list （l）
list fun，可查看对应函数 fun 的代码。可以接很多参数，例如行号等。函数名是可以使用 tab 键补全的。

## breakpoint (b)
break fun，在函数 fun 出设置断点。也是可以接很多参数的，例如说行号等。

* b fun if condition，设置条件断点，只有当 condition 为 true 时才中断。

* i breakpoint（b），显示当前设置的断点。

* delete breakpoint 1，删除1号断点，编号可以使用 i b 查看，如果不加参数，则会删除所有断点。

* disable breakpoint 1，禁止1号断点。

* enable breakpoint 1，开启1号断点。

## watch
watch condition (watch i>99)，监视变量的变化达到条件时停止程序执行。注意：监视点的设定不依赖于断点的位置，但是与变量的作用域有关。也就是说，要设置监视点必须在程序运行时才可设置。

还有一个作用是，硬件写断点。这种断点和普通的 break 有点不同，需要每次挂载 gdb 后，先利用普通的 break 让程序停下来，然后查看出你要查看变量的地址（用p）。然后再用 watch 命令设置。然后每次程序重新运行都要重新设置，因此每次变量地址的都不一样。

## 控制命令
* run (r): 运行程序。

* next (n): 下一步，跳过函数。

* setp (s): 下一步，进入函数。

* continue (c): 继续执行。

## info (i)
显示某些内容。

* i breakpoints(b)，显示当前断点。

* i variables，显示所有全局变量和静态变量。

* i functions，显示所有函数名字。

* i local，显示当前函数中的变量。

* i file，显示调试文件的信息。

* i prog，显示调试程序的执行状态

## print (p)
p var，显示变量 var 的值，或者 p exp，可以显示该表达式的值。这样可以查看几乎任何变量的值。只要你确定改变量是什么类型的指针，可以直接转化为该类型就行了，例如 p *(PDC)hdc 就可以查看这个结构的值了。还可以通过添加参数来设置输出格式：

<pre>
/x 按十六进制格式显示变量
/d 按十进制格式显示变量
/u 按十六进制格式显示无符号整型
/o 按八进制格式显示变量
/t 按二进制格式显示变量
/a 按十六进制格式显示变量
/c 按字符格式显示变量
/f 按浮点数格式显示变量
</pre>

其实它还有一个功能就是执行函数。调试 MiniGUI 的时候，最典型的用法用法就是可以将你想查看的一些 memdc 中的图像信息输出到屏幕上进行检查。方法是调用 BitBlt ，注意这种情况一些宏定义的变量无法直接使用，而是要填入真正的数值，这些可以从代码里面去差。例如先把屏幕一块地方填充成红色，然后再把 memdc 中的内容输入到屏幕的这个地方：

<pre config="brush:bash;toolbar:false;">
// SetBrushColor(hdc, color) 的宏定义是 SetDCAttr(hdc, DC_ATTR_BRUSH_COLOR, color)
// DC_ATTR_BRUSH_COLOR 值就是2
// HDC_SCREEN 的值就是0
// 如果自己知道 rgb 对应的 pixel 值的话，也可以不用 RGB2Pixel
p SetDCAttr(0, 2, RGB2Pixel(0, 255, 0, 0))

p FillBox(0, 400, 0, 360, 480)
 
p BitBlt(memdc, 0, 0, 0, 0, 0, 400, 0, 0)
</pre>

## x 
x /nfu <addr>，查看addr地址处的内存信息。n是显示多少字节，后面的显示的格式，和 p 命令是一样的。

## thread 
用法：thread xx。切换当前活动线程。用于调试多线程程序。xx 为线程号，用 info thread（th）查看，每个线程的第一个数字就是线程号。 

## backtrace (bt)
backtrace [-n] [n] 显示程序中的当前位置和表示如何到达当前位置的栈跟踪。
-n：表示只打印栈底上n层的栈信息
 n：表示只打印栈顶上n层的栈信息
不加参数，表示打印所有栈信息。

## call 
call func_name，调用和执行一个函数。例如 call print("abcd\n")。

## 查看 coredump 文件
首先要让程序在崩溃的时候产生 coredump 文件。输入 ulimit -c unlimited 命令（注意这个只对一个终端有效）。然后在程序崩溃的时候，就会产生 core.xx 的文件。使用 gdb app core.xx 命令查看（app 就是产生 core.xx 的程序）。然后就和普通的 gdb 用法一样了，用 bt 查看崩溃时的堆栈信息啊，但是就是不能执行而已。

## disassemble
对当前的执行到的代码反汇编。