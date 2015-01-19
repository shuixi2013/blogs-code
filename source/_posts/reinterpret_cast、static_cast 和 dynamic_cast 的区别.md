title: reinterpret_cast、static_cast 和 dynamic_cast 的区别
date: 2015-01-19 10:38:16
tags: [basics]
---

reinterpret_cast 可以转换任意一个32bit整数，包括所有的指针和整数。可以把任何整数转成指针，也可以把任何指针转成整数，以及把指针转化为任意类型的指针，威力最为强大！但不能将非32bit的实例转成指针。总之，只要是32bit的东东，怎么转都行！

static_cast 和 dynamic_cast 可以执行指针到指针的转换，或实例本身到实例本身的转换，但不能在实例和指针之间转换。static_cast 只能提供编译时的类型安全，而 dynamic_cast 可以提供运行时类型安全。举个例子：

<pre> 
class a;
class b:a;
class c 
</pre>

上面三个类a是基类，b继承a，c和ab没有关系。有一个函数 void function(a& a); 现在有一个对象是b的实例b，一个c的实例c。function(static_cast<a&>(b)) 可以通过而 function(static<a&>(c)) 不能通过编译，因为在编译的时候编译器已经知道c和a的类型不符，因此 static_cast 可以保证安全。 

下面我们骗一下编译器，先把c转成类型a 

<pre>
b& ref_b = reinterpret_cast<b&>c;
</pre>
 
然后 function(static_cast<a&>(ref_b)) 就通过了！因为从编译器的角度来看，在编译时并不能知道ref_b实际上是c！ 而 function(dynamic_cast<a&>(ref_b)) 编译时也能过，但在运行时就失败了，因为 dynamic_cast 在运行时检查了 ref_b 的实际类型，这样怎么也骗不过去了。
 
在应用多态编程时，当我们无法确定传过来的对象的实际类型时使用 dynamic_cast，如果能保证对象的实际类型，用 static_cast 就可以了。至于 reinterpret_cast，我很喜欢，很象c语言那样的暴力转换。

总结一下：

**dynamic_cast**: 动态类型转换，一般用在父类和子类指针或应用的互相转化。
**static_cast**: 静态类型转换，一般是普通数据类型(如: int m=static_cast<int>(3.14)) 
**reinterpret_cast**: 重新解释类型转换，很像c的一般类型转换操作 
**const_cast**: 常量类型转换，是把cosnt或volatile属性去掉 

