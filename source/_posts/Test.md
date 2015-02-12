title: Test
date: 2015-01-10 11:07:16
updated: 2015-01-10 11:07:16
tags: []
---

Welcome to [Hexo](http://hexo.io/)! This is your very first post. Check [documentation](http://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [trobuleshooting](http://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### my test

An activity is a single, focused thing that the user can do. **Almost all activities interact with the user**, so the Activity class takes care of creating a window for you in which you can place your UI with setContentView(View). While activities are often presented to the user as full-screen windows, they can also be used in other ways: as floating windows (via a theme with windowIsFloating set) or embedded inside of another activity (using ActivityGroup). There are <font color="#ff0000">two methods</font> almost all subclasses of Activity will implement:

* onCreate(Bundle) is where you initialize your activity. Most importantly, here you will usually call setContentView(int) with a layout resource defining your UI, and using findViewById(int) to retrieve the widgets in that UI that you need to interact with programmatically.
* onPause() is where you deal with the user leaving your activity. Most importantly, any changes made by the user should at this point be committed (usually to the ContentProvider holding the data). 

To be of use with Context.startActivity(), all activity classes must have a corresponding `<activity>` declaration in their package's AndroidManifest.xml.The Activity class is an important part of an application's overall lifecycle, and the way activities are launched and put together is a fundamental part of the platform's application model. For a detailed perspective on the structure of an Android application and how activities behave, please read the Application Fundamentals and Tasks and Back Stack developer guides.


**垂直同步**又称场同步（Vertical Hold），从CRT显示器的显示原理来看，单个象素组成了水平扫描线，水平扫描线在垂直方向的堆积形成了完整的画面。显示器的刷新率受显卡DAC控制，显卡DAC完成一帧的扫描后就会产生一个垂直同步信号。我们平时所说的打开垂直同步指的是将该信号送入显卡3D图形处理部分，从而让显卡在生成3D图形时受垂直同步信号的制约。主要区别在于那些高速运行的游戏，比如实况，FPS游戏，打开后能防止游戏画面高速移动时画面撕裂现象，当然打开后如果你的游戏画面FPS数能达到或超过你显示器的刷新率，这时你的游戏画面FPS数被限制为你显示器的刷新率。你会觉得原来移动时的游戏画面是如此舒服，如果达不到会出现不同程度的跳帧现象，FPS与刷新率差距越大跳帧越严重。关闭后除高速运动的游戏外其他游戏基本看不出<font color="#ff0000">画面撕裂现象</font>。


This code hightlight:

**cpp:**

```cpp
#include "DisplayHardware/HWComposer.h"
#include "Effects/Daltonizer.h"

namespace android {

// ---------------------------------------------

class Client;
class DisplayEventConnection;
class EventThread;
class IGraphicBufferAlloc;
class Layer;
class LayerDim;
class Surface;
class RenderEngine;
class EventControlThread;

enum {
    eTransactionNeeded        = 0x01,
    eTraversalNeeded          = 0x02,
    eDisplayTransactionNeeded = 0x04,
    eTransactionMask          = 0x07
};

enum
{
    /* NOTE: These enums are unknown to Android.
     * Android only checks against HWC_FRAMEBUFFER.
     * This layer is to be drawn into the framebuffer by hwc blitter */
    //HWC_TOWIN0 = 0x10,
    //HWC_TOWIN1,
    HWC_BLITTER = 100,
    HWC_DIM,
    HWC_CLEAR_HOLE
};

class SurfaceFlinger : public BnSurfaceComposer,
                       private IBinder::DeathRecipient,
                       private HWComposer::EventHandler
{
public:
    static char const* getServiceName() ANDROID_API {
        return "SurfaceFlinger";
    }

    SurfaceFlinger() ANDROID_API;

    // must be called before clients can connect
    void init() ANDROID_API;

    Daltonizer mDaltonizer;
    bool mDaltonize;
    int mDebugFPS;
    
    // add by rk for workwround some display issue. 
    int mSkipFlag;
    int mDelayFlag;
};


SurfaceFlinger::SurfaceFlinger()
    :   BnSurfaceComposer(),
        mTransactionFlags(0),
        mTransactionPending(false),
        mAnimTransactionPending(false),
        mLayersRemoved(false),
        mRepaintEverything(0),
        mRenderEngine(NULL),
        mBootTime(systemTime()),
        mVisibleRegionsDirty(false),
        mHwWorkListDirty(false),
        mAnimCompositionPending(false),
        mDebugRegion(0),
        mDebugDDMS(0),
        mDebugDisableHWC(0),
        mDebugDisableTransformHint(0),
        mDebugInSwapBuffers(0),
        mLastSwapBufferTime(0),
        mDebugInTransaction(0),
        mLastTransactionTime(0),
        mBootFinished(false),
        mPrimaryHWVsyncEnabled(false),
        mHWVsyncAvailable(false),
        mDaltonize(false),
        mHardwareOrientation(0),
        mUseLcdcComposer(0),
        mSkipFlag(0),
        mDelayFlag(0)
{
    ALOGI("SurfaceFlinger is starting");

    // debugging stuff...
    char value[PROPERTY_VALUE_MAX];

    property_get("ro.bq.gpu_to_cpu_unsupported", value, "0");
    mGpuToCpuSupported = !atoi(value);

    property_get("debug.sf.showupdates", value, "0");
    mDebugRegion = atoi(value);

    property_get("debug.sf.ddms", value, "0");
    mDebugDDMS = atoi(value);
    if (mDebugDDMS) {
        if (!startDdmConnection()) {
            // start failed, and DDMS debugging not enabled
            mDebugDDMS = 0;
        }
    }
    property_get("debug.sf.fps", value, "0");
    mDebugFPS = atoi(value);

    ALOGI_IF(mDebugRegion, "showupdates enabled");
    ALOGI_IF(mDebugDDMS, "DDMS debugging enabled");

    // hardware rotation
    property_get("ro.sf.hwrotation", value, "0");
    mHardwareOrientation = atoi(value) / 90;
    property_get("ro.sf.lcdc_composer", value, "0");
    mUseLcdcComposer = atoi(value);
}

/**
 * List of all active broadcasts that are to be executed one at a time.
 * The object at the top of the list is the currently activity broadcasts;
 * those after it are waiting for the top to finish.  As with parallel
 * broadcasts, separate background- and foreground-priority queues are
 * maintained.
 */
void SurfaceFlinger::destroyDisplay(const sp<IBinder>& display) {
    Mutex::Autolock _l(mStateLock);

    ssize_t idx = mCurrentState.displays.indexOfKey(display);
    if (idx < 0) {
        ALOGW("destroyDisplay: invalid display token");
        return;
    }

    const DisplayDeviceState& info(mCurrentState.displays.valueAt(idx));
    if (!info.isVirtualDisplay()) {
        ALOGE("destroyDisplay called for non-virtual display");
        return;
    }

    mCurrentState.displays.removeItemsAt(idx);
    setTransactionFlags(eDisplayTransactionNeeded);
}
```

**java:**

```java
import java.util.Set;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicLong;

/*
 * Copyright (C) 2006-2008 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    private static final String USER_DATA_DIR = "/data/user/"; 
    static final String TAG = "ActivityManager";
    static final String TAG_MU = "ActivityManagerServiceMU";
    static final boolean DEBUG = false;
    static final boolean localLOGV = DEBUG;
    static final boolean DEBUG_SWITCH = localLOGV || false;

    // Control over CPU and battery monitoring.
    static final long BATTERY_STATS_TIME = 30*60*1000;      // write battery stats every 30 minutes.
    static final boolean MONITOR_CPU_USAGE = true;
    static final long MONITOR_CPU_MIN_TIME = 5*1000;        // don't sample cpu less than every 5 seconds.
    static final long MONITOR_CPU_MAX_TIME = 0x0fffffff;    // wait possibly forever for next cpu sample.
    static final boolean MONITOR_THREAD_CPU_USAGE = false;

    /*
     * java mulit-line comment should use on *
     * otherwise can't render correct.
     */

    /**
     * Description of a request to start a new activity, which has been held
     * due to app switches being disabled.
     */
    static class PendingActivityLaunch {
        ActivityRecord r;
        ActivityRecord sourceRecord;
        int startFlags;
    }

    BroadcastQueue broadcastQueueForIntent(Intent intent) {
        final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
        if (DEBUG_BACKGROUND_BROADCAST) {
            Slog.i(TAG, "Broadcast intent " + intent + " on "
                    + (isFg ? "foreground" : "background")
                    + " queue");
        }
        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    }
}
```

**bash show as code file list:**

```bash
# java 层 Context 相关接口
frameworks/base/core/java/android/app/ContextImpl.java
frameworks/base/core/java/android/app/LoadedApk.java
frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/core/java/android/content/ServiceConnection

# java 层 Service 相关接口
frameworks/base/core/java/android/app/IActivityManager.java
frameworks/base/core/java/android/app/ActivityManager.java
frameworks/base/core/java/android/app/ActivityManagerNative.java
frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/java/com/android/server/am/ActiveServices.java
frameworks/base/services/java/com/android/server/am/ServiceRecord.java
```

**xml:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- Copyright (C) 2006 The Android Open Source Project

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at
  
          http://www.apache.org/licenses/LICENSE-2.0
  
     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License.
-->

<!-- Layout for a Preference in a PreferenceActivity. The
     Preference is able to place a specific widget for its particular
     type in the "widget_frame" layout. -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android" 
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:minHeight="?android:attr/listPreferredItemHeight"
    android:gravity="center_vertical"
    android:paddingRight="?android:attr/scrollbarSize">
    
    <RelativeLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="15dip"
        android:layout_marginRight="6dip"
        android:layout_marginTop="6dip"
        android:layout_marginBottom="6dip"
        android:layout_weight="1">
    
        <TextView android:id="@+android:id/title"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:singleLine="true"
            android:textAppearance="?android:attr/textAppearanceLarge"
            android:ellipsize="marquee"
            android:fadingEdge="horizontal" />
            
        // 就是在这里改啦，加一个 textColor 属性
        <TextView android:id="@+android:id/summary"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_below="@android:id/title"
            android:layout_alignLeft="@android:id/title"
            android:textAppearance="?android:attr/textAppearanceSmall"
            android:textColor="#FFFF0000" 
            android:maxLines="4" />

    </RelativeLayout>
    
    <!-- Preference should place its actual preference widget here. -->
    <LinearLayout android:id="@+android:id/widget_frame"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:gravity="center_vertical"
        android:orientation="vertical" />

</LinearLayout>
```

**markdown origin table:** (en, the origin table format is render well)

| API(left) | 存储(center) | 区域 | 特性(right) |
|:----|:--------:|:----:|----:|
| GetMemDC | on-screen | 指定大小 | 我就是要写长一点一点一点点，哈哈 |
| CreateSubMemDC | off-screen | 指定大小 | 短一点 |


**html table:** (en, the parser render bug ... ...)

---

<table border="1">
<tr>
  <th align="center">API</th>
  <th align="center">存储位置</th>
  <th align="center">区域</th>
  <th align="center">特性</th>
</tr>
<tr>
  <td align="left">CreateMemDC</td>
  <td align="center">off-screen</td>
  <td align="center">指定大小</td>
  <td align="center">我就是要写长一点一点一点点，哈哈</td>
</tr>
<tr>
  <td align="left">CreateSubMemDC</td>
  <td align="center">off-screen</td>
  <td align="center">指定大小</td>
  <td align="center">短一点</td>
</tr>
</table>

---

manually fix it:

<table border="1"><tr><th align="center">API</th><th align="center">存储位置</th><th align="center">区域</th><th align="center">特性</th></tr><tr>  <td align="left">CreateMemDC</td><td align="center">off-screen</td><td align="center">指定大小</td><td align="center">我就是要写长一点一点一点点，哈哈</td>
</tr><tr><td align="left">CreateSubMemDC</td><td align="center">off-screen</td><td align="center">指定大小</td><td align="center">短一点</td></tr></table>

My test end

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](http://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](http://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](http://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](http://hexo.io/docs/deployment.html)
