title: hierarchyviewer 无法使用问题
date: 2015-01-26 23:39:16
updated: 2015-01-26 23:39:16
categories: [Android Development]
tags: [android]
---

hierarchyviewer 这个 sdk tools 中带了一个工具，我就不多介绍了，调试 layout 神器。但是正常情况下只有在 eng 版本的机器上才能使用。

## 概述

在非 eng 机器上是无法使用的。hierarchyviewer 需要在 framework 中启用一个 view server ，然后通过 socket 和 host 上的 hierarchyviewer 进行通信。

```java
// frameworks/base/services/java/com/android/server/wm/WindowManagerService.java

    /**
     * Starts the view server on the specified port.
     *
     * @param port The port to listener to.
     *
     * @return True if the server was successfully started, false otherwise.
     *
     * @see com.android.server.wm.ViewServer
     * @see com.android.server.wm.ViewServer#VIEW_SERVER_DEFAULT_PORT
     */
    public boolean startViewServer(int port) {
        if (isSystemSecure()) {
            return false;
        }     

        if (!checkCallingPermission(Manifest.permission.DUMP, "startViewServer")) {
            return false;
        }     

        if (port < 1024) {
            return false;
        }     

        if (mViewServer != null) {
            if (!mViewServer.isRunning()) {
                try { 
                    return mViewServer.start();
                } catch (IOException e) {
                    Slog.w(TAG, "View server did not start");
                }     
            }     
            return false;
        }     

        try { 
            mViewServer = new ViewServer(this, port);
            return mViewServer.start();
        } catch (IOException e) {
            Slog.w(TAG, "View server did not start");
        }     
        return false;
    }

    private boolean isSystemSecure() {
        return "1".equals(SystemProperties.get(SYSTEM_SECURE, "1")) &&
                "0".equals(SystemProperties.get(SYSTEM_DEBUGGABLE, "0"));
    }
```

通过代码可以发现，只要 ro.secure = 0 或是 ro.debuggable = 1 就可以使用 hierarchyviewer。一般发布出去的 android 设备， ro.secure 普遍都是 1， ro.debuggable 大多数是 0，但是也有些的 ro.debuggable 是 1 来的。这2个值可以通过 adb shell 之后 getprop 查看。

网上有讨论怎么在 OEM 的设置上通过 hack（其实就是反编译 services.jar，然后修改 services 中的 smali 字节码，然后修改里面的 isSystemSecure 的返回值）来开启 hierarchyviewer 的支持的。详细看这里：

[如何在Root的手机上开启ViewServer，使得HierachyViewer能够连接](http://maider.blog.sohu.com/255448342.html "如何在Root的手机上开启ViewServer，使得HierachyViewer能够连接") 

## 问题

但是这里不是要讨论 hack 的问题。而是我本来 eng 版本的机子上，突然 hierarchyviewer 就用不了。在 host 上运行 hierarchyviewer 打印就出:

<pre>
10:13:25 E/hierarchyviewer: Unable to get view server version from device xxxxx
10:13:25 E/hierarchyviewer: Unable to get view server protocol version from device xxxxx
10:13:25 E/ViewServerDevice: Unable to debug device: xxxxx
10:13:25 E/hierarchyviewer: Missing forwarded port for xxxx
10:13:25 E/hierarchyviewer: Unable to get the focused window from device xxxx
</pre>

然后按照的说法敲命令：

<pre>
adb shell service call window 3
</pre>

输出如果是：

<pre>
Result: Parcel(00000000 00000000 '........')" 
</pre>

说明 view server 处于关闭状态。输出如果是：

<pre>
Result: Parcel(00000000 00000001 '........')" 
</pre>

说明 view server 处于开启状态。我在我的 eng 的机子试了下，发现输出是 00000000。奇怪了，之前还是能用的，怎么突然 view server 启不来啦？启动代码在上面有，我发现启动 view server 那里有 try catch，然后有句简单的打印。取出来 log 看了下，发现确实有打印： View server did not start。但是也看不出来哪里出错了。

于是我在 try catch 那加了句： e.printStackTrace() 。然后发现了异常：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/hierarchyviewer-problem/1.png)

唉，这咋是 unable to resolve “localhost" 咧？哦，我突然想起来，最近搞屏蔽应用内广告。然后借鉴了一种简单的方法：就是修改 hosts 文件。然后我在调试的时候把原来的 /system/etc/hosts 给删掉了。原来这个文件中就只有一条 host：

<pre>
127.0.0.1 localhosts
</pre>

哦，我终于知道原来这句话是用来干什么的了，原来是给 view server 用的啊。它在创建 socket 的时候直接写 localhosts 的。重新把这段加到 /system/etc/hosts 里面就好了（如果删掉了，自己创建一个就行了）。

哎呀，hierarchyviewer 又能用啦，舒服～～

