title: (转) Java 的模版和 C++ 模版的区别
date: 2015-01-31 19:38:16
categories: [Basics Knowledge]
tags: [basics]
---

java 中的模版其实应该叫泛型。 泛型本质上是提供类型的"类型参数"，它们也被称为参数化类型（parameterized type）或参量多态（parametric polymorphism）。其实泛型思想并不是 Java 最先引入的，C++ 中的模板就是一个运用泛型的例子。泛型程序设计划分为三个熟练地级别。

* java 中没有 template 的关键字，c++ 中有。

* java 语言中的泛型不能接受基本类型作为类型参数――它只能接受引用类型。这意味着可以定义 `List<Integer>`，但是不可以定义 `List<int>`。

* 在java中，尖括号通常放在方法名前，而c++则是放在方法名后，c++的方式容易产生歧义，例如`g(f<a,b>(c))`，这个则有两种解释，一种是f的泛型调用，c为参数，a，b为泛型参数。另一种解释，则是，g调用，两个bool类型的参数。

* 在 C++ 模板中，编译器使用提供的类型参数来扩充模板，因此，为 `List<A>` 生成的 C++ 代码不同于为 `List<B>` 生成的代码，`List<A>` 和 `List<B>` 实际上是两个不同的类。而 Java 中的泛型则以不同的方式实现，编译器仅仅对这些类型参数进行擦除和替换。类型 `ArrayList<Integer>` 和 `ArrayList<String>` 的对象共享相同的类，并且只存在一个 ArrayList 类。因此在c++中存在为每个模板的实例化产生不同的类型，这一现象被称为“模板代码膨胀”，而java则不存在这个问题的困扰。java中虚拟机中没有泛型，只有基本类型和类类型，泛型会被擦除，一般会修改为Object，如果有限制，例如 T extends Comparable,则会被修改为Comparable。而在C++中不能对模板参数的类型加以限制，如果程序员用一个不适当的类型实例化一个模板，将会在模板代码中报告一个错误信息。


[原始出处](http://blog.163.com/maravilla_evol/blog/static/139564699201061694833152/ "原始出处")


