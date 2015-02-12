title: 编译 Android 按键映射分析
date: 2015-01-27 23:32:16
updated: 2015-01-27 23:32:16
categories: [Android Framework]
tags: [android]
---

android 能够将不同的低层 scancode 转化成上层使用的统一的 keycode （以下分析为 android 2.2 froyo 的）。下面说的几个相关的源代码文件都在 framework/base/libs/ui 下。

## EventHub.cpp
先看看下面这段代码：

```cpp
// 在 open_device 函数里
if ((device->classes&CLASS_KEYBOARD) != 0) { 
   char devname[PROPERTY_VALUE_MAX];
   char keylayoutFilename[300];

   const char* root = getenv("ANDROID_ROOT");
   property_get("persist.sys.keylayout", devname, "qwerty");
   snprintf(keylayoutFilename, sizeof(keylayoutFilename), "%s/usr/keylayout/%s.kl", root, devname);
   bool defaultKeymap = access(keylayoutFilename, R_OK);
   if (defaultKeymap) {
      strcpy(devname, "qwerty");
      snprintf(keylayoutFilename, sizeof(keylayoutFilename),
                     "%s/usr/keylayout/%s.kl", root, devname);
   }
   LOGI("keylayout = %s, Filename = %s", devname, keylayoutFilename);
   device->layoutMap->load(keylayoutFilename);
```

这段代码是打开键盘设备，并读取按键映射表的。从代码里可以看得到映射文件是从 root/usr/keylayout/qwerty.kl （root 一般是 system）下读取的（这个 qwerty.kl 各个设备厂商应该可以自己改，我看的 x86 的默认使用的是这个）。

## kl 文件
这个文件就是 android 的按键映射文件，结构如下：

* **BACK**:
    * BEGIN: key
    * SCANCODE: 1 
    * KEYCODE: BACK
    * FLAG: WACK_DROPPED

* **POWER**:
    * BEGIN: key
    * SCANCODE: 116 
    * KEYCODE: POWER
    * FLAG: WACK

这里配合看下下面的代码，在 KeyLayoutMap.cpp 里：

```cpp
status_t
KeyLayoutMap::load(const char* filename)
{
    int fd = open(filename, O_RDONLY);
    if (fd < 0) {
        LOGE("error opening file=%s err=%s\n", filename, strerror(errno));
        m_status = errno;
        return errno;
    }

    off_t len = lseek(fd, 0, SEEK_END);
    off_t errlen = lseek(fd, 0, SEEK_SET);
    if (len < 0 || errlen < 0) {
        close(fd);
        LOGE("error seeking file=%s err=%s\n", filename, strerror(errno));
        m_status = errno;
        return errno;
    }

    char* buf = (char*)malloc(len+1);
    if (read(fd, buf, len) != len) {
        LOGE("error reading file=%s err=%s\n", filename, strerror(errno));
        m_status = errno != 0 ? errno : ((int)NOT_ENOUGH_DATA);
        return errno != 0 ? errno : ((int)NOT_ENOUGH_DATA);
    }
    errno = 0;
    buf[len] = '\0';

    int32_t scancode = -1;
    int32_t keycode = -1;
    uint32_t flags = 0;
    uint32_t tmp;
    char* end;
    status_t err = NO_ERROR;
    int line = 1;
    char const* p = buf;
    enum { BEGIN, SCANCODE, KEYCODE, FLAG } state = BEGIN;
    while (true) {
        String8 token = next_token(&p, &line);
        if (*p == '\0') {
            break;
        }
        switch (state)
        {
            case BEGIN:
                if (token == "key") {
                    state = SCANCODE;
                } else {
                    LOGE("%s:%d: expected key, got '%s'\n", filename, line,
                            token.string());
                    err = BAD_VALUE;
                    goto done;
                }
                break;
            case SCANCODE:
                scancode = strtol(token.string(), &end, 0);
                if (*end != '\0') {
                    LOGE("%s:%d: expected scancode (a number), got '%s'\n",
                            filename, line, token.string());
                    goto done;
                }
                //LOGI("%s:%d: got scancode %d\n", filename, line, scancode );
                state = KEYCODE;
                break;
            case KEYCODE:
                keycode = token_to_value(token.string(), KEYCODES);
                //LOGI("%s:%d: got keycode %d for %s\n", filename, line, keycode, token.string() );
                if (keycode == 0) {
                    LOGE("%s:%d: expected keycode, got '%s'\n",
                            filename, line, token.string());
                    goto done;
                }
                state = FLAG;
                break;
            case FLAG:
                if (token == "key") {
                    if (scancode != -1) {
                        //LOGI("got key decl scancode=%d keycode=%d"
                        //       " flags=0x%08x\n", scancode, keycode, flags);
                        Key k = { keycode, flags };
                        m_keys.add(scancode, k);
                        state = SCANCODE;
                        scancode = -1;
                        keycode = -1;
                        flags = 0;
                        break;
                    }
                }
                tmp = token_to_value(token.string(), FLAGS);
                //LOGI("%s:%d: got flags %x for %s\n", filename, line, tmp, token.string() );
                if (tmp == 0) {
                    LOGE("%s:%d: expected flag, got '%s'\n",
                            filename, line, token.string());
                    goto done;
                }
                flags |= tmp;
                break;
        }
    }
    if (state == FLAG && scancode != -1 ) {
        //LOGI("got key decl scancode=%d keycode=%d"
        //       " flags=0x%08x\n", scancode, keycode, flags);
        Key k = { keycode, flags };
        m_keys.add(scancode, k);
    }

done:
    free(buf);
    close(fd);

    m_status = err;
    return err;
}
```

