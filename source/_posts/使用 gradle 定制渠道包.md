title: 使用 gradle 定制渠道包
date: 2016-03-27 11:33:16
updated: 2016-12-08 16:37:18
categories: [Android Development]
tags: [android]
---

gradle 目前 android studio 采用的全新打包方式，其实也算不上全新了，因为已经改用 2、3 年了吧。相对 ant 来说，gradle 在打不同渠道包，和不同渠道包的定制上比 ant 要优秀不少。改用 gradle 也快一年了，这里稍微记录一下用法和一些坑。

## 简单使用

[这里](http://developer.android.com/tools/building/building-cmdline.html "这里") 有介绍怎么使用 gradle 构建 android apk， [这里](http://gradle.org/ "这里") 是 gradle 的官网，可以查看 gradle 的具体用法。其实要简单的使用 gradle 编译出一个 apk 很简单，和 ant 很像，只要弄一个模板照着套一下就可以了。这里稍微写下我目前使用的模板吧。注意一下，我这里的模板全部是使用**本地的库**去构建的，没用采用网上比较流行的让 gradle 去 maven 的仓库中去在线获取，因为老外的网络总是比较稳定的，所以我个人比较倾向于提前把需要的库下载到本地来引用，要升级就升级本地库就可以。

首先每一个要编译的工程下面要有一个叫 **build.gradle** 的文件，这个就是 gradle 的编译脚本。然后主工程（生成 apk 的那个工程）下还需要有下面几个文件：

* **settings.gradle**
记录了编译这个 apk 需要引用到的工程

* **gradlew**
gradle 命令脚本（linux, mac 版）

* **gradlew.bat**
gradle 命令脚本（window 版）

* **gradle/**
这个文件夹下有一个 wrapper 的文件夹，wrapper 下有2个文件： gradle-wrapper.jar 和 gradle-wrapper.properties，我使用的 gradle 都是本地的，所以要放这个（好像和版本有关系，目前我用的是 2.2.1 的），网上很多 wrapper 是在线的，就不用发这个。


所以一般一个 apk 的 gradle 构建的目录结构就是这样的：

<pre>
| app/
|  |----- settings.gradle
|  |----- build.gradle
|  |----- gradlew
|  |----- gradlew.bat
|  |----- gradle/
|           |---- wrapper/
|                    |---- gradle-wrapper.jar
|                    |---- gradle-wrapper.properties
|
| module1/
|  |---- build.gradle
|
| module2/
|  |---- build.gradle
|
|
</pre>

### 主工程配置

主工程配置其实就只要写 settings.gradle 和 build.gradle 这2个文件，其它的直接复制过来就行了。

settings.gradle:

```groovy
// 这个 include 声明了编译这个 app 需要哪里模块（库）
include 'module1', 'module2', 'module3', 'module4', 'moduel5-debug', 'moduel5-release', 'moduel6', 'moduel7'

// 下面是定义上面声明的模块工程路径，注意这里使用的相对路径，不要使用绝对路径，除非你只有一个人开发，原因就不要我多说了吧
// 这里的相对路径是：module 和 app 放在同级目录，就是上面那个结构图那样的
project(':module1').projectDir = new File('../module1')
project(':module2').projectDir = new File('../module2')
project(':module3').projectDir = new File('../module3')
project(':module4').projectDir = new File('../module4')
project(':module5-debug').projectDir = new File('../module5/debug')
project(':module5-release').projectDir = new File('../module5/releaes')
project(':module6').projectDir = new File('../module6')
project(':module7').projectDir = new File('../module7')
```

build.grale（这个是 app 工程的模板）:

```groovy
// Top-level build file where you can add configuration options common to all sub-projects/modules.

// this is someting gradle need to run
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        // 这里可以指定使用 gradle 的版本, 如果本地没有的话，会自动去网上下载
        classpath 'com.android.tools.build:gradle:1.0.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

// this is someting project need
allprojects {
    repositories {
        jcenter()
    }
}

// 编译主工程，要选择 application gradle 插件
apply plugin: 'com.android.application'

android {
    // 指定编译 sdk 和 build tools 版本
    // 最好主工程和模块的都一样，当然好像有个命名可以让 app 的忽略模块指定的
    compileSdkVersion 23
    buildToolsVersion "21.1.2"

    lintOptions {
        // 忽略 lint 检测出的错误，当然好的应用是应该打开这个选项
        // 但是打开后，只要 lint 检测出错误，就会编译失败，例如使用高于 min sdk level 的 api，layout 中缺少某几个元素的配置等等
        // 由于我们暂时没那么精力顾及这些，暂时关闭了
        abortOnError false
    }
    dexOptions { 
        // 这个不是解决方法数超过 65535 的，而且解决某些情况下 R 资源太多了，编译会报个错误
        // 还需要在 project.properties 中加入 dex.force.jumbo=true
        // for limit string in one dex file, we have too many R drawable(emoji)... ...
        jumboMode = true 
    }
    // 配置默认配置
    defaultConfig {
        // apk 的包名、min sdk、target sdk、版本号等
        // 注意 AndroidManifest 不要写得和这里的不一样，会被覆盖掉的
        applicationId "com.heyemoji.onemms"
        minSdkVersion 16
        targetSdkVersion 17
        versionCode 10
        versionName "1.9"
    }
    sourceSets {
        // 配置默认的代码路径，这里我没用使用 gradle 默认的路径而是修改了。
        // 这样其实很不好，会踩坑的，后面说定制渠道包那会说明
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
            jniLibs.srcDirs = ['libs']
        }
        // 可以为 debug 版本额外配置一个 AndroidManifest，这里不详细说这个，
        // 也是后面定制渠道包那再说
        debug {
            manifest.srcFile 'debug/AndroidManifest.xml'
        }
    }
    signingConfigs {
        // release 版的签名配置，好像据说有个办法可以隐藏密钥的密码的，我暂时没管它了
        // debug 版没有配置的会用 android sdk 默认的调试签名
        release {
            storeFile file("release.keystore")
            keyAlias "launcherkey"
            storePassword "showmethebug?"
            keyPassword "showmethebug!"
        }
    }
    // 根据不同的编译类型进行配置，主要分为 debug 和 release 2种类型
    buildTypes {
        release {
            // release 开启混淆，以及混淆的配置文件
            // 这里为什么要叫 minify 呢，因为混淆会把 com.android.view.xx.xx 的类名变成 a.b.c 之类的
            // 这样能有效的减少编译后的 dex 文件的大小，所以叫 minify（最小化），
            // 所以一般开启混淆的 release 版的大小能比 debug 版小几M左右
            // 配置好这个选项能有效的减小最终发布 apk 的大小
            minifyEnabled true
            proguardFiles 'proguard.flags'
            signingConfig signingConfigs.release
        }
        debug {
            // 配置 debug 版 version name 带 "-dev" 后缀
            versionNameSuffix "-dev"
        }
    }
    // 这里是在编译完成后，在生成的文件中（在 build/output/ 下面）遍历，
    // 找到已经签名过的 apk 文件（不以 ungaligned.apk 结尾的 .apk 文件），
    // 将其改名字为 "app-v(版本号)-(产品名字)-debug/release.apk" 格式的名字。
    // 别小看这个步骤，经常打包发布的，这个自动名字的小功能会很事情的 
    applicationVariants.all { variant ->
        variant.outputs.each { output ->
            def outputFile = output.outputFile
            if (outputFile != null && outputFile.name.endsWith('.apk') && !outputFile.name.endsWith('unaligned.apk') ) {
                def flavor = variant.mergedFlavor
                def flaverName = variant.properties.get('flavorName')
                if ('' != flaverName) {
                    flaverName = '-' + flaverName
                }
                def debugFlag = '-release'
                if (variant.buildType.isDebuggable()) {
                    debugFlag = "-debug"
                }
                def suffix = flavor.getVersionName() + flaverName + debugFlag
                def fileName = "app-v${suffix}.apk"
                output.outputFile = new File(outputFile.parent, fileName)
            }
        }
    }
}

// 配置工程依赖关系
dependencies {
    // jar 可以直接引用，同理要使用相对路径
    compile files('../common-libs/common-jar1.jar')
    compile files('../common-libs/common-jar2.jar')
    compile files('./libs/app-jar1.jar')
    compile files('./libs/app-jar2.jar')
    // 这里只能引用在 settings.gradle 中声明的模块，当然这里说的是本地的
    compile project(':moduel1')
    compile project(':moduel2')
    compile project(':moduel3')
    compile project(':moduel4')
    // 还可以根据 debug 和 release 引用不同的模块
    // 当然要同时在 settings.gradle 中声明
    debugCompile project(':module5-debug')
    releaseCompile project(':module5-release')
}
```

### 模块(库)配置

模块工程就只要配置 build.gradle 文件就可以了：

```groovy
// 模块的插件要选择 android-library 这个 gradle 插件
apply plugin: 'android-library'

android {
    // 编译 sdk 和 build tools 版本最好保持和主工程一致
    compileSdkVersion 23
    buildToolsVersion "21.1.2"

    // 同样关闭 lint 检测出错
    lintOptions {
        abortOnError false
    }
    defaultConfig {
        // 这里的 min sdk 和 target sdk 也最好和主工程保持一致
        minSdkVersion 14
        targetSdkVersion 17
        // 模块的版本号统一写成 1 和 1.0，因为生成的 apk 版本是以主工程为准的
        versionCode 1
        versionName "1.0"
    }
    // 同样是指定代码路径（同样不推荐这么做）
    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
        }
    }
    buildTypes {
        // 模块关闭混淆，当然可以配置
        release {
            minifyEnabled false
        }
    }
}

dependencies {
    // 模块也可以直接引用 jar
    compile files('../common-libs/common-jar3.jar')
    // 模块中引用的模块，也必须要在主工程的 settings.gradle 声明才行
    compile project(':moduel6')
    compile project(':moduel7')
}
```

和 ant 一样，gradle 中 jar，可以直接引用，而不用变成模块（库）工程。但是 jar 中无法包含 .R 资源（无法在代码中引用），要有 .R 资源的库，必须写成模块工程。但是自从 google 改用 android studio（gradle）构建后，为了改善这种情况，定义一种新的库包，叫 .aar 。其实这个就是个 zip 包，包括了 jar 和 res 资源。有了 aar 之后，就能把 aar 像 jar 一直直接引用了。使用 gradle 编译的模块，都会生成 aar 文件的。但是这个目前有个 bug：就是如果模块工程中引用了 aar，那么就会在编译出来的 apk 中产生找不 aar 中的 .R 的情况（在运行的时候）。但是如果是主工程直接引用 aar 就没这个问题。鉴于目前的这个 bug，只要是有 res 的库，我全都是整成模块工程来引用的。直接引用 aar(jar) 还有一个问题，就是在混淆的时候，如果一个 aar(jar) 被多个工程直接引用，那么在混淆的时候会出错，解决办法还是只有把 jar 变成模块工程来引用（网上说应该要整理你工程的依赖关系，让 jar 只被引用一次，我就日了狗了，像 android-support-v4.jar 这类 jar，一堆模块需要引用好不）。

### 编译命令

写完配置文件后，第一次编译的时候，需要主工程目录下生成一个 local.properties 文件，里面包含了本地 sdk 和 ndk 的信息。使用

<pre>
android update project -p .
</pre>

生成，那个 "." 是工程路径， "." 代表是当前路径，你也可以指定被的路径，例如： android update project -p demo/ 之类的，但是生成的 local.proerties 是在你指定的工程目录下的。如果有些时候生成这个文件的时候报你当前没有指定的 sdk 的时候，需要检查下主工程目录下的 project.properties 指定的 target=android-17 是不是指定了你没有的 sdk 版本（当然这个 target 最好设置得和 build.gradle 中的一样，android 这个命令只看这个 project.properties 中设置的，不看 build.gradle 中指定的）。如果要是还是生成不了，其实可以自己手动编辑一个，这个 local.proerties 很简单的，就配置一个 sdk 路径就行了：

```bash
# This file is automatically generated by Android Tools.
# Do not modify this file -- YOUR CHANGES WILL BE ERASED!
#
# This file must *NOT* be checked into Version Control Systems,
# as it contains information specific to your local configuration.

# location of the SDK. This is only used by Ant
# For customization when using a Version Control System, please read the
# header note.
sdk.dir=/home/mingming/android/android-sdk-linux
```

有了 local.proerties ， 就可以巧 gradle 的命令了：

```bash
// 清除之前编译生成的所有文件
./gradlew clean
// 编译 apk
./gradlew build
```

上面只是说了最常用的2个 gradle 命令，其中 build 的命令是会编译出所有的产品包的（包括每一个产品的 debug 和 release 版），非常适合最后的发布，打包。如果要单独编译出某个产品的某个版本，可以使用 ./gradlew task 查看当前支持编译的哪些产品的版本。

## 定制不同的渠道包

这里说的渠道包，是传统的说法，一般是在 AndroidManifest 中定义一个 meta-data 标签，然后在代码里面取出来，做一些统计上报处理。然后只要不同的 apk 编译前修改这个 meta-data 标签中的值，然后编译几个 apk 就算不同的渠道包了。当然鉴于这种思路，可以扩展很多别的功能，例如说在 meta-data 中定义一个 BUILD_TYPE 标签，debug 和 release 版本修改成不一样值，这样在代码里就知道当前是哪种编译类型，例如说 debug 版可以尽量多的输出一些调试信息，而在 release 则不输出。这样比以前在发布前去注释下一个 Log 要好用多了，引用我头的一句话，靠人工去做这个工作，总会有出错的时候，有些时候发布的时候忘记注释掉一些代码了，就不好了。用这种方法，根本不需要在发布前专门修改代码。

还有功能更强大的，例如说 debug 版和 release 版，采用不同的代码。其实上面已经稍微演示了一下这个功能了，针对 debug build 和 release build 可以引用不同的模块的，但是这些模块都是实现同一个功能的，只是处理不一样而已，这个特别适合于一些运行时的调试工具（其实上面那个 moduel5-deubg 和 moduel5-release 就是大名鼎鼎的 leakcanary 的 debug 和 release 引用）。

### 采用 gradle 默认配置

还有更进一步的需要，例如说不同的渠道包，我需要不同的 apk icon、名字、还有一些资源的图片不一样。这些都可以通过 gradle 来定制。下面就所下具体怎么做：

首先 gradle 有一个 productFlavors 配置属性：

```groovy
android {
    // ... ...

    productFlavors {
        apk1 {
            applicationId "com.my.apk"
        }
        apk2 {
            applicationId "com.my.apk.apk2"
        }
    }

    // ... ...
}
````

在这个 productFlavors 中可以定义多个产品名字（渠道名字）。然后就使用 gradle 编译出不同产品的 apk，定义了不同的产品后在 ./gradlew task 中就会多几个编译选项用于单独编译这几个产品。如果不定义 productFlavors 的话，那么就会使用默认产品（这个产品是没名字的）。但是定义了多个产品名字后，要选择一个当作默认产品，例如说上面不同的产品，包名是不一样的，要把默认产品的包名字写到 defaultConfig 里面去。这个主要为了导入 idea（android studio）调试的时候能编译出一个 apk 出来。

可以看得到上面不同的产品，可以定义不一样的包名，其实还可以定义别的不一样的东西，例如说 versionName 不一样，这些我是通过 AndroidManifest 来实现的。在继续说下去前，提醒一下，如果不同渠道包修改了包名，你的 apk 要注意以下几点：

* 第一：
AndroidManifest 中所有的 activity、service、receiver、provider 全部要写**全名**，不要再写 .xx 的称了，否则除了默认渠道，其它的会找不到的

* 第二：
注意 layout 里面的自定义 view 属性的引用，要写成： **xmlns:app="http://schemas.android.com/apk/res-auto"** ，不要再写死包名了，否则也上面说的问题的

* 第三：
代码里面不能在任何地方写实包名，例如弄个 final static String PKG_NAME 之类的，必须采用动态获取包名方式

* 第四： 
有 provider 的，注意定义 authority 的时候，不能像以前一样写死包名，要动态获取，像这样：

```java
    /* 
     * This authority is used for writing to or querying from the clock
     * provider.
     */
    //public static final String AUTHORITY = "com.my.apk";
    public static String AUTHORITY() {
        return MyApp.getPkgName();
    }
```

当然这还涉及一些引用到这个 authority 的地方，例如说实现 provider 的 UriMatcher 定义，以前是直接在类的 static 代码块里定义的，但是现在不能这么做了，因为这个时候 Application onCreate 还没跑到，我们还没获取到动态的 apk 包名，所以可以改成在一个 init 函数里面初始化，这个 init 函数在 Application 的 onCreate 里调用就可以了：

```java
    private final static UriMatcher sURLMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    /*static {
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(OpenDoorContract.SessionColumns.BASE_PATH), SESSION);
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(OpenDoorContract.KeyColumns.BASE_PATH), KEY);
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(OpenDoorContract.RecordDateColumns.BASE_PATH), RECORD_DATE);
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(OpenDoorContract.RecordColumns.BASE_PATH), RECORD);
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(Session.SESSION_LIST_VIEW_PATH), SESSION_LIST_VIEW);
    }*/
    public final static void init() {
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(OpenDoorContract.SessionColumns.BASE_PATH), SESSION);
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(OpenDoorContract.KeyColumns.BASE_PATH), KEY);
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(OpenDoorContract.RecordDateColumns.BASE_PATH), RECORD_DATE);
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(OpenDoorContract.RecordColumns.BASE_PATH), RECORD);
        sURLMatcher.addURI(OpenDoorContract.AUTHORITY(),
            convertUriMatcherPath(Session.SESSION_LIST_VIEW_PATH), SESSION_LIST_VIEW);
    }
