title: Android 一些有意思的命令小工具 —— service
date: 2015-01-30 16:32:16
categories: [Android Framework]
tags: [android]
---

android 自带了很多 cmds 的小工具，都挺用有的，也有些比较有意思。这里介绍的是 service 这个工具。说之前先把相关源码位置说下（4.4 的）：

```bash
# service 命令模块
frameworks/native/cmds/service
```

## 作用

这个工具的作用是让你能够在 shell 中调用 android system services 的一部分接口。至于怎么调用，看它的 usage 就清除了：

<pre>
Usage: service [-h|-?]
       service list
       service check SERVICE
       service call SERVICE CODE [i32 INT | s16 STR] ...
Options:
   i32: Write the integer INT into the send parcel.
   s16: Write the UTF-16 string STR into the send parcel.
</pre>

它的用法就是 service 命令 + 参数。service 的命令很少一般有用的就2个：

### list

列出当前系统有所有的 system service :

<pre>
ming ~ $ adb shell service list
Found 73 services:
0       media_router: [android.media.IMediaRouterService]
1       print: [android.print.IPrintManager]
2       assetatlas: [android.view.IAssetAtlas]
3       dreams: [android.service.dreams.IDreamManager]
4       commontime_management: []
5       samplingprofiler: []
6       diskstats: []
7       appwidget: [com.android.internal.appwidget.IAppWidgetService]
8       backup: [android.app.backup.IBackupManager]
9       uimode: [android.app.IUiModeManager]
10      serial: [android.hardware.ISerialManager]
11      usb: [android.hardware.usb.IUsbManager]
12      audio: [android.media.IAudioService]
13      wallpaper: [android.app.IWallpaperManager]
14      dropbox: [com.android.internal.os.IDropBoxManagerService]
15      search: [android.app.ISearchManager]
16      country_detector: [android.location.ICountryDetector]
17      location: [android.location.ILocationManager]
18      devicestoragemonitor: []
19      notification: [android.app.INotificationManager]
20      updatelock: [android.os.IUpdateLock]
21      servicediscovery: [android.net.nsd.INsdManager]
22      ethernet: [android.net.ethernet.IEthernetManager]
23      connectivity: [android.net.IConnectivityManager]
24      wifi: [android.net.wifi.IWifiManager]
25      wifip2p: [android.net.wifi.p2p.IWifiP2pManager]
26      netpolicy: [android.net.INetworkPolicyManager]
27      netstats: [android.net.INetworkStatsService]
28      textservices: [com.android.internal.textservice.ITextServicesManager]
29      network_management: [android.os.INetworkManagementService]
30      clipboard: [android.content.IClipboard]
31      statusbar: [com.android.internal.statusbar.IStatusBarService]
32      device_policy: [android.app.admin.IDevicePolicyManager]
33      lock_settings: [com.android.internal.widget.ILockSettings]
34      mount: [IMountService]
35      accessibility: [android.view.accessibility.IAccessibilityManager]
36      input_method: [com.android.internal.view.IInputMethodManager]
37      bluetooth_manager: [android.bluetooth.IBluetoothManager]
38      input: [android.hardware.input.IInputManager]
39      window: [android.view.IWindowManager]
40      alarm: [android.app.IAlarmManager]
41      consumer_ir: [android.hardware.IConsumerIrService]
42      vibrator: [android.os.IVibratorService]
43      battery: []
44      hardware: [android.os.IHardwareService]
45      content: [android.content.IContentService]
46      account: [android.accounts.IAccountManager]
47      user: [android.os.IUserManager]
48      entropy: []
49      permission: [android.os.IPermissionController]
50      cpuinfo: []
51      dbinfo: []
52      gfxinfo: []

... ...
</pre>

### call

这个是主要功能，就是调用指定 service 的接口。具体的在下面的源码分析说一边分析一边说。

## 分析

这个工具的代码其实很少，只有一个 service.cpp 文件，而且只有 300 行不到。我们慢慢看，首先 mian 函数开头：

