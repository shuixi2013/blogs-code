title: Android 坑爹大全 —— SeekBar
date: 2015-01-26 23:48:16
categories: [Android Development]
tags: [android]
---

一般用 SeekBar 除了 google 自己那一票 app 以及系统，开发者几乎都不会直接用原来的 SeekBar 的图的，都要自己换图。这里就所说 SeekBar 换图的事。

## 结构
首先来看看 SeekBar 的构成：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/suck-problem-seek-bar/1.jpeg)

SeekBar 由一个进度条和上面的滑块构成。一般换图就换这2个东西就 OK 了。一般是要准备3张图片：进度条2张，一张背景（background）、一张滑动的进度（progress）、一张是滑块（thumb）。

拿个例子说事， UI 说界面的亮度调节的视觉是这样的：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/suck-problem-seek-bar/2.jpeg)

然后给了3张图，一般换图么， SeekBar 继承关系是： SeekBar ---> AbsSeekBar --> ProgressBar 。 换个 progressDrawable（ProgressBar的） 和 thumb（AbsSeekBar的） 就行了。

```xml
    <SeekBar
        android:layout_width="300dp"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:progressDrawable="@drawable/seek_drawable"
        android:thumb="@drawable/seek_thumb"
        />
```

thumb 的 drawable 直接是滑块的 png 就行。progressDrawable 如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" >
    <item 
        android:id="@android:id/background" 
        android:drawable="@drawable/seek_progress_bk"
        >
    </item>
    <item android:id="@android:id/progress" >
        <clip android:drawable="@drawable/seek_progress" >
        </clip>
     </item>
</layer-list>
```

progressDrawable 可以用 LayerDrawable 同时指定进度条的背景（background）和进度（progress），progress 的 clip 的意思是说：这个要用这个图片的一部门来表示进度，而不是用图片的全部来表示，想想看进度条的表示方式，就是这样的啦。这个 clip 还可以设置方向，进度条可以是水平的也可以是竖直的。

然后坑就来了。一般 SeekBar 的滑块都会比进度条大，这里 UI 给的是 25x29 ，那个进度是 9-patch 的，20x6。我原来以为 SeekBar 的大小应该是 300 x 29，滑块是 25x29，然后那个进度条高度就是 6，居中显示。但是出来的结果确是滑块的大小确实 25x29，但是进度条却是 13 的高度（居中倒是居中），感觉肥了不少。

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/suck-problem-seek-bar/3.jpeg)

刚开始以为 ProgressBar 可以设啥 padding 之类的东西，但是不找到，然后尝试把进度体的图改称和 thumb 的一样高（png 的用透明填充），但是发现还是达不到预期效果。无奈只好去翻代码了。 

## 坑爹的设计

先是在 ProgressBar.java 中发现 progressDrawable 设置的地方：

```java
        Drawable drawable = a.getDrawable(R.styleable.ProgressBar_progressDrawable);
        if (drawable != null) {
            drawable = tileify(drawable, false);
            // Calling this method can set mMaxHeight, make sure the corresponding
            // XML attribute for mMaxHeight is read after calling this method
            setProgressDrawable(drawable);
        }    
```

这段在 ProgressBar 的构造函数中，发现 xml 里指定的 progressDrawable 其实调用 setProgressDrawable 来设置的：

```java
    public void setProgressDrawable(Drawable d) {
        boolean needUpdate;
        if (mProgressDrawable != null && d != mProgressDrawable) {
            mProgressDrawable.setCallback(null);
            needUpdate = true;
        } else {
            needUpdate = false;
        }
                
        if (d != null) {
            d.setCallback(this);
            if (canResolveLayoutDirection()) {
                d.setLayoutDirection(getLayoutDirection());
            }

            // Make sure the ProgressBar is always tall enough
            int drawableHeight = d.getMinimumHeight();
            if (mMaxHeight < drawableHeight) {
                mMaxHeight = drawableHeight;
                requestLayout();
            }
        }
        mProgressDrawable = d;
        if (!mIndeterminate) {
            mCurrentDrawable = d;
            postInvalidate();
        }

        if (needUpdate) {
            updateDrawableBounds(getWidth(), getHeight());
            updateDrawableState();
            doRefreshProgress(R.id.progress, mProgress, false, false);
            doRefreshProgress(R.id.secondaryProgress, mSecondaryProgress, false, false);
        }
    }