可以看得出，android 会解析 kl 文件里的项目，然后把解析后的结果保存到一个 Vector 结构里（android 自己写一个 vector，不是 c++ std 的）。 kl 文件的每一项分为4段：

* **BEGIN**
这一段统一是 "key"，应该是按键映射的标识。

* **SCANCODE**
这一段直接将字符串转化为数字，也就是说 kl 文件里的这一段存储的是数字。这里值就是从低层硬件读取出来值。也就是我们在 OM  项目中， vfb 应该发送给 android 的值。这个值不是 android sdk 中描述的值（这个值是上层应用使用的）。不同的硬件会不一样。

* **KEYCODE**
这个值就是 android sdk 里描述的啦。不过这一段有很多都是字符串，不是直接保存数值的，这个有个转化关系，具体的后面再说。

* **FLAG**
这里目前来说就2个值 WAKE 和 WAKE_DROPPED 。好像 WAKE 代表可以在锁机状态下唤醒屏幕。

## KeycodeLabels.h
上面说了 KEYCODE 那里很多是保存字符串的，那么转化的地方就在这个文件里（这个文件在 framework/base/include/ui 下）：

```cpp
struct KeycodeLabel {
    const char *literal;
    int value;
};

// 这里不贴全了，都在这
static const KeycodeLabel KEYCODES[] = { 
    { "SOFT_LEFT", 1 },
    { "SOFT_RIGHT", 2 },
    { "HOME", 3 },
    { "BACK", 4 },
    { "CALL", 5 },
    { "ENDCALL", 6 },
    { "0", 7 },
    { "1", 8 },
    { "2", 9 },
    { "3", 10 },
    { "4", 11 },
    { "5", 12 },
    { "6", 13 },
    { "7", 14 },
    { "8", 15 },
    { "9", 16 },
    { "STAR", 17 },
    { "POUND", 18 },
    { "DPAD_UP", 19 },
    ... ...
```

前面说的那个 load 代码里 KEYCODE 那一段的 keycode = token_to_value(token.string(), KEYCODES); 中的 token_to_value 就是通过上面这个数组来得到对应的数值的。这个数值就是 android sdk 里写的给上层应用程序使用的按键值了。所以一般来说，我们修改 kl 文件就可以改变按键映射了。

