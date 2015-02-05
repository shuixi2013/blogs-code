title: Android 一些有意思的命令小工具 —— dumpsys
date: 2015-01-30 16:30:16
categories: [Android Framework]
tags: [android]
---

android 自带了很多 cmds 的小工具，都挺用有的，也有些比较有意思。这里介绍的是 dumpsys 这个工具。说之前先把相关源码位置说下（4.4 的）：

```bash
# service 命令模块
frameworks/native/cmds/dumpsys
```

## 作用

这个工具的作用是让你能够在 shell 中 dump（导出）系统服务（SS）的当前状态，然后输出到 console 中。不过都能输出到 console 中，重定向一下，就能输出到文件中啦，这个工具是**非常有用的**。用法很简单直接敲 dumpsysy 就行了。不过这样是导出所有实现了 dump 接口的 SS 的状态（基本上 SS 都实现了这个接口），然后输出非常的长，类似下面这样：

```bash
Currently running services:
  SurfaceFlinger
  accessibility
  account
  activity
  alarm
  appwidget
  audio
  backup
  battery
  batteryinfo
  bluetooth_manager
  clipboard
  commontime_management
  connectivity
  content
  country_detector
  cpuinfo
  dbinfo
  device_policy
  devicestoragemonitor
  diskstats
  display
  dreams
  drm.drmManager
  dropbox
  entropy
  ethernet
  gfxinfo
  hardware
  input 
  input_method
  iphonesubinfo
  isms
  location
  lock_settings
  media.audio_flinger
  media.audio_policy
  media.camera
  media.player
  meminfo
  mount 
  netpolicy
  netstats
  network_management
  notification
  package
  parentalmode
  permission
  phone 
  power 
  samplingprofiler
  scheduling_policy
  search
  sensorservice
  serial
  servicediscovery
  simphonebook
  statusbar
  telephony.registry
  textservices
  throttle
  uimode
  updatelock
  usagestats
  usb
  user
  vibrator
  wallpaper
  wifi
  wifip2p
  window
-------------------------------------------------------------------------------
DUMP OF SERVICE SurfaceFlinger:
Build configuration: [sf] [libui] [libgui]
Visible layers (count = 5)
+ LayerDim 0x40941008 (DimAnimator)
  Region transparentRegion (this=0x40941144, count=1)
    [  0,   0,   0,   0]
  Region visibleRegion (this=0x40941014, count=1)
    [  0,   0,   0,   0]
      layerStack=   0, z=        0, pos=(0,0), size=(  16,  16), crop=(   0,   0,  -1,  -1), isOpaque=0, needsDithering=0, invalidate=0, alpha=0x00, flags=0x00000000, mShouldTransform=0, tr=[1.00, 0.00][0.
      client=0x4290cb20, identity=2
+ LayerDim 0x40941270 (DimSurface)
  Region transparentRegion (this=0x409413ac, count=1)
    [  0,   0,   0,   0]
  Region visibleRegion (this=0x4094127c, count=1)
    [  0,   0,   0,   0]
      layerStack=   0, z=        0, pos=(0,0), size=(  16,  16), crop=(   0,   0,  -1,  -1), isOpaque=0, needsDithering=0, invalidate=0, alpha=0x00, flags=0x00000000, mShouldTransform=0, tr=[1.00, 0.00][0.
      client=0x4290cb20, identity=3
+ Layer 0x40437d28 (com.android.systemui.ImageWallpaper)
  Region transparentRegion (this=0x40437e64, count=1)
    [  0,   0,   0,   0]
  Region visibleRegion (this=0x40437d34, count=1)
    [  0,   0, 768, 1024]
      layerStack=   0, z=    21000, pos=(-695,0), size=(1466,1024), crop=( 695,   0,1463,1024), isOpaque=1, needsDithering=0, invalidate=0, alpha=0xff, flags=0x00000000, mShouldTransform=0, tr=[1.00, 0.00]
      client=0x40436a68, identity=6
      format= 2, activeBuffer=[1466x1024:1472,  1], queued-frames=0, mRefreshPending=0
            mTexName=5 mCurrentTexture=-1
            mCurrentCrop=[0,0,0,0] mCurrentTransform=0
            mAbandoned=0
            -BufferQueue maxBufferCount=4, mSynchronousMode=1, default-size=[1466x1024], default-format=1, transform-hint=00, FIFO(0)={}
             [00] state=FREE    , crop=[0,0,0,0], xform=0x00, time=0x9e84ca7f2, scale=FREEZE
             [01] state=FREE    , crop=[0,0,-1,-1], xform=0x00, time=0, scale=FREEZE
             [02] state=FREE    , crop=[0,0,-1,-1], xform=0x00, time=0, scale=FREEZE
             [03] state=FREE    , crop=[0,0,-1,-1], xform=0x00, time=0, scale=FREEZE
+ Layer 0x40941648 (com.bbk.studyos.launcher/com.bbk.studyos.launcher.activity.Launcher)

... ...
```

虽然说可以用搜索，但是太长了还是比较麻烦，可以在接某个（或是几个） SS 的名字，就可以导出指定 SS 的当前状态。例如说像下面这样：