```

然后 setProgressDrawable 是把设置的 progressDrawable 保存在了 mProgressDrawable 的成员变量中。然后通过搜索发现根据 ProgressBar 的样式 mProgressDrawable 为被选为 mCurrentDrawable：

```java
    public synchronized void setIndeterminate(boolean indeterminate) {
        if ((!mOnlyIndeterminate || !mIndeterminate) && indeterminate != mIndeterminate) {
            mIndeterminate = indeterminate;
    
            if (indeterminate) {
                // swap between indeterminate and regular backgrounds
                mCurrentDrawable = mIndeterminateDrawable;
                startAnimation();
            } else {
                mCurrentDrawable = mProgressDrawable;
                stopAnimation();
            }
        }
    } 
```

然后去 onDraw 里面看看，原来 mCurrentDrawable 就是画进度条的东西：

```java
    @Override 
    protected synchronized void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        
        Drawable d = mCurrentDrawable;
        if (d != null) {
            // Translate canvas so a indeterminate circular progress bar with padding
            // rotates properly in its animation
            canvas.save();
            if(isLayoutRtl()) { 
                canvas.translate(getWidth() - mPaddingRight, mPaddingTop);
                canvas.scale(-1.0f, 1.0f);
            } else {
                canvas.translate(mPaddingLeft, mPaddingTop);
            }
            long time = getDrawingTime();
            if (mHasAnimation) {
                mAnimation.getTransformation(time, mTransformation);
                float scale = mTransformation.getAlpha();
                try {
                    mInDrawing = true;
                    d.setLevel((int) (scale * MAX_LEVEL));
                } finally {
                    mInDrawing = false;
                }
                postInvalidateOnAnimation();
            }
            d.draw(canvas);
            canvas.restore();
            if (mShouldStartAnimationDrawable && d instanceof Animatable) {
                ((Animatable) d).start();
                mShouldStartAnimationDrawable = false;
            }
        }
    }
```

onDraw 里面是直接使用 drawable 的 draw 方法来绘制的，那说明肯定是在哪个地方使用 setBounds 来设置 drawable 的 bound 来设置绘制位置的（这里称赞一下 android 的 drawable 家族的设计，真的很棒，抽象的很好，功能也强大，背景不管是 bitmap、9-patch、color、gradient、selector、layer 都只要 draw 就能画出来，只要是 drawable 都能够当作背景参数传入）。

然后搜一下 setBrounds 的地方，发现并不多，然后和 mProgressDrawable 相关的 setBounds 的地方就一个：

```java
    private void updateDrawableBounds(int w, int h) {
        // onDraw will translate the canvas so we draw starting at 0,0.
        // Subtract out padding for the purposes of the calculations below.
        w -= mPaddingRight + mPaddingLeft;
        h -= mPaddingTop + mPaddingBottom;

        int right = w;
        int bottom = h;
        int top = 0;
        int left = 0;

        if (mIndeterminateDrawable != null) {
            // Aspect ratio logic does not apply to AnimationDrawables
            if (mOnlyIndeterminate && !(mIndeterminateDrawable instanceof AnimationDrawable)) {
                // Maintain aspect ratio. Certain kinds of animated drawables
                // get very confused otherwise.
                final int intrinsicWidth = mIndeterminateDrawable.getIntrinsicWidth();
                final int intrinsicHeight = mIndeterminateDrawable.getIntrinsicHeight();
                final float intrinsicAspect = (float) intrinsicWidth / intrinsicHeight;
                final float boundAspect = (float) w / h;
                if (intrinsicAspect != boundAspect) {
                    if (boundAspect > intrinsicAspect) {
                        // New width is larger. Make it smaller to match height.
                        final int width = (int) (h * intrinsicAspect); 
                        left = (w - width) / 2;        
                        right = left + width;          
                    } else {
                        // New height is larger. Make it smaller to match width.
                        final int height = (int) (w * (1 / intrinsicAspect));
                        top = (h - height) / 2;        
                        bottom = top + height;         
                    }
                }
            }
            if (isLayoutRtl()) {           
                int tempLeft = left;           
                left = w - right;              
                right = w - tempLeft;
            }
            mIndeterminateDrawable.setBounds(left, top, right, bottom);
        }

        if (mProgressDrawable != null) {
            mProgressDrawable.setBounds(0, 0, right, bottom);
        }
    }
