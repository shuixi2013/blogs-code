title: Android SystemUI 分析——状态栏图标
date: 2015-03-04 10:20:16
updated: 2015-03-04 10:20:16
categories: [Android Framework]
tags: [android]
---

通知篇说了一下有关 Notification 相关的东西，其中 Notification 有一个 UI 表现就是状态栏的图标，不过这篇说的是另外一种东西，虽然 UI 表现上和 Notification 的状态栏图标一样，但是使用上是不一样的。还是先照例把相关代码位置啰嗦一下（4.2.2）：

```bash
# 应用接口相关代码
frameworks/base/core/java/android/app/StatusBarManager.java

# 系统内部接口相关代码
frameworks/base/core/java/com/android/internal/statusbar/IStatusBarService.aidl
frameworks/base/core/java/com/android/internal/statusbar/IStatusBar.aidl
frameworks/base/core/java/com/android/internal/statusbar/StatusBarIcon.java

# System Server 相关代码
frameworks/base/services/java/com/android/server/StatusBarManagerService.java

# SystemUI 相关代码
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/StatusBarIconView.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/BaseStatusBar.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/CommandQueue.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/PhoneStatusBar.java
```

## 区分

我们先来看一张图：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/SystemUI-status-bar-icon/status-icon.png)

图中状态栏左边那一堆图标是通知篇说的 Notification 的小图标，右边那一堆就是这篇要说的系统状态图标。虽然这篇的名字是状态栏图标，但是因为通知篇说过 Notification 的了，所以这里只说系统的状态图标。通知的图标是每一个通知一个，然后如果通知太多的话，后面会显示一个 "+" 的图标代表有更多的通知图标，所以说左边通知的图标是不确定的，是由运行不同的程序来决定的。而系统状态图标是固定的，在 framework 中代码写死一共有多少种系统状态，然后系统运行不同功能的时候显示某个状态的图标，完成后，图标消失。例如说，设置（关闭）闹钟、开启（关闭）飞行模式、开启（关闭）静音。这里稍微注意下，只有图上右边框出来的部分（只是一小部分而已）才是系统状态，wifi、电量信息和时间不属于系统状态图标的。

## 接口

我们先来看系统状态图标的接口，接口依旧在 StatusBarManager（SBM） 中：

```java
/* @hide */
interface IStatusBarService
{
... ...

    void setIcon(String slot, String iconPackage, int iconId, int iconLevel, String contentDescription);
    void setIconVisibility(String slot, boolean visible);
    void removeIcon(String slot);

... ...
}
```

对外接口很简单就3个：

* **setIcon**: 设置一个状态图标，主要参数是图标
* **setIconVisibility**: 设置一个状态图标显示（隐藏）
* **removeIcon**: 移除一个状态图标

值得一提的是，这些接口都是系统内部使用的，因为整个 SBM 都是 hide 的（aidl、java 都是）。应用只能通过一些别的方法间接的去改变系统状态图标，无法直接更改。例如要让系统状态显示设置了闹钟，就要发送 android.intent.action.ALARM_CHANGED 广播（SystemUI 中有接收这个广播的地方，然后调用上面的接口来改变系统状态图标显示）。

这个设计还是必须的，因为如果随便开放接口给第三方应用直接改系统状态显示，有些捣乱的应用就要乱来了。不过虽然是 hide 的，java 有反射可以干坏事，不过下面会说到，接口里面权限检测的，所以反射也干不了坏事。

## 固定的数据

看完上面接口，是不是觉得 setIcon 为什么不叫 addIcon，来看下系统的实现（设计）就明白了。我们直接去 StatusBarManagerService（SBMS）里去看，不看 Bp 端的：