```cpp
int main(int argc, char* const argv[])
{
    // 取 SM 
    sp<IServiceManager> sm = defaultServiceManager();
    fflush(stdout);
    if (sm == NULL) {
        aerr << "service: Unable to get default service manager!" << endl;
        return 20; 
    }
        
    bool wantsUsage = false;
    int result = 0;
    
    // 敲 service -h 或是 service -? 是出前面的 usage 提示
    while (1) {
        int ic = getopt(argc, argv, "h?");
        if (ic < 0)
            break;

        switch (ic) {
        case 'h':
        case '?':
            wantsUsage = true;
            break;
        default:
            aerr << "service: Unknown option -" << ic << endl;
            wantsUsage = true;
            result = 10; 
            break;
        }   
    } 

... ...

}

```

### check

接下来就有一个 check 的命令：

```cpp
... ...

    if (optind >= argc) {
        wantsUsage = true;
    } else if (!wantsUsage) {
        if (strcmp(argv[optind], "check") == 0) {
            optind++;
            if (optind < argc) {
                // 是调 SM 的 checkService 接口
                sp<IBinder> service = sm->checkService(String16(argv[optind]));
                aout << "Service " << argv[optind] <<
                    (service == NULL ? ": not found" : ": found") << endl;
            } else {
                aerr << "service: No service specified for check" << endl;
                wantsUsage = true;
                result = 10;
            }
        }

... ...
            
        } else {
            aerr << "service: Unknown command " << argv[optind] << endl;
            wantsUsage = true;
            result = 10;
        }
    }

... ...
```

