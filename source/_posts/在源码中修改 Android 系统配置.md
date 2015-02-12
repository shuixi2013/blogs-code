title: 在源码中修改 Android 系统配置
date: 2015-01-27 23:36:16
updated: 2015-01-27 23:36:16
categories: [Android Framework]
tags: [android]
---

Android 的系统（System Settings）的配置文件在 framework/base/core/res/res/values/config.xml 里面。编译模块 framework/base/core/res 能得到 out/target/product/xx/system/framework/framework-res.apk ，这个配置文件中定义的值就打包在这个 apk 中（这个 apk 在板子的 /system/framework 下）。 

例如说，你想把系统设置中自动亮度的开关打开：那就在 config.xml 中找到 config_automatic_brightness_available ，然后改成 true 重新编译 framework-res.apk 就行了。不过一般来说，修改 framework/base/core/res 下的 config.xml 是没用的。 google 提供了一个给 OEM 厂商灵活修改这个配置的方式，在 build/target/board/xx/overlay/frameworks/base/core/res 下面还会有一个 config.xml （xx 就是你 OEM 自己的产品名字，例如 galaxy 之类的）， 如果 build 下面有 overlay 这个目录，那么 framework 里面设置的值是会被这里的覆盖的。所以说改就要改 build 下面那个 overlay 里面的。然后把编出来的 framework-res.apk 替换一下就行了。

最后如果想检验下 framework-res.apk 里面的配置是不是对的，可以用 apk tool 反编译出来看看就可以了。

