title: Google android source code build 问题总结
date: 2015-01-31 10:58:16
categories: [Android Framework]
tags: [android, install]
---

## 编译 external/chromium_org 出错

编译 external/chromium_org 的时候如果报类似下面的错误：

<pre config="brush:bash;toolbar:false;">
Traceback (most recent call last):
  File "../../base/android/jni_generator/jni_generator.py", line 1065, in <module>
    sys.exit(main(sys.argv))
  File "../../base/android/jni_generator/jni_generator.py", line 1061, in main
    options.optimize_generation)
  File "../../base/android/jni_generator/jni_generator.py", line 996, in GenerateJNIHeader
    jni_from_javap = JNIFromJavaP.CreateFromClass(input_file, namespace)
  File "../../base/android/jni_generator/jni_generator.py", line 507, in CreateFromClass
    stderr=subprocess.PIPE)
  File "/usr/lib/python2.7/subprocess.py",/usr/java/jdk1.6.0_45/bin line 709, in __init__
    errread, errwrite)
  File "/usr/lib/python2.7/subprocess.py", line 1326, in _execute_child
    raise child_exception
OSError: [Errno 2] No such file or directory
make: *** [/home/odexcide/android-4./out/target/product/generic/obj/GYP/shared_intermediates/ui/gl/jni/Surface_jni.h] Error 1
make: *** Waiting for unfinished jobs....
</pre>

那是 jdk 到 javap 没装好。其实不一定是没装，装完 jdk6 后，默认 java 的命令路径是 /usr/bin/java 这个其实是一个 /usr/java/jdk1.6.0_45/bin/java 的链接来的。去 /usr/java/jdk1.6.0_45/bin 下其实是有 javap（这个东西是用来反编译 java class 的） 的，这就好办了，自己手动在 /usr/bin/ 下创建一个 javap 的软链接就行了。

## 5.0 编译 external/chromium_org 出错

如果 javap 设置好，编这个 chromium_org 还是出错，那么可以在 chromium_org 的 Android.mk 加入这么一句：

<pre>
PRODUCT_PREBUILT_WEBVIEWCHROMIUM :=yes
</pre>

这句好像是说不自己编译 chromium 的 webiew（webkit？？），用预编译好的（源码里自带现成的）。