```java
    // slot 是标示
    public void setIcon(String slot, String iconPackage, int iconId, int iconLevel,
            String contentDescription) {
        // 检测调用者的权限
        enforceStatusBar();

        // 喜闻乐见的 binder 业务函数同步锁
        synchronized (mIcons) {
            // 通过 slot 来查找 status bar icon 的数据索引
            int index = mIcons.getSlotIndex(slot);
            // 找不到的话，报异常
            if (index < 0) {
                throw new SecurityException("invalid status bar icon slot: " + slot);
            }   

            // new 数据对象
            StatusBarIcon icon = new StatusBarIcon(iconPackage, UserHandle.OWNER, iconId,
                    iconLevel, 0,
                    contentDescription);
            //Slog.d(TAG, "setIcon slot=" + slot + " index=" + index + " icon=" + icon);
            mIcons.setIcon(index, icon);

            if (mBar != null) {
                try {
                    // 最后还得通过 IPC 调用到 UI（SystemUI）通知下 UI 更新数据
                    mBar.setIcon(index, icon);
                } catch (RemoteException ex) {
                }   
            }   
        }   
    }
```

最开始有前面说的权限检测：

```java
    private void enforceStatusBar() {
        mContext.enforceCallingOrSelfPermission(android.Manifest.permission.STATUS_BAR,
                "StatusBarManagerService");    
    }
```

检测下调用者是否有权限调用 SBMS，估计应该是要 system app 才行吧。然后有一个叫 StatusBarIconList mIcons 的数据结构：

```java
public class StatusBarIconList implements Parcelable {
    // 主要是封装了一个 StatusBarIcon 数组，然后对应一个 Slots 的 String 数组
    private String[] mSlots;
    private StatusBarIcon[] mIcons;
   
    public StatusBarIconList() {
    }

... ...

    public void defineSlots(String[] slots) {
        final int N = slots.length;    
        String[] s = mSlots = new String[N];
        for (int i=0; i<N; i++) {      
            s[i] = slots[i];
        }
        mIcons = new StatusBarIcon[N]; 
    }

    public int getSlotIndex(String slot) {
        final int N = mSlots.length;   
        for (int i=0; i<N; i++) {      
            if (slot.equals(mSlots[i])) {  
                return i;
            }
        }
        return -1;
    }

    public void setIcon(int index, StatusBarIcon icon) {
        mIcons[index] = icon.clone();  
    }

    public void removeIcon(int index) {
        mIcons[index] = null;
    }

... ...

}
```

名字也很形象，内部是用一个数组封装了下 StatusBarIcon，然后以 String 为标志（我觉得内部用 HashMap<String, StatusBarIcon> 会更好一些）。这里数据用得和 Notification 是一样的，都是 StatusBarIcon。然后关键点的方法也贴出来了，都比较简单一些贴这，后面提到的话翻回来看吧。前面说到系统状态图标是固定的，所以这里没有任何动态添加 StatusBarIcon 的接口，只有一个 defineSlots 的接口。之后，这个数组长度就固定了，哦，我收回我前面说用 HashMap 更好的话，长度固定的话，用数组最快。

然后这个数组是在 SBMS 构造函数那里初始化的：

```java
    /*
     * Construct the service, add the status bar view to the window manager
     */
    public StatusBarManagerService(Context context, WindowManagerService windowManager) {
        mContext = context;
        mWindowManager = windowManager;
        mWindowManager.setOnHardKeyboardStatusChangeListener(this);

        final Resources res = context.getResources();
        mIcons.defineSlots(res.getStringArray(com.android.internal.R.array.config_statusBarIcons));
    }
```

xml 中的数组在这：

```xml
    <!-- Do not translate. Defines the slots for the right-hand side icons.  That is to say, the
         icons in the status bar that are not notifications. -->
    <string-array name="config_statusBarIcons">
       <item><xliff:g id="id">ime</xliff:g></item>
       <item><xliff:g id="id">sync_failing</xliff:g></item>
       <item><xliff:g id="id">sync_active</xliff:g></item>
       <item><xliff:g id="id">gps</xliff:g></item>
       <item><xliff:g id="id">bluetooth</xliff:g></item>
       <item><xliff:g id="id">nfc</xliff:g></item>
       <item><xliff:g id="id">tty</xliff:g></item>
       <item><xliff:g id="id">speakerphone</xliff:g></item>
       <item><xliff:g id="id">mute</xliff:g></item>
       <item><xliff:g id="id">volume</xliff:g></item>
       <item><xliff:g id="id">wifi</xliff:g></item>
       <item><xliff:g id="id">cdma_eri</xliff:g></item>
       <item><xliff:g id="id">data_connection</xliff:g></item>
       <item><xliff:g id="id">phone_evdo_signal</xliff:g></item>
       <item><xliff:g id="id">phone_signal</xliff:g></item>
       <item><xliff:g id="id">battery</xliff:g></item>
       <item><xliff:g id="id">alarm_clock</xliff:g></item>
       <item><xliff:g id="id">secure</xliff:g></item>
       <item><xliff:g id="id">clock</xliff:g></item>
    </string-array>
```

