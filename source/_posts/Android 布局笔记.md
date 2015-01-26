title: Android 布局笔记
date: 2015-01-25 23:15:16
categories: [Android Development]
tags: [android]
---

说实在我个人觉得 Android 搞这套玩意比 MiniGUI 麻烦多了。以前没怎么系统的研究、学习，遇到了不少问题。现在记一下。

## onMeasure

实现 onMeasure 方法基本需要完成下面三个方面的事情（最终结果是你自己写相应代码得出测量值并调用view的一个方法进行设置，告诉给你的view安排位置大小的父容器你要多大的空间）。

* 传递进来的参数：widthMeasureSpec 和 heightMeasureSpec 是你对你应该得出来的测量值的限制。

The overidden onMeasure() method is called with width and height measure specifications(widthMeasureSpec and heightMeasureSpec parameters,both are integer codes representing dimensions) which should be treated as requirements for the restrictions on the width and height measurements you should produce.

* 你在 onMeasure 计算出来设置的 width 和 height 将被用来渲染组件。应当尽量在传递进来的 width 和 height 声明之间。虽然你也可以选择你设置的尺寸超过传递进来的声明。但是这样的话，父容器可以选择，如 clipping, scrolling, 或者抛出异常，或者（也许是用新的声明参数）再次调用 onMeasure 。

Your component's onMeasure() method should calculate a measurement width and height which will be required to render the component.it should try to stay within the specified passed in.although it can choose to exceed them(in this case,the parent can choose what to do,including clipping,scrolling,throwing an excption,or asking the onMeasure to try again,perhaps with different measurement specifications).

* 一但 width 和 height 计算好了，就应该调用 View.setMeasuredDimension(int width,int height) 方法，否则将导致抛出异常。

Once the width and height are calculated,the setMeasureDimension(int width,int height) method must be called with the calculated measurements.Failure to do this will result in an exceptiion being thrown.

### 例子分析

在 Android 提提供的一个自定义 View 示例中（在 API Demos 中的 view/LabelView）可以看到一个重写 onMeasure 方法的实例，也比较好理解：

```java
    /**
     * @see android.view.View#measure(int, int)
     */
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(measureWidth(widthMeasureSpec),
                measureHeight(heightMeasureSpec));
    }

    /**
     * Determines the width of this view
     * @param measureSpec A measureSpec packed into an int
     * @return The width of the view, honoring constraints from measureSpec
     */
    private int measureWidth(int measureSpec) {
        int result = 0;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        if (specMode == MeasureSpec.EXACTLY) {
            // We were told how big to be
            result = specSize;
        } else {
            // Measure the text
            result = (int) mTextPaint.measureText(mText) + getPaddingLeft()
                    + getPaddingRight();
            if (specMode == MeasureSpec.AT_MOST) {
                // Respect AT_MOST value if that was what is called for by measureSpec
                result = Math.min(result, specSize);
            }
        }

        return result;
    }

    /**
     * Determines the height of this view
     * @param measureSpec A measureSpec packed into an int
     * @return The height of the view, honoring constraints from measureSpec
     */
    private int measureHeight(int measureSpec) {
        int result = 0;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        mAscent = (int) mTextPaint.ascent();
        if (specMode == MeasureSpec.EXACTLY) {
            // We were told how big to be
            result = specSize;
        } else {
            // Measure the text (beware: ascent is a negative number)
            result = (int) (-mAscent + mTextPaint.descent()) + getPaddingTop()
                    + getPaddingBottom();
            if (specMode == MeasureSpec.AT_MOST) {
                // Respect AT_MOST value if that was what is called for by measureSpec
                result = Math.min(result, specSize);
            }
        }
        return result;
    }
```

直接看 measureWidth 首先看到的是参数，分别代表宽度和高度的 MeasureSpec。 Android2.2 文档中对于 MeasureSpec 中的说明是：

1. 一个 MeasureSpec 封装了从父容器传递给子容器的布局需求。
2. 每一个MeasureSpec代表了一个宽度，或者高度的说明。
3. 一个MeasureSpec是一个大小跟模式的组合值。一共有三种模式：