``` 

```java
public class MyApp extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        setDefaultLanguage();

        // init self pkg name
        sAppContext = getApplicationContext();
        sPkgName = getPackageName();
        sVersionName = AppUtils.getAppVersionName(sAppContext, sPkgName);
        sAppId = AppUtils.getValueFromMetaData(sAppContext,
            "UMENG_APPKEY", "");

        String buildType = AppUtils.getValueFromMetaData(sAppContext,
            KEY_BUILD_TYPE, BUILD_TYPE_DEBUG);
        sDebug = BUILD_TYPE_DEBUG.equals(buildType);

        OpenDoorProvider.init();
        DateHelper.init(sAppContext);
        OpenDoorProvider.activeDB(sAppContext);

        // ... ...
    }

}
```

然后上面讲到了，不建议修改默认的代码路径（就是配置 sourceSets）。这个是因为默认，gradle 就支持了不同的产品，不同的版本，可以定义不同的代码文件和资源文件。如果你修改了默认的路径的话，gradle 的这条特性就没了。然后要自己通过 gradle 的别的配置去配置。我之前是因为看 gradle 默认的代码路径和 eclipse 的不一样，我一开始看着不太爽，就给它改了 -_-||。结果通过 gradle 别的配置去支持这个功能，一堆坑，因为不管我怎么搞，最后编译的产品的一些属性总会覆盖别的，就是说最后虽然能编译出不同的产品的 apk，但是里面内容还是一样的。反正我是最后怎么搞，都搞不好，最后试了下默认的路径，结果就可以了，也没什么问题。

现在说下，默认 gradle 的路径要怎么放。gradle 默认的代码路径是放在工程目录的 src 文件夹下面。在 src 下面可以分不同的产品放不同的文件夹，默认的是 main 文件夹，下面放 AndroidManifest.xml, project.properties, jniLibs（so 库放这里）, res（.R 资源放这）, assets（assets 资源放这），java（java 代码放这里，这里就可以是 src/xx/xx/xx.java 了），jni（native 代码放这里）。

然后不同的产品根据产品名字和编译类型可以放置不同的文件夹，然后在不同的文件夹下可以定制这个产品版本的特性。例如说放额外的 AndroidManifest.xml，放不同的 res 文件。但是要**注意一点**：这里不同的产品定制的代码，首先还是会采用 main 里面的，然后再把产品定制的**合并**过去。注意是合并，什么叫合并，举个例子： main 里面默认的 AndroidManifest.xml 中的一个叫 BUILD_TYPE 的 meta-data 字段是 release，那如果我在 apk1Debug 中的 AndroidManifest.xml 中把这个 BUILD_TYPE 的 meta-data 写成了 debug，那么编译的 apk1 的 debug 版的 AndroidManifest.xml 的 BUILD_TYPE 就是 debug 而不是 release。所以说不同的产品里面只要定制不同的部分就行了，不用把所有的 AndroidManifest.xml 都重写一篇。关于更多的 AndroidManifest.xml 的合并可以看下 [官方合并说明](http://developer.android.com/tools/building/manifest-merge.html "官方合并说明")。要注意一点，要覆盖默认的 meta-data 的值，需要加一个 tools:replace 的声明，像下面这样：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <application>
        <meta-data tools:replace="android:value"
            android:name="BUILD_TYPE"
            android:value="debug" />
    </application>
</manifest>
```