4.2 的原生系统有上面 19 个系统状态图标。我们接下来把剩下2个接口也看完：

```java
    // setIconVisibility 其实是通过 setIcon 改变 StatusBarIcon 中的一个属性而已
    public void setIconVisibility(String slot, boolean visible) {
        enforceStatusBar();

        synchronized (mIcons) {        
            int index = mIcons.getSlotIndex(slot);
            if (index < 0) {
                throw new SecurityException("invalid status bar icon slot: " + slot);
            }

            StatusBarIcon icon = mIcons.getIcon(index);
            if (icon == null) {            
                return;
            }

            if (icon.visible != visible) { 
                icon.visible = visible;        

                if (mBar != null) {            
                    try {
                        mBar.setIcon(index, icon);     
                    } catch (RemoteException ex) { 
                    }
                }
            }
        }
    }

    // 这个是从 StatusBarIconList 找到对应的数据，然后置 NULL
    public void removeIcon(String slot) { 
        enforceStatusBar();

        synchronized (mIcons) {        
            int index = mIcons.getSlotIndex(slot);
            if (index < 0) {
                throw new SecurityException("invalid status bar icon slot: " + slot);
            }

            mIcons.removeIcon(index);      

            if (mBar != null) {            
                try {
                    mBar.removeIcon(index);        
                } catch (RemoteException ex) { 
                }
            }
        }
    } 
```

## SystemUI 的处理

上面 SBMS 虽然有3个接口，但是 UI 这边就只有2个：

```java
/* @hide */
oneway interface IStatusBar
{
    void setIcon(int index, in StatusBarIcon icon);
    void removeIcon(int index);

... ...

}
```

上面 SBMS 的实现也看到了，也只是用到 UI 这边2个接口。前面通知篇分析了 SystemUI 状态栏的一些设计和 UI 和 SS 那些关系的，这里就省略了（忘了的回去看一下）。桥接依旧在 CommandQueue：

```java
    public void setIcon(int index, StatusBarIcon icon) {
        synchronized (mList) {
            int what = MSG_ICON | index;
            mHandler.removeMessages(what);
            mHandler.obtainMessage(what, OP_SET_ICON, 0, icon.clone()).sendToTarget();
        }   
    }

    public void removeIcon(int index) {
        synchronized (mList) {
            int what = MSG_ICON | index;
            mHandler.removeMessages(what);
            mHandler.obtainMessage(what, OP_REMOVE_ICON, 0, null).sendToTarget();
        }   
    } 

... ...

    private final class H extends Handler {
        public void handleMessage(Message msg) {
            final int what = msg.what & MSG_MASK;
            switch (what) {
                case MSG_ICON: {
                    final int index = msg.what & INDEX_MASK;
                    final int viewIndex = mList.getViewIndex(index);
                    switch (msg.arg1) {
                        case OP_SET_ICON: {
                            StatusBarIcon icon = (StatusBarIcon)msg.obj;
                            StatusBarIcon old = mList.getIcon(index);
                            if (old == null) {
                                mList.setIcon(index, icon);
                                mCallbacks.addIcon(mList.getSlot(index), index, viewIndex, icon);
                            } else {
                                mList.setIcon(index, icon);
                                mCallbacks.updateIcon(mList.getSlot(index), index, viewIndex,
                                        old, icon);
                            }
                            break;
                        }
                        case OP_REMOVE_ICON:
                            if (mList.getIcon(index) != null) {
                                mList.removeIcon(index);
                                mCallbacks.removeIcon(mList.getSlot(index), index, viewIndex);
                            }
                            break;
                    }
                    break;
                }
... ...

            }           
        }                   
    }
```