```

加点打印看看设置的 bound 是多少（编译出来的是 frameworks.jar），发现是 （0，0，268，29），哎，这里的是 29 的高度啊。在 onDraw 再加点打印（把 drawable 的 getBounds() 打印出来），发现是 (0, 8, 268, 21) ，这里就和显示上的对上了，高度 13 ，肥了很多，那应该在别的地方有设置。

分别去 SeekBar、AbsSeekBar 里搜索下 setBounds，发现 SeekBar 里没有， AbsSeekBar 里有：

```java
    private void updateThumbPos(int w, int h) {
        Drawable d = getCurrentDrawable();
        Drawable thumb = mThumb;
        int thumbHeight = thumb == null ? 0 : thumb.getIntrinsicHeight();
        // The max height does not incorporate padding, whereas the height
        // parameter does
        int trackHeight = Math.min(mMaxHeight, h - mPaddingTop - mPaddingBottom);
            
        int max = getMax();
        float scale = max > 0 ? (float) getProgress() / (float) max : 0;
    
        if (thumbHeight > trackHeight) {
            if (thumb != null) {
                setThumbPos(w, thumb, scale, 0); 
            }   
            int gapForCenteringTrack = (thumbHeight - trackHeight) / 2;
            if (d != null) {
                // Canvas will be translated by the padding, so 0,0 is where we start drawing
                d.setBounds(0, gapForCenteringTrack, 
                        w - mPaddingRight - mPaddingLeft, h - mPaddingBottom - gapForCenteringTrack
                        - mPaddingTop);
            }   
        } else {
            if (d != null) {
                // Canvas will be translated by the padding, so 0,0 is where we start drawing
                d.setBounds(0, 0, w - mPaddingRight - mPaddingLeft, h - mPaddingBottom
                        - mPaddingTop);
            }   
            int gap = (trackHeight - thumbHeight) / 2;
            if (thumb != null) {
                setThumbPos(w, thumb, scale, gap);
            }   
        }   
    }
```

哎，好多加、减的计算咧，分析上面的代码会发现是这样的： 如果 thumbHeight > trackHeight 的，那么进度条的 bounds 其实是居中。thumbHeight 的是由设置的 thumb 的 drawable 的 IntrinsicHeight 来决定的（一般就是图片的高度）。主要在于 trackHeight 是多少， 打印出来发现是 13 : 

<pre>
int trackHeight = Math.min(mMaxHeight, h - mPaddingTop - mPaddingBottom);
</pre>

打印出来发现，padding 是 0，高度 h 是 29，这个高度看 AbsSeekBar 的 onMeasure 函数可以知道如果没有明确指定 view 的高度（使用 wrap_content）的话，由 thumb 和 currentDrawable（progressDrawable） 中 IntrinsicHeight 最大（加上 padding ）的决定。这里 thumb 的 29 明显比 progressDrawable 的 6 要大，所以主要就看 mMaxHeight 了， 这个 mMaxHeight 在 AbsSeekBar 中没有定义，那就是父类 ProgressBar 的，跑去一看发现在构造函数中：

```java
mMaxHeight = a.getDimensionPixelSize(R.styleable.ProgressBar_maxHeight, mMaxHeight);
```

这个东西由 xml 中的 android:maxHeight 决定的，如果不指定，则使用系统 style 中的默认值，这里好像是 13。果然坑爹咧，进度条的高度系统不是用你图片的高度，而是使用 android:maxHeight 来决定（大多数情况下由这个决定，因为很前面的代码发现，这个需要 thumb 图片的高度比进度条的图片高才行，不过一般都是这样的）。

所以我设了下 maxHeight 为 6 （进度条图片高度），果然进度条就不肥了，高度老老实实变成 6 了，官方文档啥都没说。

```xml
    <SeekBar
        android:layout_width="300dp"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:progressDrawable="@drawable/seek_drawable"
        android:thumb="@drawable/seek_thumb"
        android:maxHeight="6dp"
        /> 
```

果然不肥了：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/suck-problem-seek-bar/4.jpeg)

