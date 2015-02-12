title: Android Provision 的影响和作用
date: 2015-01-28 20:11:16
updated: 2015-01-28 20:11:16
categories: [Android Framework]
tags: [android]
---

Provision 是 android 中的一个系统应用（源码位置在： packages/apps/Provision 下面）。它的主要作用是作为开机引导用户进行一些基本设置。但是在原生的 android 系统中，这个 provision 非常的简单，只有一个空白的 activity，这个主要就是留给 OEM 厂商自己定制的（像 google 的 nexus 进行让里你登陆 google 帐号）。

网上说这个东西启动比桌面还早，看下 AndroidManifest.xml:

```xml
    <application>
        <activity android:name="DefaultActivity"
                android:excludeFromRecents="true">
            <intent-filter android:priority="1">
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.HOME" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
    </application>
```

其实这个就是一个桌面而已，为什么比普通桌面启动快，只不过它把优先级设置得比普通桌面高了一点而已： android:priority="1" ，普通桌面不设置的话，默认是 0。package manager 在解析 Intent 的时候会优先返回优先级高的。

然后这个 apk 的就只有一个 activity：

```java
public class DefaultActivity extends Activity {

    @Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);

        // Add a persistent setting to allow other apps to know the device has been provisioned.
        Settings.Global.putInt(getContentResolver(), Settings.Global.DEVICE_PROVISIONED, 1); 
        Settings.Secure.putInt(getContentResolver(), Settings.Secure.USER_SETUP_COMPLETE, 1); 

        // remove this activity from the package manager.
        PackageManager pm = getPackageManager();
        ComponentName name = new ComponentName(this, DefaultActivity.class);
        pm.setComponentEnabledSetting(name, PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
                PackageManager.DONT_KILL_APP);

        // terminate the activity.
        finish();
    }
}
```

太简单了。但是要注意2个地方，一个是在 Settings provdier 的 global 表中写入了一条记录： "device_provisioned = 1"（这点非常重要），第二个是调用 package manager 的接口，把自己给 disable 掉了。

先说第一点： 这个条记录非常重要，代表设备已经准备就绪，可以正常使用，换句话说，如果没有这条记录的话，那设备是无法正常使用的。事实上还真是这样，因为系统中那一票 services 都回检测这个值，如果没有的话，都回相应的罢工，例如说无法锁屏、按 Home 键无反应、无法发通知等等。我之前在调一个东西的时候，需要清除 settings provider 中的某个数值，当时为了图简单，我就直接把 settings.db 删掉了。重启后系统各种问题，当时查了好久，坑死爹了。

然后后面第二点就解释为什么手动把 settings.db 删掉了，就各种不能用了，因为 provision 调用 pm 的 disable 把自己 disable 掉了。这就导致下次在启动 HOME 应用的时候，pm 回忽略 disabled 状态的应用。所以如果你手动把 settings.db 删掉了，下次重启就悲剧了。

知道原因了就很好办了，如果调试真的要删除 settings.db ，那自己手动插入 "device_provisioned = 1" 就可以了。

这里可以看出 google 挺黑的，你不好好运行 provision 登陆我的 google 帐号，哥就不让你好好用 android 设备。