经过桥接的 CommandQueue 最后交给 UI 实现的接口又变成3个了：

```java
    /*
     * These methods are called back on the main thread.
     */
    public interface Callbacks {
        public void addIcon(String slot, int index, int viewIndex, StatusBarIcon icon);
        public void updateIcon(String slot, int index, int viewIndex,
                StatusBarIcon old, StatusBarIcon icon);
        public void removeIcon(String slot, int index, int viewIndex);

... ...

    }
```

然后 UI 这边又有一个 StatusBarIconList 的 mList，最后这个 list 是在这的：

```java
// =================== BaseStatusBar.java ===================

    public void start() {
        mWindowManager = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        mWindowManagerService = WindowManagerGlobal.getWindowManagerService();
        mDisplay = mWindowManager.getDefaultDisplay();

        mProvisioningObserver.onChange(false); // set up
        mContext.getContentResolver().registerContentObserver(
                Settings.Global.getUriFor(Settings.Global.DEVICE_PROVISIONED), true,
                mProvisioningObserver);

        mBarService = IStatusBarService.Stub.asInterface(
                ServiceManager.getService(Context.STATUS_BAR_SERVICE));

        // 在 SystemUI 这里又 new 了一个 StatusBarIconList 出来
        // Connect in to the status bar manager service
        StatusBarIconList iconList = new StatusBarIconList();
        ArrayList<IBinder> notificationKeys = new ArrayList<IBinder>();
        ArrayList<StatusBarNotification> notifications = new ArrayList<StatusBarNotification>();
        mCommandQueue = new CommandQueue(this, iconList);

... ...

    }

// ================== CommandQueue.java =====================

    public CommandQueue(Callbacks callbacks, StatusBarIconList list) {
        mCallbacks = callbacks;
        mList = list;
    }
```

比起通知，系统状态数据只存了2份（SBMS，SystemUI），还算好的了（通知存了3份： NM，SBMS，SystemUI）。然后从上面的代码来看，3个接口主要是：

* **addIcon**: 如果调用的 setIcon 的 slot 没有在 SystemUI 中的数据有记录，那么会调用这个，让 SystemUI 添加一个新的状态图标（StatusBarIconView）
* **updateIcon**: 如果调用的 setIcon 的 slot 在 SystemUI 中有记录，那么会调用这个，让 SystemUI 更新下已有的状态图标状态
* **removeIcon**: 让 SystemUI 移除一个已经添加的状态图标

我们以 PhoneStatusBar 为例，来看下相关的实现：

```java
public class PhoneStatusBar extends BaseStatusBar {
... ...

    // mSystemIconArea 是包括 wifi、电量、时间和系统状态图标一起的容器（在右边那一大堆）
    // right-hand icons
    LinearLayout mSystemIconArea;
            
    // mStatusIcons 是单独系统状态图标，包含在上面的 mSystemIconArea 里面
    // left-hand icons
    LinearLayout mStatusIcons;
    // mNotificationIcons 是左边的通知的图标
    // the icons themselves
    IconMerger mNotificationIcons;
    // [+>
    View mMoreIcon; 


    // ================================================================================
    // Constructing the view
    // ================================================================================
    protected PhoneStatusBarView makeStatusBarView() {
... ...

        mSystemIconArea = (LinearLayout) mStatusBarView.findViewById(R.id.system_icon_area);
        mStatusIcons = (LinearLayout)mStatusBarView.findViewById(R.id.statusIcons);
        mNotificationIcons = (IconMerger)mStatusBarView.findViewById(R.id.notificationIcons);
        mNotificationIcons.setOverflowIndicator(mMoreIcon);
        mStatusBarContents = (LinearLayout)mStatusBarView.findViewById(R.id.status_bar_contents);
        mTickerView = mStatusBarView.findViewById(R.id.ticker);

... ...
        return mStatusBarView;
    }

... ....

    // 实现比较简单就是 new 一个 view（StatusBarIconView）出来，然后添加到 parent 里面
    public void addIcon(String slot, int index, int viewIndex, StatusBarIcon icon) {
        if (SPEW) Slog.d(TAG, "addIcon slot=" + slot + " index=" + index + " viewIndex=" + viewIndex
                + " icon=" + icon);
        StatusBarIconView view = new StatusBarIconView(mContext, slot, null);
        view.set(icon);
        mStatusIcons.addView(view, viewIndex, new LinearLayout.LayoutParams(mIconSize, mIconSize));
    }
        
    // 更新状态的还要去 StatusBarIconView 里面去看
    public void updateIcon(String slot, int index, int viewIndex,
            StatusBarIcon old, StatusBarIcon icon) {
        if (SPEW) Slog.d(TAG, "updateIcon slot=" + slot + " index=" + index + " viewIndex=" + viewIndex
                + " old=" + old + " icon=" + icon);
        StatusBarIconView view = (StatusBarIconView)mStatusIcons.getChildAt(viewIndex);
        view.set(icon);
    }
            
    // 移除的，就是 removeView 而已
    public void removeIcon(String slot, int index, int viewIndex) {
        if (SPEW) Slog.d(TAG, "removeIcon slot=" + slot + " index=" + index + " viewIndex=" + viewIndex);
        mStatusIcons.removeViewAt(viewIndex);
    }

... ...

}
```

