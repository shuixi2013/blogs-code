title: Java 备忘
date: 2015-01-25 18:48:16
tags: [android]
---

Java 小菜鸟的备忘。

## 方法和成员变量的默认权限
在类里面的方法，如果不加修饰权限关键字（public, protected, private 等），那默认就是包权限（package）。同一个包里的可以在类外面访问。不过个人感觉在编码中前面加上 `/* package */` 会更好。

## 静态代码段
在类里面，成员变量可以定义为 static，表示所有该类的实例都共享这一个变量。同时如果只想初始化一次这个 static 变量的话，就可以使用静态代码段，入下面所示：

```java
class test {
    private static final Canvas mCanvas = new Canvas();
	
    static {
        mCanvas.setDrawFilter(new PaintFlagsDrawFilter(Paint.DITHER_FLAG,
                Paint.FILTER_BITMAP_FLAG));
    }
}
```