然后是一些 res 资源的定制。前面说了不同的产品，可能一些图标、字符串、图片不一样，这个其实很简单，把资源的名字定下来，然后 main 里面放默认产品的，其它产品文件夹的 res 下放不同的图片，然后名字和 main 里面的一样就可以了。最后编译出来不同产品的 apk 使用的资源就会不一样了。而且这样做，**不同产品的 apk 只会有一份图片**，不会把其它产品的图片包含进来，apk 的体积不会受不同渠道包定制资源的业务需求而变大。由于资源会覆盖的特性，这里稍微提一下题外话，写带资源的库的时候，**资源一定要带一个前缀**，避免覆盖使用你库的工程的资源（一些 github 上著名的库，已经都是这么做的了，看来大家都有共识了）。

前面说了，可以分产品，然后每个产品又分为 debug 和 release 版。那么我有些属性只是在 debug 和 release 中区分，例如说 BUILD_TYPE；有些属性又是产品级别的区别，例如说 provider 的 authorities；有些是既和产品相关也和 debug、releaes 版相关的，例如说渠道名。这些都是可以解决的。前面讲了 src 下的文件夹可以按照产品名和编译类型命名：例如说你定义了2个产品 apk1 和 apk2，每个产品分为 debug 和 release 2个版本，那么可以命名的文件夹有：