实现都比较简单，然后去 StatusBarIconView 里面看看更新状态的：

```java
    /*
     * Returns whether the set succeeded.
     */
    public boolean set(StatusBarIcon icon) {
        final boolean iconEquals = mIcon != null
                && streq(mIcon.iconPackage, icon.iconPackage)
                && mIcon.iconId == icon.iconId;
        final boolean levelEquals = iconEquals
                && mIcon.iconLevel == icon.iconLevel;
        final boolean visibilityEquals = mIcon != null
                && mIcon.visible == icon.visible;
        final boolean numberEquals = mIcon != null
                && mIcon.number == icon.number;
        mIcon = icon.clone();
        setContentDescription(icon.contentDescription);
        // 图标变化
        if (!iconEquals) {
            Drawable drawable = getIcon(icon);
            if (drawable == null) {
                Slog.w(TAG, "No icon for slot " + mSlot);
                return false;
            }   
            setImageDrawable(drawable);
        }   
        // 图标索引变化
        if (!levelEquals) {
            setImageLevel(icon.iconLevel);
        }   

        // 这个 number 暂时没管是什么东西
        if (!numberEquals) {
            if (icon.number > 0 && mContext.getResources().getBoolean(
                        R.bool.config_statusBarShowNumber)) {
                if (mNumberBackground == null) {
                    mNumberBackground = getContext().getResources().getDrawable(
                            R.drawable.ic_notification_overlay);
                }   
                placeNumber();
            } else {
                mNumberBackground = null;
                mNumberText = null;
            }   
            invalidate();
        }   
        // 显示/隐藏状态变化
        if (!visibilityEquals) {
            setVisibility(icon.visible ? VISIBLE : GONE);
        }   
        return true;
    }
```

通过上面可以看到前面 SBMS 的 setIconVisibility 改变 StatusBarIcon 的 visible 属性，最后到 UI 这就是变成改变状态图标的 view 的 VISIBLE（GONE）属性。

然后前面说 4.2 系统的状态图标数组有 19 个，一般状态栏应该是放不下的。其实不用担心，因为虽然数据数组有 19 个，但是只是 NULL 的占位而已。要调用 SBMS 的 setIcon 才会正在填充数据进去，进而通知 UI 添加状态图标 view 到 WindowManager（WM）中（显示在界面上）。dumpsys 一下 SBMS 会发现 4.2 默认系统的状态图标有下面几个：

