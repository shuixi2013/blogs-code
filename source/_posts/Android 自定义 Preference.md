title: Android 自定义 Preference
date: 2015-01-25 23:32:16
tags: [android]
---

有些时候系统提供的 Preference 不满足我们的要求的时候，我们就需要自己定制了。现在产品要求 ChekBoxPreference 的 summary 的颜色要能动态改变，在关闭的时候是默认颜色，在开启的时候变成红色。现在我们就可以自己定制啦。

## 简单的修改 xml

先说说简单的情况。如果字体颜色只是静态的话，可以不用改代码，改改 layout xml 就好了。系统自己的 xml 是 framework/base/core/res/res/layout/preference.xml 。把这个文件复制，然后在其基础上改改 text 的属性就好了，然后在使用 CheckBoxPreference 的时候指定自己定制的 xml（这里我叫 custom_preference.xml）。

```html
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

```html
// 改好后，在用 CheckBoxPreference 的时候，指定自己的 xml layout
<CheckBoxPreference android:key="lock_screen" 
    	android:title="@string/settings_lock_screen_title" 
    	android:defaultValue="false" 
        android:layout="@layout/custom_preference"
    	android:summaryOff="@string/settings_lock_screen_off_summary" 
    	android:summaryOn="@string/settings_lock_screen_on_summary" 
    	android:summary="@string/settings_lock_screen_off_summary">
    </CustomCheckBoxPreference>
```

## 扩展代码

如果需要复杂点的功能，就需要继承 CheckBoxPreference ，然后自己定制代码了。这里以动态的改字体颜色为例：继承 CheckBoxPreference 后，重载父类的 onBindView 函数（这个函数好像是在将数据显示到视图上的时候调用的）：

```java
public class CustomCheckBoxPreference extends CheckBoxPreference {

	public CustomCheckBoxPreference(Context context, AttributeSet attrs,
			int defStyle) {
		super(context, attrs, defStyle);
		// TODO Auto-generated constructor stub
	}
	
	@Override
	protected void onBindView(View view) {
		super.onBindView(view);
		
		boolean isChecked = isChecked();
		Resources res = view.getResources();
		
		TextView summaryView = (TextView)view.findViewById(com.android.internal.R.id.summary);
		if (summaryView != null) {
			if (isChecked) {
				summaryView.setTextColor(res.getColor(R.color.red));
			} else {
				//summaryView.setTextAppearance(getContext(), com.android.internal.R.attr.textAppearanceSmall);
				summaryView.setTextColor(res.getColor(com.android.internal.R.color.secondary_text_light));
			}
		}
	}
}
```

写好自己的类之后，在用的时候把原来的 CheckBoxPreference 换成自己的类就可以了：

```html
<cn.fmsoft.launcher2.CustomCheckBoxPreference android:key="lock_screen" 
    	android:title="@string/settings_lock_screen_title" 
    	android:defaultValue="false" 
    	android:summaryOff="@string/settings_lock_screen_off_summary" 
    	android:summaryOn="@string/settings_lock_screen_on_summary" 
    	android:summary="@string/settings_lock_screen_off_summary">
    </cn.fmsoft.launcher2.CustomCheckBoxPreference>
```

## 参考
参考出处：[Preference 使用小结](http://www.cnblogs.com/franksunny/archive/2011/10/21/2219890.html "Preference 使用小结")