这个 check 命名就是调用 SM 的 checkService 接口。至于这个接口是干啥的，[看这里](http://light3moon.com/2015/01/28/Android Binder 分析——系统服务 Binder 对象的传递 "看这里")。不过不知道是不是权限原因，这里从 SM 返回好像都是 NULL，然后老是打印 Service xx not found 。我没去深究，因为这个命令好像也没啥用。

### list

下面继续：

```cpp
... ...

        }
        else if (strcmp(argv[optind], "list") == 0) {
            Vector<String16> services = sm->listServices();
            aout << "Found " << services.size() << " services:" << endl;
            for (unsigned i = 0; i < services.size(); i++) {
                String16 name = services[i];
                sp<IBinder> service = sm->checkService(name);
                aout << i
                     << "\t" << good_old_string(name)
                     << ": [" << good_old_string(get_interface_name(service)) << "]"
                     << endl;
            }
        } else if (strcmp(argv[optind], "call") == 0) {

... ...
```

这个 list 好像也是 SM 的命令来的。SM 会返回一个当前注册的 system service 的列表过来，然后这边一个 for 循环打印这些服务的信息。哎，这里的 checkService 又 OK 了。可能是字符串传递的问题吧。我们来看下另外2个小函数：

```cpp
// 这个好像是字符串转化相关的，String8 和 String16 都是 android 在 native 
// 对 char 串的封装
static String8 good_old_string(const String16& src)
{
    String8 name8;
    char ch8[2];
    ch8[1] = 0;
    for (unsigned j = 0; j < src.size(); j++) { 
        char16_t ch = src[j];
        if (ch < 128) ch8[0] = (char)ch;
        name8.append(ch8);
    }
    return name8;
}
```

然后这个 `get_interface_name` 就是上面后面 [] 打印的东西，这个是每个 service 的标示，之前 binder 几篇的文章有说到每次 binder 通信，首先会检测这个标示的。

```cpp
// get the name of the generic interface we hold a reference to
static String16 get_interface_name(sp<IBinder> service)
{
    if (service != NULL) {
        Parcel data, reply;
        status_t err = service->transact(IBinder::INTERFACE_TRANSACTION, data, &reply);              
        if (err == NO_ERROR) {
            return reply.readString16();                                                                                  
        }
    }
    return String16();
}
```

然后在 Binder.cpp 中，就是 binder Bn 的父类中默认处理中：

```cpp
status_t BBinder::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch (code) {
        case INTERFACE_TRANSACTION:
            reply->writeString16(getInterfaceDescriptor());
            return NO_ERROR;

... ...

        default:
            return UNKNOWN_TRANSACTION;
    }
}
```

这就获取每个 system service 的标示。

### call

接下来这个才是最有用的命令：

```cpp
... ...
        
        } else if (strcmp(argv[optind], "call") == 0) {
            optind++;
            if (optind+1 < argc) {
                int serviceArg = optind;
                // 从 sm 那获取指定的 service 接口（Bp）
                sp<IBinder> service = sm->checkService(String16(argv[optind++]));
                // 获取服务的标示
                String16 ifName = get_interface_name(service);
                int32_t code = atoi(argv[optind++]);
                if (service != NULL && ifName.size() > 0) {
                    Parcel data, reply;

                    // 呵呵，先写服务的标示
                    // the interface name is first
                    data.writeInterfaceToken(ifName);

                    // 支持的参数有3种类型：i32、s16 和 intent
                    // then the rest of the call arguments
                    while (optind < argc) {
                        // i32 就是 int32，非常简单的类型，
                        // 直接使用 parcel 的 writeInt32 
                        if (strcmp(argv[optind], "i32") == 0) {
                            optind++;
                            if (optind >= argc) {
                                aerr << "service: no integer supplied for 'i32'" << endl;
                                wantsUsage = true;
                                result = 10;
                                break;
                            }
                            data.writeInt32(atoi(argv[optind++]));
                        } else if (strcmp(argv[optind], "s16") == 0) {
                            // s16 就是 String16，也是比较简单的类型
                            // parcel 都有现成的接口可用
                            optind++;
                            if (optind >= argc) {
                                aerr << "service: no string supplied for 's16'" << endl;
                                wantsUsage = true;
                                result = 10;
                                break;
                            }
                            data.writeString16(String16(argv[optind++]));
                        } else if (strcmp(argv[optind], "null") == 0) {
                            optind++;
                            data.writeStrongBinder(NULL);
                        } else if (strcmp(argv[optind], "intent") == 0) {
       
// 下面这个 intent 类型，就有点麻烦了，我暂时没去管它 ... ...
... ...

                        } else {
                            aerr << "service: unknown option " << argv[optind] << endl;
                            wantsUsage = true;
                            result = 10;
                            break;
                        }
                    }

                    // parcel 打好包好，调用 service 的 transact 函数
                    service->transact(code, data, &reply);
                    // 打印 parcel 打包的返回值
                    aout << "Result: " << reply << endl;
                } else {
                    aerr << "service: Service " << argv[serviceArg]
                        << " does not exist" << endl;
                    result = 10;
                }
            } else {

... ...
```

从上面的代码来看，call 的格式是：

<pre>
service call service code i32 xx
</pre>

service 就是当初 SM addService 的那串字符，就是 list 中第一个名字。例如 WindowManagerService 的就是 window。code 的话就是定义 service 接口的时候那从 `android.os.IBinder.FIRST_CALL_TRANSACTION` 开始累加的数字。然后后面就是参数了，开始要把参数类型写上，然后才是参数。上面说了一共支持3种类型：i32（int32）、s16（String16）和 intent（复合类型）。


来点例子，例如我那篇 [hierarchyviewer 无法使用问题](http://light3moon.com/2015/01/26/hierarchyviewer 无法使用问题 "hierarchyviewer 无法使用问题") 调用 wm 开启 view server 就是这么敲：

<pre>
service call window 1 i32 4939
</pre>

这个 `TRANSACTION_CODE` 怎么确定咧。这个就要去翻源码啦。例如上面 wm 的那个接口，我在 IWindwManager.java（wm 是用 aidl 的，要去 out 下面去翻 aidl 自动生成的 java 文件） 中：

```java
static final int TRANSACTION_startViewServer = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
static final int TRANSACTION_stopViewServer = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
static final int TRANSACTION_isViewServerRunning = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);

// 然后去翻下 IBinder 发现是从 1 开始的 
    enum {
        FIRST_CALL_TRANSACTION  = 0x00000001,
        LAST_CALL_TRANSACTION   = 0x00ffffff,
... ...
    };
```

所以要调用 wm 的 startViewServer 接口，code 就要填 1 啦。然后我们再去看下 startViewServer 的接口：

```java
    /*
     * Starts the view server on the specified port.
     *
     * @param port The port to listener to.
     *
     * @return True if the server was successfully started, false otherwise.
     *
     * @see com.android.server.wm.ViewServer
     * @see com.android.server.wm.ViewServer#VIEW_SERVER_DEFAULT_PORT
     */
    @Override
    public boolean startViewServer(int port) 
```

接口参数就一个 int 的 port（socket 的端口吧），所以后面那个参数就要填 i32 4939 （话说网上怎么知道端口要用 4939 的）。

然后我们来看下 IWindowManager.aidl 中一段有意思的注释：

```cpp
    /**
     * ===== NOTICE =====
     * The first three methods must remain the first three methods. Scripts
     * and tools rely on their transaction number to work properly.
     */ 
    // This is used for debugging
    boolean startViewServer(int port);   // Transaction #1
    boolean stopViewServer();            // Transaction #2
    boolean isViewServerRunning();       // Transaction #3
```

大致意思是说：请把这2个接口放在最前面声明，因为 aidl 是按顺序生成 code 号的，这样就能保证这3个接口的 code 固定是 1、2、3。这个明显是 android 故意留给这个 shell 命令的小工具的。所以你要自己要加一些 service 的接口，尽量从 `LAST_CALL_TRANSACTION` 从后往前面加，避免影响一些系统工具的使用。


然后如果是没有参数的话，后面那个参数可以不填，例如查询 wm view server 是否开启的接口：

<pre>
service call window 3
</pre>

然后就是返回值了。binder 的返回值都是通过 parcel 打包的，前面 binder 分析过了，每个服务的 Bp 都会自己解析返回值的 parcel 包，然后里面的返回值拆出来，转化成接口具体的返回值返回给调用者。但是这里就直接打印 parcel 了。前面 binder 也分析过，parcel 就是简单的把数据塞到内存里面而已（当然有一些类型有一点点格式）。这里也只是把 parcel 内存中的东西打印出来而已。

我们来看上面那个查询 wm view server 是否开启的结果：

<pre>
root@H9:/ # service call window 3
Result: Parcel(00000000 00000001   '........')
</pre>

返回值，第一个 int32 是错误值，0 表示没有错误。然后从第二开始才是真正的返回值。这个 startViewServer 的返回值看代码就是一个 int（那个接口是 boolean，在 Bp 那是根据 0、1 来判断的）。所以那个 00000001 就代表 view server 已经启动咯。如果是 00000000 就是没启动。


好，我们再来个稍微复杂点的例子。wm 中有一个这个接口：

```java
    @Override
    public float getAnimationScale(int which) { 
        switch (which) {
            case 0: return mWindowAnimationScale;
            case 1: return mTransitionAnimationScale;
            case 2: return mAnimatorDurationScale; 
        }
        return 0;
    }
```

而且这个正好在 Settings 中的开发这选项有可以设置（可以拿来测试用）。开发者选项-->绘图：

<pre>
窗口动画缩放    --> mWindowAnimationScale
过渡动画缩放    --> mTransitionAnimationScale
动画程序时长缩放 --> mAnimatorDurationScale
</pre>

我们拿第一个玩一下，因为第一个窗口动画缩放，只要设置成 10x 窗口的缩放动画就会变得很慢。去翻下 IWindowManager.java 4.4 的是：

```java
static final int TRANSACTION_getAnimationScale = (android.os.IBinder.FIRST_CALL_TRANSACTION + 48);
```

所以要获取 mWindowAnimationScale 的值的话就应该这么敲：

<pre>
service call window 49 i32 0
</pre>

得到的结果如下：

<pre>
root@H9:/ # service call window 49 i32 0
Result: Parcel(00000000 3f800000   '.......?')
</pre>

嘿嘿，返回的是 float，顺带学习下 float 在内存中的表示：[看这里啦](http://light3moon.com/2015/01/19/[转] float 类型在内存中的表示 "看这里啦")。

0x3f800000 就是 1.0 的 float。然后去 Settings 的调试选项，把这个调成 动画缩放 10x，再敲一下，结果是：

<pre>
root@H9:/ # service call window 49 i32 0
Result: Parcel(00000000 41200000   '...... A')
</pre>

0x41200000 是 10.0 的 float。

## 限制

从上面来看，这个小工具其实挺有用的。可以很方便的自己留个小后门或是加一些调试小接口。这个东西，需要知道接口的 `TRANSACTION_CODE` 和参数才能正常调用，很符合留后门给自己用的风范 ^_^。


还有从上面的分析来看，这个工具只能调用一些参数是 void、int、String 或是 intent 相关的接口。还有返回值如果是比较复杂的 class（结构体）要分析也比较困难，简单的 int、float、String 还好看懂。不过虽然有局限性，还是一个不错的小工具了。