```bash
Icon list:
   0: (ime) StatusBarIcon(pkg=com.sohu.inputmethod.sogouuser=0 id=0x7f020076 level=0 visible=true num=0 )
   1: (sync_failing) StatusBarIcon(pkg=com.android.systemuiuser=0 id=0x7f020187 level=0 visible=false num=0 )
   2: (sync_active) StatusBarIcon(pkg=com.android.systemuiuser=0 id=0x7f020186 level=0 visible=false num=0 )
   3: (gps) null
   4: (bluetooth) StatusBarIcon(pkg=com.android.systemuiuser=0 id=0x7f020154 level=0 visible=false num=0 )
   5: (nfc) null
   6: (tty) StatusBarIcon(pkg=com.android.systemuiuser=0 id=0x7f020188 level=0 visible=false num=0 )
   7: (speakerphone) null
   8: (mute) null
   9: (volume) StatusBarIcon(pkg=com.android.systemuiuser=0 id=0x7f020171 level=0 visible=true num=0 )
  10: (wifi) null
  11: (cdma_eri) StatusBarIcon(pkg=com.android.systemuiuser=0 id=0x7f020173 level=0 visible=false num=0 )
  12: (data_connection) null
  13: (phone_evdo_signal) null
  14: (phone_signal) null
  15: (battery) null
  16: (alarm_clock) StatusBarIcon(pkg=com.android.systemuiuser=0 id=0x7f02013a level=0 visible=true num=0 )
  17: (secure) null
  18: (clock) null
Notification list:
   0: StatusBarNotification(pkg=com.android.systemui id=17303780 tag=null score=0 notn=Notification(pri=0 contentView=com.android.systemui/0x1090099 vibrate=null sound=file:///system/media/audio/notificati
  mDisabled=0x3e10000
  mDisableRecords.size=2
    [0] userId=0 what=0x3010000 pkg=android token=android.os.Binder@4162e520
    [1] userId=0 what=0xe00000 pkg=WindowManager.LayoutParams token=android.os.Binder@419f0da0
```

19 个列表中有很多是空的（null）的，要让系统显示那些状态，需要调用 SBMS 的 setIcon 设置才会填充图标数据。4.2 中主要在 SystemUI 的 PhoneStatusBarPolicy 中设置的（以 Phone UI 为例子）：

```java
    public PhoneStatusBarPolicy(Context context) {
        mContext = context;
        mService = (StatusBarManager)context.getSystemService(Context.STATUS_BAR_SERVICE);

        // listen for broadcasts
        IntentFilter filter = new IntentFilter();
        filter.addAction(Intent.ACTION_ALARM_CHANGED);
        filter.addAction(Intent.ACTION_SYNC_STATE_CHANGED);
        filter.addAction(AudioManager.RINGER_MODE_CHANGED_ACTION);
        filter.addAction(BluetoothAdapter.ACTION_STATE_CHANGED);
        filter.addAction(BluetoothAdapter.ACTION_CONNECTION_STATE_CHANGED);
        filter.addAction(TelephonyIntents.ACTION_SIM_STATE_CHANGED);
        filter.addAction(TtyIntent.TTY_ENABLED_CHANGE_ACTION);
        mContext.registerReceiver(mIntentReceiver, filter, null, mHandler);

        // storage
        mStorageManager = (StorageManager) context.getSystemService(Context.STORAGE_SERVICE);
        mStorageManager.registerListener(
                new com.android.systemui.usb.StorageNotification(context));

        // TTY status
        mService.setIcon("tty",  R.drawable.stat_sys_tty_mode, 0, null);
        mService.setIconVisibility("tty", false);

        // Cdma Roaming Indicator, ERI
        mService.setIcon("cdma_eri", R.drawable.stat_sys_roaming_cdma_0, 0, null);
        mService.setIconVisibility("cdma_eri", false);

        // bluetooth status
        BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter();
        int bluetoothIcon = R.drawable.stat_sys_data_bluetooth;
        if (adapter != null) {
            mBluetoothEnabled = (adapter.getState() == BluetoothAdapter.STATE_ON);
            if (adapter.getConnectionState() == BluetoothAdapter.STATE_CONNECTED) {
                bluetoothIcon = R.drawable.stat_sys_data_bluetooth_connected;
            }   
        }   
        mService.setIcon("bluetooth", bluetoothIcon, 0, null);
        mService.setIconVisibility("bluetooth", mBluetoothEnabled);

        // Alarm clock
        mService.setIcon("alarm_clock", R.drawable.stat_sys_alarm, 0, null);
        mService.setIconVisibility("alarm_clock", false);

        // Sync state
        mService.setIcon("sync_active", R.drawable.stat_sys_sync, 0, null);
        mService.setIcon("sync_failing", R.drawable.stat_sys_sync_error, 0, null);
        mService.setIconVisibility("sync_active", false);
        mService.setIconVisibility("sync_failing", false);

        // volume
        mService.setIcon("volume", R.drawable.stat_sys_ringer_silent, 0, null);
        mService.setIconVisibility("volume", false);
        updateVolume();
    }
```

