title: Java 反射的参数问题
date: 2015-01-25 18:50:16
updated: 2015-01-25 18:50:16
categories: [Android Development]
tags: [android]
---

在 java 开发中有时候会用到反射（虽然不推荐频繁使用，但是某些时候确实很方便）。至于怎么用直接看 java docs 或者网上就能找到一大堆（或者看我以前写的东西可以）。但是有些时候一些函数比较高级，在使用反射调用的时候就需要注意了。

## 基本类型数组参数

例如这样的：

```java

	private void setIntsWithArray(int[] ints) {
		mInt1 = ints[0];
		mInt2 = ints[1];
	}

```

这种还算简单的。直接使用相关类型的数组的 class 就行了：

```java

ReflectUtils.invokeMethod(Target.class, targetObj, 
	"setIntsWithArray", 
	new Class[] {int[].class}, 
	new int[] {45, 55});

```

然后那个 invokeMethod 是这样的：

```java

	/**
	 * Invoke specified class method.
	 * 
	 * @param objClass
	 * @param object
	 * @param methodName
	 * @param paramTypes
	 * @param args
	 * @return
	 */
	public final static Object invokeMethod(Class<?> objClass, Object object, 
			String methodName, Class<?>[] paramTypes, Object... args) {
		
		if (null == objClass || null == object ||  
				null == methodName) {
			return null;
		}
		
		Method method = null;
		
		try {
			method = objClass.getDeclaredMethod(methodName, paramTypes);
			method.setAccessible(true);
	        return method.invoke(object, args);
	    } catch (Exception e) {
	        e.printStackTrace();
	        return null;
	    }
	}

```

## 对象类型数组参数

上面那个还算简单的，但是这种就有点麻烦了。这种的函数原型是类似这样的：

```java

	public static class Test {
		private int mTest1;
		public Test() {
			mTest1 = -1;
		}
		public Test(int test) {
			mTest1 = test;
		}
	}

	private void setIntsWithClassArray(Test[] tests) {
		mInt1 = tests[0].mTest1;
		mInt2 = tests[1].mTest1;
	}

```

这种就不能直接像上那样直接传递 Test[] 过去。这样话，穿过去的参数数目会不对的。例如函数的参数只是一个 Test[] 数组，如果你传一个 Test[2] 过去，参数会变成2个。正确的做法是像下面这样：

```java

    	Test[] tests = new Test[] {
    		new Test(65),
    		new Test(75)
    	};
    	Class<?> testClassArray = Class.forName(
    			new Test[] {}.getClass().getName());
    	
    	ReflectUtils.invokeMethod(Target.class, targetObj, "setIntsWithClassArray", 
    			new Class[] {Test[].class}, 
    			testClassArray.cast(tests));

```

这样参数数目才对。至于原因我也不是很清楚，反正参考了下度娘的结果，然后自己试了几次发现这样可以。

以后遇到别的什么一些特殊的参数或是返回值可以一起记录在这里 ... ...

