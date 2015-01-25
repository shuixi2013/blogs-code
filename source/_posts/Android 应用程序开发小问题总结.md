title: Android 应用程序开发小问题总结
date: 2015-01-25 22:30:16
tags: [android]
---

新手问题多多 -_-|| 

## 权限问题

使用某些 api 进行操作时需要申请特定的权限的（最典型的就是写sdcard）。这类 api 一般来 sdk 文档中会有说明的，看的时候看仔细点，并且养成 catch 异常，并且把异常输出到 log 的好习惯。这样如果是因为权限问题失败的话，可以马上从 logcat 中看到类似 "Permission Denied" 输出。申请特定权限，编辑 AndroidManifest.xml 文件。点击 Premission 标签页，add Use Premission ，然后在下拉列表中选择对应的权限就可以了。当然你可以手动在 xml 中添加，但是使用 eclipse 带的编辑器比较方便。添加了权限后，会在安装 apk 的时候有提示。

## 切换 activity 的问题

一般来说一个应该是由多个 activity 组成的。这个时候就需要涉及到 activity 之间的切换了。一般来说启动别的 activity 可以使用如下代码：

```cpp
// 一般启动使用 Intent，Intent 的第一参数一般是填发起启动 activity 的 activity。
// 第二个参数就是你要启动的 activity 的 class。
// 后面那个可以设置一些启动参数，具体的可以去看 sdk docs
Intent editIntent = new Intent(this, EditGestureActivity.class);
editIntent.setFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);

// 这个可以用于 activity 之间交换数据
gestureName = (String)(item.get("ivText"));
if (gestureName != null)
	editIntent.putExtra("gestureName", gestureName);

// 然后通过 startActivity 启动 activity
try {
	startActivity(editIntent);
} catch (ActivityNotFoundException e) {
	Log.d("Mylog", "Error: " + e.toString());
}
```

不过这个需要注意2点：

* 如果你是启动你应用中的一个非主应用，那么你需要手动在 AndroidManifest.xml 文件的 `<application>` 标签上添加你的 activity 的标识，否则 startActivity 是无法启动你指定的 activity 的：

代码：

```html
<application android:icon="@drawable/icon" android:label="@string/app_name">
   ... ...
        
   // 你要启动的 activity 类
   <activity android:name=".EditGestureActivity"
       android:label="@string/app_name">
   </activity>

</application>
```

* 如果你是启动别的应用程序的主应用，就是你应用程序启动时候的第一个 activity，在 xml 文件的 `<activity>` 标签中有这样的标识的：

代码：

```html
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
```

那在上面的 Intent 设置中还需要加上 editIntent.addCategory(android.content.Intent.CATEGORY_LAUNCHER); 这样的代码才行。这样就可以启动别的应用程序了。