然后稍微贴一下 PhoneStatusBarPolicy 中更新这些状态的代码：

```java
    private BroadcastReceiver mIntentReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            // 前面的说的 SystemUI 注册闹钟状态设置的广播，间接对外提供接口
            if (action.equals(Intent.ACTION_ALARM_CHANGED)) {
                updateAlarm(intent);           
            }
            else if (action.equals(Intent.ACTION_SYNC_STATE_CHANGED)) {
                updateSyncState(intent);       
            }
            else if (action.equals(BluetoothAdapter.ACTION_STATE_CHANGED) ||
                    action.equals(BluetoothAdapter.ACTION_CONNECTION_STATE_CHANGED)) {
                updateBluetooth(intent);       
            }
            else if (action.equals(AudioManager.RINGER_MODE_CHANGED_ACTION)) {
                updateVolume();                
            }
            else if (action.equals(TelephonyIntents.ACTION_SIM_STATE_CHANGED)) {
                updateSimState(intent);        
            }
            else if (action.equals(TtyIntent.TTY_ENABLED_CHANGE_ACTION)) {
                updateTTY(intent);             
            }
        }
    };

... ...

    // 贴一个就行了，其他的差不多的
    private final void updateAlarm(Intent intent) {
        boolean alarmSet = intent.getBooleanExtra("alarmSet", false);
        mService.setIconVisibility("alarm_clock", alarmSet);
    }
```

还有一个输入法的在这里（InputMethodManagerService.java）：

```java
    @Override
    public void updateStatusIcon(IBinder token, String packageName, int iconId) {
        int uid = Binder.getCallingUid();
        long ident = Binder.clearCallingIdentity();
        try {
            if (token == null || mCurToken != token) {
                Slog.w(TAG, "Ignoring setInputMethod of uid " + uid + " token: " + token);
                return;
            }
            
            synchronized (mMethodMap) {
                if (iconId == 0) {
                    if (DEBUG) Slog.d(TAG, "hide the small icon for the input method");
                    if (mStatusBar != null) {
                        mStatusBar.setIconVisibility("ime", false);
                    }   
                } else if (packageName != null) {
                    if (DEBUG) Slog.d(TAG, "show a small icon for the input method");
                    CharSequence contentDescription = null;
                    try {
                        // Use PackageManager to load label
                        final PackageManager packageManager = mContext.getPackageManager();
                        contentDescription = packageManager.getApplicationLabel(
                                mIPackageManager.getApplicationInfo(packageName, 0,
                                        mSettings.getCurrentUserId()));
                    } catch (RemoteException e) {
                        /* ignore */ 
                    }   
                    if (mStatusBar != null) {
                        mStatusBar.setIcon("ime", packageName, iconId, 0,
                                contentDescription  != null
                                        ? contentDescription.toString() : null);
                        mStatusBar.setIconVisibility("ime", true);
                    }
                }   
            }               
        } finally {
            Binder.restoreCallingIdentity(ident);
        }       
    }
```

## 总结

这篇内容比较简单。系统状态图标数据和通知的状态栏图标是一样的，都是 StautsBarIcon，视图也是一样的（StatusBarIconView），因为它们 UI 表现是一样的。只不过一个是显示应用的信息，一个是显示系统的状态信息；一个可以动态改变数量，一个固定数量而已。不过还是这2种不同系统 UI 元素还是要区分一些，不然有些时候搞混了，就找不到在哪定制这些东西了。