A MeasureSpec encapsulates the layout requirements passed from parent to child Each MeasureSpec represents a requirement for either the width or the height.A MeasureSpec is compsized of a size and a mode.There are three possible modes:

* **1. UPSPECIFIED：**
父容器对于子容器没有任何限制，子容器想要多大就多大。

UNSPECIFIED The parent has not imposed any constraint on the child.It can be whatever size it wants.

* **2.EXACTLY：**
父容器已经为子容器设置了尺寸，子容器应当服从这些边界，不论子容器想要多大的空间。

EXACTLY The parent has determined and exact size for the child.The child is going to be given those bounds regardless of how big it wants to be.

* **3.AT_MOST：**
子容器可以是声明大小内的任意大小。

AT_MOST The child can be as large as it wants up to the specified size.

### MeasureSpec
MeasureSpec 是 View 类下的静态公开类。MeasureSpec 中的值作为一个整型是为了减少对象的分配开支。此类用于将 size 和 mode 打包或者解包为一个整型。

MeasureSpecs are implemented as ints to reduce object allocation.This class is provided to pack and unpack the size,mode tuple into the int.

我比较好奇的是怎么样将两个值打包到一个int中，又如何解包。MeasureSpec 类代码如下（注释已经被我删除了，因为在上面说明了）：

```java
public static class MeasureSpec {
private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
    	 
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;
    public static final int EXACTLY     = 1 << MODE_SHIFT;
    public static final int AT_MOST     = 2 << MODE_SHIFT;
     
    public static int makeMeasureSpec(int size, int mode) {
        return size + mode;
    }

    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }
    
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
}
```

我无聊的将他们的十进制值打印出来了：

<pre>
mode_shift=30, 
mode_mask=-1073741824, 
UNSPECIFIED=0, 
EXACTLY=1073741824, 
AT_MOST=-2147483648
</pre>

然后觉得也应该将他们的二进制值打印出来，如下：

<pre>
mode_shift=11110, // 30
mode_mask=11000000000000000000000000000000,
UNSPECIFIED=0, 
EXACTLY=1000000000000000000000000000000, 
AT_MOST=10000000000000000000000000000000
</pre>

MODE_MASK  = 0x3 << MODE_SHIFT ：也就是说 MODE_MASK 是由11左移30位得到的。因为 Java 用补码表示数值.最后得到的值最高位是1所以就是负数了。对于上面的数值我们应该这样想,不要把0x3看成3而要看成二进制的11，而把 MODE_SHIFF 就看成30。那为什么是二进制的11呢？因为只有三各模式,如果有四种模式就是111了因为111三个位才可以有四种组合对吧。我们这样来看：

<pre>
UNSPECIFIED=00000000000000000000000000000000, 
EXACTLY=01000000000000000000000000000000, 
AT_MOST=10000000000000000000000000000000
</pre>

也就是说 0, 1, 2 对应 00, 01, 10。当跟11相与时 00 &11 还是得到 00, 11&01 -> 01, 10&11 -> 10。

## onLayout
onLayout 的调用在 onMeasure 之后，主要是决定子 View 如何摆放的。 需要注意的是，如果自定义的 View 只重载了 onLayout 而没重载 onMeasure 的话，就会导致子 View 布局不正确（如果 View 自己都没有在 onMeasure 里设置自己的话，自己的布局都会不对，例如 View 不是继承现有的 View）。要想子 View 布局正确，需要在 onMeasuer 里调用子 View 的 measure 方法设置子 View 的大小，然后在 onLayout 里对子 View 进行布局，才会正确显示。 

## 一些系统属性分析

### padding 和 margin 的区别
在 android 默认 view 中 padding (android:paddingTop) 是用于 view 外部的边界距离（例如说 layout 中子 view 之间的）。而 margin (android:layout_marginLeft) 是 view 内部的边界距离（例如说 Button 中字体的距边界的距离）。



