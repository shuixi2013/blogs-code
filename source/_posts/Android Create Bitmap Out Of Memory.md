title: Android Create Bitmap Out Of Memory
date: 2015-01-25 23:25:16
categories: [Android Development]
tags: [android]
---

## 问题

Android 对图片的解码、创建是有内存限制的，在弄一些图片多的程序，不小心很容易出 Out of Memory（OOM）的错误。图片用的内存好像是 native 的内存，由于 4.0 普通 UI 也使用了 GPU 硬件加速，导致系统有不少 UI 的缓冲，所以在高分辨率 4.0 的手机上这个问题更加明显（Galaxy Note、Galaxy Neuxs 等等，估计 native 分配给图片的内存用得差不多了）。在网上找了下资料，发现一个比较有用的方法来避免这个问题。在解码图片的时候，指定一些参数来优化下内存使用情况： BitmapFactory.Options 。这个是 Android 解码图片的参数。

## 解决方法

## BitmapFactory.Options.inSampleSize
这个值是设置解码图片大小的。 <= 1 的话就是图片原始大小。 > 1 的话就会缩小图片， 例如： 4 就是 1/4 。如果你用的图片大小比原始图片要小的话，合理的设置这个值可以降低系统在解码图片时候所使用的内存。PS：文档说这个值如果是 2^n 次方速度会比较快。BitmapFactory.Options 还有个参数 inJustDecodeBounds 可以让系统只是计算图片的一些信息（原始大小），单是不解码，也就是不占用内存。可以配合这个计算出合适的 inSampleSize。

### BitmapFactory.Options.inPurgeable
设置这个值可以让系统在回收内存的时候把图片 pixels 占用的内存回收掉。被回收的图片如果需要再次显示的话，系统会重新解码、载入。这个好像比弱引要好用不少。

### 参考代码
上个参考代码吧：

```java
public static Bitmap decodeBitmap(InputStream is, int targetW, int targetH){
		if (null == is) {
			Log.e(TAG, "InputStream is null!");
			return null;
		}
		
	    Bitmap bmp = null;
	    
	    try {
	        // decide target image size.
	        BitmapFactory.Options bfSizeOp = new BitmapFactory.Options();
	        bfSizeOp.inJustDecodeBounds = true;
	        
	        BitmapFactory.decodeStream(is, null, bfSizeOp);

	        int scale = 1;
	        if (targetW <= 0 && targetH <= 0 ) {
	        	scale = 1;
	        } else {
	        	if (bfSizeOp.outHeight > targetW || bfSizeOp.outWidth > targetH) {
	        		scale = (int)Math.pow(2, 
	        				(int) Math.round(
	        						Math.log(targetW / (double) Math.max(bfSizeOp.outHeight, bfSizeOp.outWidth)) / Math.log(0.5)));
	        	}
	        }
	        
	        // decode with inSampleSize and let it auto-gc.
	        BitmapFactory.Options bfOp = new BitmapFactory.Options();
	        bfOp.inSampleSize = scale;
	        bfOp.inPurgeable = true;
	        bmp = BitmapFactory.decodeStream(is, null, bfOp);
	        
	    } catch (Exception e) {
	    	e.printStackTrace();
	    	return null;
	    }
	    
	    return bmp;
	}
```

## 总结

虽然上面的方面可以缓解高分辨率 4.0 手机上创建 bitmap oom 的问题，但是也不能完全依赖这种方法。还需要针对应用进行优化才行。例如说，只持有显示时候的图片，后台不显示的图片要即使释放掉，等等。还有一个有效的方法，在图片多的地方，持有图片使用软件引用（`SoftReference<Bitmap>`），这样系统就能够即使的回收不使用的图片的内存，当然你得自己处理图片被回收后，重新加载的问题。

**PS**
其实这个问题在新的 Android sdk doc 里的 Android Training --> Advanced Training --> Displaying Bitmaps Efficiently 里就有说明。这个就当是翻译了吧。