```bash
dumpsys window
```

这样就能导出你想看的 SS 的当前状态。稍微注意一下，这里 SS 的名字是注册到 SM 中的那个名字，Binder 篇的时候介绍过了一般不是带包名类名的那个，而是短的那个。具体的可以去翻源代码，或者可以先：

```bash
dumpsys -l
``` 

这样会罗列出所有的 SS 的名字（这个 -l 参数是 4.4 才加的，之前的直接 dumpsys 在一开始也会列出所有的 SS 的名字），然后复制、粘贴一下就行了。

## 分析

这个小工具的作用和基本用法知道了，然后我们看看实现（代码不长我直接全贴了）：

```cpp
static int sort_func(const String16* lhs, const String16* rhs)
{
    // 排下序，按首字母排序，我没去具体看 native String16 的排序代码，
    // 看上面的输出应该是按 ASCI 码值排序的 
    return lhs->compare(*rhs);
}

int main(int argc, char* const argv[])
{
    signal(SIGPIPE, SIG_IGN);
    // 获取 native 层的 SM
    sp<IServiceManager> sm = defaultServiceManager();
    fflush(stdout);
    if (sm == NULL) {
        ALOGE("Unable to get default service manager!");
        aerr << "dumpsys: Unable to get default service manager!" << endl;
        return 20; 
    }

    Vector<String16> services;
    Vector<String16> args;
    bool showListOnly = false;
    // 如果只是 dumpsys -l 的话，那就只要列出当前所有的 SS 的名字就可以
    if ((argc == 2) && (strcmp(argv[1], "-l") == 0)) {
        showListOnly = true;
    }
    // 如果只是要列出所有的 SS
    // 或者是没指定 SS，那么就取所有的 SS
    if ((argc == 1) || showListOnly) {
        // 调用 SM 的 listServices 获取所有的 SS 的 IBinder 接口（Bp）
        services = sm->listServices();
        // 使用自定义排序函数排序
        services.sort(sort_func);
        // 如果没给参数，就默认给 -a
        args.add(String16("-a"));
    } else {
        // 指定了要获取的 SS ，就添加指定的 SS
        services.add(String16(argv[1]));
        // 获取指定要 dump 的参数
        for (int i=2; i<argc; i++) {
            args.add(String16(argv[i]));
        }   
    }

    const size_t N = services.size();

    if (N > 1) {
        // first print a list of the current services
        aout << "Currently running services:" << endl;

        // 如果 dump 的 SS 数量大于 1，先打印一下 SS 的名字
        for (size_t i=0; i<N; i++) {
            sp<IBinder> service = sm->checkService(services[i]);
            if (service != NULL) {
                aout << "  " << services[i] << endl;
            }
        }
    }

    if (showListOnly) {
        return 0;
    }

    for (size_t i=0; i<N; i++) {
        sp<IBinder> service = sm->checkService(services[i]);
        if (service != NULL) {
            if (N > 1) {
                aout << "------------------------------------------------------------"
                        "-------------------" << endl;
                aout << "DUMP OF SERVICE " << services[i] << ":" << endl;
            }
            // 调用 SS IBinder 的 接口
            int err = service->dump(STDOUT_FILENO, args);
            if (err != 0) {
                aerr << "Error dumping service info: (" << strerror(err)
                        << ") " << services[i] << endl;
            }
        } else {
            aerr << "Can't find service: " << services[i] << endl;
        }
    }

    return 0;
}
```

从上面的代码可以看到这个工具是通过 SM 获取 SS 的 IBinder 接口的。SM 主要是 native 的层，所以 native 应用要调用也挺方便。然后导出的内容主要是调用 IBinder 的 dump 接口。IBinder 直接留有 dump 这个抽象接口，所以从 Binder 设计开始，android 就想要要留个东西方便调试（查看当前 SS 的状态）：

```cpp
class IBinder : public virtual RefBase
{
public:

... ...

    virtual status_t        dump(int fd, const Vector<String16>& args) = 0;

... ...
}
```

然后 native 的 Bn 有默认实现：

```cpp
// =============== Binder.cpp ==================

status_t BBinder::dump(int fd, const Vector<String16>& args)
{
    return NO_ERROR;
}
```

接下去是 java 层的：

