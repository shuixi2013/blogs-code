title: (转) Android locales 本地化
date: 2015-01-27 23:37:16
updated: 2015-01-27 23:37:16
categories: [Android Framework]
tags: [android]
---

## 1. ICU
ICU4C(ICU for C，http://site.icu-project.org/)是ICU在C/C++平台下的版本, ICU(International Component for Unicode)是基于"IBM公共许可证"的，与开源组织合作研究的, 用于支持软件国际化的开源项目。ICU4C提供了C/C++平台强大的国际化开发能力，软件开发者几乎可以使用ICU4C解决任何国际化的问题，根据各地的风俗和语言习惯，实现对数字、货币、时间、日期、和消息的格式化、解析，对字符串进行大小写转换、整理、搜索和排序等功能，必须一提的是，ICU4C提供了强大的BIDI算法，对阿拉伯语等BIDI语言提供了完善的支持

ICU首先是由Taligent公司开发的，Taligent公司现在被合并为IBM?公司全球化认证中心的Unicode研究组，然后ICU由IBM和开源组织合作继续开发，开源组织给与了ICU极大的帮助开始ICU只有Java平台的版本，后来这个平台下的ICU类被吸纳入SUN公司开发的JDK1.1，并在JDK以后的版本中不断改进。C++和C平台下的ICU是由JAVA平台下的ICU移植过来的，移植过的版本被称为ICU4C，来支持这C/C++两个平台下的国际化应用。

ICU4C和ICU4C区别不大，但由于ICU4C是开源的，并且紧密跟进Unicode标准，ICU4C支持的Unicode标准总是最新的；同时，因为JAVA平台的ICU4J的发布需要和JDK绑定，ICU4C支持Unicode标准改变的速度要比ICU4J快的多。

ICU用户指南： [指南地址](http://userguide.icu-project.org/locale "指南地址")

## 2. ANDROID 语言包
Android 使用的语言包就是ICU4C，位置：external/icu4c。
Android2.1及2.2支持的26种语言（locales）
Android2.3以上版本支持的57种语言（locales）.

定制语言，在PRODUCT_LOCALES字段里添加需要语言，如：

<pre config="brush:bash;toolbar:false;">
PRODUCT_LOCALES := en_US zh_CN
</pre>

则系统里只有英语和汉语两种语言。然后语言的选择处理是在external/icu4c/stubdata/Android.mk里进行的，如下：

<pre config="brush:bash;toolbar:false;">
config := $(word 1, \

            $(if $(findstring ar,$(PRODUCT_LOCALES)),large) \

            $(if $(findstring da,$(PRODUCT_LOCALES)),large) \

            $(if $(findstring el,$(PRODUCT_LOCALES)),large) \
        .....    \
            us)
</pre>

在android2.2中最终生成/system/lib/libicudata.so
在android2.3以上版本中最终生成使用的是/system/usr/ict/icudt44l.dat.

## 3. 默认语言
在PRODUCT_LOCALES字段里，将要选择的语言放在第一位，如：PRODUCT_LOCALES := en_US zh_CN

## 4. 增加语言支持
增加系统版本支持的语言范围内的语言，在setting中增加语言选择。 /build/target/product目录下，language_full.mk|language_small.mk 看你的编译选项使用那个文件了，修改PRODUCT_LOCALES ，如下PRODUCT_LOCALES包括了57中语言的支持：

<pre config="brush:bash;toolbar:false;">
PRODUCT_LOCALES :=ar_EG ar_IL bg_BG ca_ES cs_CZ da_DK de_AT de_CH de_DE de_LI el_GR en_AU en_CA en_GB en_IE en_IN en_NZ en_SG en_US en_ZA es_ES es_US fi_FI fr_BE fr_CA fr_CH fr_FR he_IL hi_IN hr_HR hu_HU id_ID it_CH it_IT ja_JP ko_KR lt_LT lv_LV nb_NO nl_BE nl_NL pl_PL pt_BR pt_PT ro_RO ru_RU sk_SK sl_SI sr_RS sv_SE th_TH tl_PH tr_TR uk_UA vi_VN zh_CN zh_TW
</pre>

增加android系统不支持的语言，如在android2.2中增加对越南语，泰语（这两中语言android2.3中才支持）的支持。例：android2.2系统添加希伯来文

我大概是这样修改的：
  1.frameworks\base\data\fonts目录下的字体库，替换为希伯来的
  2.将mk文件中的 PRODUCT_LOCALES 添加he_IL希伯来的支持
  3.external\icu4c\stubdata\Android.mk 添加希伯来的国籍问题 $(if $(findstring he,$(PRODUCT_LOCALES)),large) \
  4.在应用程序中，在res下建立目录 values-iw-rIL，并翻译成希伯来文。。
  5.make clean，之后整个make

修改或增加ICU资源或定义，比如对locales地区数字、货币、百分比等书写习惯进行修改。轻易不要修改，除非ICU资源本身有bug.

<pre config="brush:bash;toolbar:false;">
   /external/icu4c/data/locales/

  NumberElements{
        ",",
        " ",
        ";",
        "%",
        "0",
        "#",
        "-",
        "E",
        "‰",
        "∞",
        "NaN",
        "+",
    }
    NumberPatterns{
        "#,##0.###",
        "¤#,##0.00",
        "#,##0%",
        "#E0",
    }
</pre>

编译：
   /external/icu4c/runConfigureICU Linux
   make -j2

这个目录下的文件定义了各国或地区的语言使用习惯，编译后在android2.3中生成icudtxxl-all.dat,icudtxxl-large.dat数据文件（android2.2稍有不同） 数据文件android编译过程中将复制到out/target/product/xxxxx/system/usr/icu/icudtxxl.dat,在android中经JDK读取解释，如/java/text/NumberFormat.java 这样同一个数字在不同的语言locales设置情况下，将按地区习惯显示

## 参考
[如何添加一种语言？](http://blog.csdn.net/zhq56030207/article/details/6239979 "如何添加一种语言？")
[http://www.eoeandroid.com/thread-46129-1-1.html](http://www.eoeandroid.com/thread-46129-1-1.html "http://www.eoeandroid.com/thread-46129-1-1.html")
[http://www.eoeandroid.com/thread-46129-1-1.html](http://www.eoeandroid.com/thread-46129-1-1.html "http://www.eoeandroid.com/thread-46129-1-1.html")
[http://developer.android.com/reference/java/util/Locale.html](http://developer.android.com/reference/java/util/Locale.html "http://developer.android.com/reference/java/util/Locale.html")
[http://topic.csdn.net/u/20110308/11/7b29dfdf-f106-45ac-baa1-c4bcf19252f5.html?32827](http://topic.csdn.net/u/20110308/11/7b29dfdf-f106-45ac-baa1-c4bcf19252f5.html?32827 "http://topic.csdn.net/u/20110308/11/7b29dfdf-f106-45ac-baa1-c4bcf19252f5.html?32827")
[http://hi.baidu.com/xie_jack/blog/item/ac3f390aef09339a0a7b823b.html](http://hi.baidu.com/xie_jack/blog/item/ac3f390aef09339a0a7b823b.html "http://hi.baidu.com/xie_jack/blog/item/ac3f390aef09339a0a7b823b.html")
[http://blog.csdn.net/seker_xinjian/article/details/6289191](http://blog.csdn.net/seker_xinjian/article/details/6289191 "http://blog.csdn.net/seker_xinjian/article/details/6289191")
[http://www.douban.com/group/topic/13422793/](http://www.douban.com/group/topic/13422793/ "http://www.douban.com/group/topic/13422793/")