* **debug/**
所有的 debug 版本（包括 apk1、apk2 的 debug版）会使用这个文件下的配置

* **release/**
所有的 release 版本（包括 apk1、apk2 的 releaes版）会使用这个文件下的配置

* **apk1/**
所有的 apk1 产品（包括 apk1 的 debug 和 release版）会使用这个文件下的配置

* **apk1Debug/**
apk1 的 debug版 会使用这个文件下的配置

* **apk1Release/**
apk1 的 release版 会使用这个文件下的配置

* **apk2/**
所有的 apk2 产品（包括 apk2 的 debug 和 release版）会使用这个文件下的配置

* **apk2Debug/**
aapk2 的 debug版 会使用这个文件下的配置

* **apk2Release/**
aapk2 的 release版 会使用这个文件下的配置

看完这个应该就知道这么定制了吧。例如说 BUILD_TYPE 只分 debug 和 release 的，就在 debug/ 和 release/ 下放不同的 AndroidManifest：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <application>
        <meta-data tools:replace="android:value"
            android:name="BUILD_TYPE"
            android:value="debug" />
    </application>
</manifest>
```

例如说 provider 的 authorities 只分 apk1 和 apk2，那么在 apk2/ 下放 AndroidManifest（apk1 为默认的在 main 下的 AndroidManifest 有定义了）：
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <application>

        <provider android:name="com.myapp.apk.MyProvider"
            tools:replace="android:authorities"
            android:authorities="com.myapp.apk.apk1"
            android:exported="false" />

    </application>
</manifest>
```

例如说渠道名字，既要分产品名字，也要分 debug 和 release 版的，那么就要分别在 apk2Debug、apk2Release 下放 AndroidManifest（同理 apk1 为默认的，在 main 下的 AndroidManifest 里）：
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    tools:replace="android:versionName"
    android:versionName="1.0-dev">
    <application>
        <meta-data tools:replace="android:value"
            android:name="UMENG_CHANNEL"
            android:value="apk2-dev" />
    </application>
</manifest>
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">
    <application>
        <meta-data tools:replace="android:value"
            android:name="UMENG_CHANNEL"
            android:value="apk2" />
    </application>
</manifest>
```

需要注意一点是 src/ 下这些文件夹的命名，如果你 build.gradle 定义的 productFlavors 名字是 apk1 和 apk2 ，那么文件夹的名字就是 apk1 和 apk2（注意区分大小写的），然后加上编译类型是 apk1Debug 和 apk1Relealse（apk2同理），后面的编译类型首字母是大写的。

然后根据上面的说明能推测出覆盖关系是这样的： apk1Debug 的会覆盖 apk1 和 debug 中的，所以说其实可以在 apk1Debug 中把只区分 apk1 和 debug 都写了，当然这样也可以，但是就要写4份（apk1Debug、apk1Release、apk2Debug、apk2Release），如果再多一个产品就是6份了，再多几个 ... 呵呵。

上面说了这么多，总结一下，使用 gradle 默认的产品定制支持，只要在默认代码路径下（src/）放置要定制的配置、资源就行了，编译系统就会自行合并、覆盖，大概文件结构是这样的：

<pre>
| app/
|  |
|  |--src/ 
|      |-- main/
|      |     |---- AndroidManifest.xml（完整版）
|      |     |-- res/（完整版）
|      |
|      |-- debug/
|      |     |---- AndroidManifest.xml（debug 版定制）
|      |
|      |-- release/
|      |     |---- AndroidMainfest.xml（release 版定制）
|      |
|      |-- apk2/
|      |     |---- AndroidMainfest.xml（apk2 渠道定制）
|      |     |-- res/（main 同名资源 apk2 渠道定制）
|      |
|      |-- apk2Debug/
|      |     |---- AndroidMainfest.xml（apk2 渠道 debug 版定制）
|      |
|      |-- apk2Release/
|      |     |---- AndroidMainfest.xml（apk2 渠道 release 版定制）
|
</pre>

上面这里还没 java 代码上的定制，所以没就没列出来了。从这里就能看出 gradle 定制不同的渠道包有多强大了。这里再提一下， main 里面是默认的产品（这里就是 apk1），然后导入 idea（android studio） 开发调试的时候，编译出来的就是 apk1Debug 版，要编译其它的版本要使用 gradle 命令行编译。

### 自定义配置

上面是采用 gradle 默认支持的方式。但是也可以不怎么干，就像前面简介那样，开了默认的代码路径。那这种情况下怎么实现不同的产品之间定制呢。我们现在的做法是，把核心代码（包括资源）变成一个 core 模块，然后不同产品，建立不同的 app 工程，引用这个 core 模块，如果直接使用的 .R 资源，可以在 AndroidManifest 中定义一个产品名字的 meta-data，然后在代码中根据不同的产品，获取不同的资源（带个后缀就好区分了）。如果不直接使用 .R 的资源可以放到 app 工程中。然后 core 模块里面的 AndroidManifest 是空的，每个 app 写自己不同的 AndroidManifest 就实现不同的定制了。

这样实现和上面采用 gradle 默认方式的区别在于：

1. 自定义的如果直接使用 .R 资源，那么在最后编译的 apk 中会存在重复的资源（不同产品之间的），会增大 apk 的体积
2. 自定义的需要在代码中判断当前的产品是哪个，从而引用不同的 .R
3. 自定义的编译不同的产品需要到不同的 app 工程目录去
4. 上面说的都是缺点，但是它有一个很重要的优点，也是默认不支持的：可以在 idea（android studio）中导入不同的产品，进行开发调试，所以感觉这个不像渠道包了，而是挂同一份代码的不同马甲 apk（我们现在很多应用就是这样的）。

这种方式的文件结构目录是这样的：

<pre>
| core/
|  |---- AndroidManifest.xml（空壳）
|  |-- res/（完整） 
|  |-- src/（完整）
|
|  apk1
|   |---- settings.gradle
|   |---- build.gradle（apk1 版）
|   |---- AndroidManifest.xml（apk1 版）
|   |-- res/（apk1 版）
|   |-- src/（apk1 版）
|
|  apk2
|   |---- settings.gradle
|   |---- build.gradle（apk2 版）
|   |---- AndroidManifest.xml（apk2 版）
|   |-- res/（apk2 版）
|   |-- src/（apk2 版）
|
</pre>


## 小结

所以看不同的使用场景，应用不同的实现方式。并没有说哪一种要好过另一种，这就叫工具是死的，人是活的。感觉 gradle 这种定制不同产品的方式，颇有当年 C/C++ 预编译命令的味道。

## 附录：AndroidManifest 能 override 的属性

稍微记录一下：

### minSdkVersion

其实好像 module 的 AndroidManifest 可以不指定 minSdkVersion, targetSdkVersion 和 versionCode, versionName 的。但是有一些还是已经指定了，如果 module 指定了 minSdkVersion ，那么 app 的 minSdkVersion 一定不能大于 moudle 的，否则就会报错。你可能会说确实应该要这样，但是有些时候由于某些蛋疼的原因，其实 module 的 minSdkVersion 并不是真正使用了（就是说实际上这个 module 能在更低的 api level 上跑的，但是就是偏偏写了一个大的 minSdkVersion）。所以有些我们想只以 app AndroidManifest 中指定的 minSdkVersion 为准，忽略 module 中指定的 minSdkVersion。这个是可以做到的，通过一个 **tools:overrideLibrary** 属性指定：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.heyemoji.emojiup"
    android:versionCode="1"
    android:versionName="1.0">

    <uses-sdk android:minSdkVersion="9" android:targetSdkVersion="17" 
        tools:overrideLibrary="android.support.v4, com.keyboard.common.utilsmodule"/>

... ... 

</manifest>
```

例如说，上面这个就是覆盖了 module： android.support.v4 和 com.keyboard.common.utilsmodule 中指定的 minSdkVersion （注意上面写的 module 的 pakcageName）。具体的可以看上面说的 [官方合并说明](http://developer.android.com/tools/building/manifest-merge.html "官方合并说明") 


## 附录：关于 Flavors 的问题

gradle 是可以定义不同的 Flavors，app 和 module 都可以，但是我强烈建议只在 app 中定义。如果你在 module 里分了 Flavors，在编译 app 的时候敲 task 确实能看到多了几个 Flavors 的 task 出来，但是其实你如果 build module 的 Flavors 是无法打包 app 出来的，但是如果直接 build，module 又无法根据 Flavors 走不同的分支。当然有可能是我自己还没学会怎么整，但是我还是建议只在 app 使用 Flavors，如果由于某些特殊原因，代码的不同是在 module 里面的，其实也可以绕到 app 中去，在 app 定义 Flavors，然后 module 抽出大多数相同的一部分变成一个 module，然后根据不同的 Flavors 建几个分支 module，然后在 app 中不同的 Flavors 引用不同的分支 module 就行了（分支 module 都引用抽出来的那个大 module）。

还有导入了 idea（IDE）后也是可以编译不同的 Flavors 的，在 idea 的左边的有一个 Build Variants 的选项，可以选择要编译的 Flavors 的：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/gradle-custom-flavors/ide_build_flavors.jpg)