```java
// =============== IBinder.java ==================

public interface IBinder {

... ....

    /*
     * Print the object's state into the given stream.
     * 
     * @param fd The raw file descriptor that the dump is being sent to.
     * @param args additional arguments to the dump request.
     */
    public void dump(FileDescriptor fd, String[] args) throws RemoteException;

... ...

}

// =============== Binder.java ==================

public class Binder implements IBinder {

... ...

    /**
     * Implemented to call the more convenient version
     * {@link #dump(FileDescriptor, PrintWriter, String[])}.
     */
    public void dump(FileDescriptor fd, String[] args) {
        FileOutputStream fout = new FileOutputStream(fd);
        PrintWriter pw = new FastPrintWriter(fout);
        try {
            final String disabled;
            synchronized (Binder.class) {
                disabled = sDumpDisabled;
            }  
            if (disabled == null) {
                try {
                    dump(fd, pw, args);
                } catch (SecurityException e) {
                    pw.println("Security exception: " + e.getMessage());
                    throw e;
                } catch (Throwable e) {
                    // Unlike usual calls, in this case if an exception gets thrown
                    // back to us we want to print it back in to the dump data, since
                    // that is where the caller expects all interesting information to
                    // go.
                    pw.println();
                    pw.println("Exception occurred while dumping:");
                    e.printStackTrace(pw);
                }
            } else {
                pw.println(sDumpDisabled);
            }
        } finally {
            pw.flush();
        }
    }

    /*
     * Print the object's state into the given stream.
     * 
     * @param fd The raw file descriptor that the dump is being sent to.
     * @param fout The file to which you should dump your state.  This will be
     * closed for you after you return.
     * @param args additional arguments to the dump request.
     */
    protected void dump(FileDescriptor fd, PrintWriter fout, String[] args) {
    }

... ...

}
```

从 dumpsys 通过 native 的 IBinder 一直调用到 java 层的 Binder dump，到 java 的 Binder 有一个同名但是参数不一样的抽象函数。多了的一个参数是 java io 的 PrintWriter，我对这个不是很熟悉（好像是可以控制可以输出到哪去的），所以在这里不讨论这个。然后其实导出什么东西，具体是由不同的 SS 来实现的。我们拿 AMS 来看一下：

```java
// =============== ActivityManagerService.java ================== 

    @Override
    protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
        if (checkCallingPermission(android.Manifest.permission.DUMP)
                != PackageManager.PERMISSION_GRANTED) {
            pw.println("Permission Denial: can't dump ActivityManager from from pid="
                    + Binder.getCallingPid()
                    + ", uid=" + Binder.getCallingUid()
                    + " without permission "
                    + android.Manifest.permission.DUMP);
            return;
        }

        boolean dumpAll = false;
        boolean dumpClient = false;
        String dumpPackage = null;

        int opti = 0;
        while (opti < args.length) {
            String opt = args[opti];
            if (opt == null || opt.length() <= 0 || opt.charAt(0) != '-') {
                break;
            }
            opti++;
            if ("-a".equals(opt)) {
                dumpAll = true;
            } else if ("-c".equals(opt)) {
                dumpClient = true;
            } else if ("-h".equals(opt)) {
                pw.println("Activity manager dump options:");
                pw.println("  [-a] [-c] [-h] [cmd] ...");
                pw.println("  cmd may be one of:");
                pw.println("    a[ctivities]: activity stack state");
                pw.println("    b[roadcasts] [PACKAGE_NAME] [history [-s]]: broadcast state");
                pw.println("    i[ntents] [PACKAGE_NAME]: pending intent state");
                pw.println("    p[rocesses] [PACKAGE_NAME]: process state");
                pw.println("    o[om]: out of memory management");
                pw.println("    prov[iders] [COMP_SPEC ...]: content provider state");
                pw.println("    provider [COMP_SPEC]: provider client-side state");
                pw.println("    s[ervices] [COMP_SPEC ...]: service state");
                pw.println("    service [COMP_SPEC]: service client-side state");
                pw.println("    package [PACKAGE_NAME]: all state related to given package");
                pw.println("    all: dump all activities");
                pw.println("    top: dump the top activity");
                pw.println("  cmd may also be a COMP_SPEC to dump activities.");
                pw.println("  COMP_SPEC may be a component name (com.foo/.myApp),");
                pw.println("    a partial substring in a component name, a");
                pw.println("    hex object identifier.");
                pw.println("  -a: include all available server state.");
                pw.println("  -c: include client state.");
                return;
            } else {
                pw.println("Unknown argument: " + opt + "; use -h for help");
            }
        }

... ...

    }
```

这里我们不具体看 AMS 导出了什么东西（AMS 的数据结构一大票，所以能导出很多东西的），但是从代码中发现，其实 dumpsys 还可以传不同的参数给不同的 SS。例如上面的 AMS 有 -c、-h 的参数。从上面 dumpsys 代码中发现，只有指定 SS 的命令格式才能接特定的参数，例如说这样:

```bash
dumpsys activity -c
```

全部导出的不支持的（默认给你加了 -a），这个是因为不同的 SS 说支持的参数不一样（完全由各个 SS 实现决定），所以只有指定 SS 才能接参数。至于哪些 SS 支持哪些参数，具体的去看 SS 的 dump 实现代码去吧。然后说一点如果自己改某些 SS ，例如说加了某些字段，可以在 dump 中把这些字段也导出来了，这样调试自己加的功能的时候会方便不少。

## 总结

dumpsys 这个工具最大的作用在于，可以直接打印（导出到文件）所有（指定）SS 的一些预先写好的状态，对于调某些问题的时候，可以通过这个工具查看到当前 SS 中一些数值的信息，这样可以不用额外加打印就能追踪到一些现象中的 SS 的某些状态。所以对于某些重启现象就消失的情况，这个工具是非常管用的。




