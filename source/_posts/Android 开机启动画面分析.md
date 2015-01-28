title: Android 开机启动画面分析
date: 2015-01-28 19:46:16
categories: [Android Framework]
tags: [android]
---

android 开机启动画面一共有3种：

## framebuffer logo
在 linux kernel framebuffer 驱动中的 logo（代码在 kernel/drivers/video/fbmem.c）。是一个 ppm 文件。可以通过工具制作（下面这些工具在 ubuntu 上自己安装相应的工具就可以）：

1. pngtopnm logo.png > logo_linux.pnm
2. pnmquant 224 logo_linux.pnm > logo_linux_clut224.pnm
3. pnmtoplainpnm logo_linux_clut224.pnm > logo_linux_clut224.ppm
4. 替换 kernel/drivers/video/logo 下的文件即可。

这是最早的静态启动画面。一般产品发布的时候会把这个 logo 去掉的，一般是一个小企鹅，比较丑的说。这里不做过多分析。

## init 启动画面
开机的 init (代码在 system/core/init 下面)进程的启动 logo，是一张静态图片（init 的分析可以参考《深入理解 Android 卷I》的第3章）。在其中一个 action 中会调用 load 去加载 565 rle 格式的一张图片，然后写入 framebuffer 中显示出来：

init.c:

```cpp
static int console_init_action(int nargs, char **args)
{
    int fd;

    if (console[0]) {
        snprintf(console_name, sizeof(console_name), "/dev/%s", console);
    }

    fd = open(console_name, O_RDWR);
    if (fd >= 0)
        have_console = 1;     
    close(fd);
    
    // 加载 565 rle 图片并且显示
    // INIT_IMAGE_FILE 定义是 /initlogo.rle，最终打包在 ramdisk.img 里面的
    // 一般 OEM 定制可以把自己制作好的 rle 放到 devices 下面
    if( load_565rle_image(INIT_IMAGE_FILE) ) {
        // 如果没有图片或是加载失败，者直接往 fb 中写入 ANDROID 这几个字符
        fd = open("/dev/tty0", O_WRONLY); 
        if (fd >= 0) {
            const char *msg;  
                msg = "\n"    
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"              
            "\n"  // console is 40 cols x 30 lines
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"
            "             A N D R O I D "; 
            write(fd, msg, strlen(msg));   
            close(fd);
        }
    }
    return 0;
}
```

logo.c:

```cpp
int load_565rle_image(char *fn)
{
    struct FB fb;
    struct stat s;
    unsigned short *data, *bits, *ptr;
    unsigned count, max;
    int fd;

    if (vt_set_mode(1)) 
        return -1; 

    // 打开 rle 文件
    fd = open(fn, O_RDONLY);
    if (fd < 0) {
        ERROR("cannot open '%s'\n", fn);
        goto fail_restore_text;
    }

    if (fstat(fd, &s) < 0) {
        goto fail_close_file;
    }

    // 将 rle 文件读入内存
    data = mmap(0, s.st_size, PROT_READ, MAP_SHARED, fd, 0); 
    if (data == MAP_FAILED)
        goto fail_close_file;

    // 打开 fb，并 mmap 映射到程序内存（这个函数的实现在 logo.c 里面）
    if (fb_open(&fb))
        goto fail_unmap_data;

    max = fb_width(&fb) * fb_height(&fb);
    ptr = data;
    count = s.st_size;
    bits = fb.bits;
    // 把 rle 数据写入 fb，这个写法要参看 rle 的格式
    while (count > 3) {
        unsigned n = ptr[0];
        if (n > max)
            break;
        android_memset16(bits, ptr[1], n << 1); 
        bits += n;
        max -= n;
        ptr += 2;
        count -= 4;
    }

    munmap(data, s.st_size);
    fb_update(&fb);
    fb_close(&fb);
    close(fd);
    unlink(fn);
    return 0;

fail_unmap_data:
    munmap(data, s.st_size);
fail_close_file:
    close(fd);
fail_restore_text:
    vt_set_mode(0);
    return -1;
}
```

`load_565rle_image` 函数的输入参数是一个 rle 文件，该函数会把这个文件内容转换为rgb565，写入 framebuffer。以下百度百科来的 rle 简介 [rle简介](http://baike.baidu.com/view/18819.htm?fromTaglist "rle简介") ：

RLE全称（run-length encoding），翻译为游程编码，又译行程长度编码，又称变动长度编码法（run coding），在控制论中对于二值图像而言是一种编码方法，对连续的黑、白像素数(游程)以不同的码字进行编码。游程编码是一种简单的非破坏性资料压缩法，其好处是加压缩和解压缩都非常快。其方法是计算连续出现的资料长度压缩之。

RLE 文件格式为 [count(2 bytes), color(2 bytes)], count最大为 65535； color 为 RGB565, 当然也可以是 YUV422 ，我觉得只要是 2bytes 以下的都可以，但是由于最终我们写入到 framebuffer 中的都是 RGB565，从效率的角度，rle 文件的 color 值为RGB565最好。

`load_565rel_image` 源码第141行：写入n个像素点，像素点的 color 值是 rle 中的原始数据，这对于 rgb565 的 framebuffer 没有问题，但是如果 framebuffer 是RGB32（ARGB），就无法工作了。解决办法是做个转换，把 2bytes 的 RGB565 转换为 4bytes 的 ARGB 即可:

```cpp
#define rgb32_r(rle) (((rle & 0xf800) >> 11) << 3)
#define rgb32_g(rle) (((rle & 0x07e0) >> 5 << 2))
#define rgb32_b(rle) (((rle & 0x001f) << 3))
#define rgb32(rle) (rgb32_r(rle) << 16 | rgb32_g(rle) << 8 | rgb32_b(rle) << 0) 


        unsigned int *bits;
        bits = (unsigned int *)fb.bits;
        while (count > 3) {
            unsigned n = ptr[0];
            if (n > max)
                break;
            out = rgb32(ptr[1]);
            android_memset32(bits, out, n << 2);
            bits += n;
            max -= n;
            ptr += 2;
            count -= 4;
```

现在再回去，看看怎么得到 rle 格式的文件，一般来说分两步走，先把原始图片转为RGB565，然后再把 RGB565 转为 rle（当然不是必需的，如果有个能把 png 直接转为 rle 的工具）。

1. 先用 GMIP 生成一个 800X480 的 png 文件，为什么是 800x480，图片大小必须和屏大小一致，至于是不是 png 无所谓，只要有工具能转为 RGB565 即可
2. convert 命令 png 为 RGB565: `convert -depth 8 android_logo.png rgb:android_logo.raw`
3. 转换 RGB565 为 rle 文件（这个工具是源码编译出来的，在 out/host/linux-x86/bin 下面）：`rgb2565 -rle < android_logo.raw > initlogo.rle` 
4. 替换 ramdisk 中的 initlogo.rle，重新启动即可看到新的启动画面（这个是标准的，不同的平台这个东西在的地方可能不一样）。


注意 `load_565rle_image` 152行 unlink(fn) 会删除 initlogo.rle 文件，进入系统后，会发现 /initlogo.rle 文件已经被删除了，这样可以节约一点 ramdisk 文件系统的内存空间，对于 ramdisk 做 rootfs，这没有问题，因为 ramdisk 镜像本身并没有被修改。但是如果根文件系统是其他文件系统，就需要把 unlink(fn) 注释掉，以防止 initlog.rle 被永久删除。

其实 rle 在一段是我抄别人的： [参考](http://blog.csdn.net/kickxxx/article/details/7548670 "参考")

## bootanimtion
bootanimtion 是 system/bin 下的一个本地程序（代码在 framework/base/cmds/bootanimation 下面），在 inin.rc 中会有以下的定义：

<pre config="brush:bash;toolbar:false;">
service bootanim /system/bin/bootanimation
    user graphics
    group graphics
    disabled
    oneshot
</pre>

会在 surfaceflinger 刚启动的时候启动（surfaceflinger 中的 startBootAnim 通过 property_set("ctl.start", "bootanim") 通过 property 服务启动 bootanimation， 在 init.rc 中定义的 service）。 这个程序会一直循环的贴图（这里的动画就是通过不停的贴图来实现的），然后每个循环检测一个 property 值，如果这个值被设置了，就会退出循环。这个值由 surfaceflinger 写入， surfaceflinger 初始化好之后， framework 显示可以正工作了，就让 bootanimation 退出了（bootFinished 中 写入这个值）。当然上层在这里有超时设置的，如果在一段时间这个 property 值没有被设置的话， bootanimation 也会被强制终止掉。

```cpp
void BootAnimation::checkExit() {
    // Allow surface flinger to gracefully request shutdown
    if(mShutdown)//shutdown animation
        {   
        return;
        }   

    char value[PROPERTY_VALUE_MAX];
    property_get(EXIT_PROP_NAME, value, "0");
    int exitnow = atoi(value);
    if (exitnow) {
        requestExit();
    }
}
```

上面配置可以看得到 bootanimation 被配置到了 graphics 组，这个组可以绘制图像，但是没有写 property 的权限（可以自己去看提供 property 的服务代码），不过有读的权限。所以 bootanimation 是不能靠写入 property 来和上层的 services 进行通信的（我之前曾经想这么做 -_-||）。

bootanimation 是通过 GL 来进行绘制的（简单、粗暴、有效哈），通过 OES 扩展能比较轻松的进行 2D 贴图。 bootaniamtion 会去找 /system/media/bootanimation.zip 这个动画包，如果没有的话，就会显示 android 一排闪亮的字（用 native 的 AssetManager 去加载 framework/base/core/res/assets/image 下面的 android-logo-mask.png 和 android-logo-shine.png 这2张图片的），如果有的会则会解析这个 zip 包。然后读取里面 desc.txt 这个配置文件，设置一些动画属性，然后循环绑定纹理开始贴图。

动画分为2个部分：配置文件（固定为  desc.txt）、资源图片（仅支持 png 格式）。 desc.txt 和 ini 文件类似，一行、一行解析的，分为2种格式：

xx xx xx： 3个参数的，前2个是图像的宽、高，后面一个动画的 fps。例如：768 324 10，表示图像是 768x324，动画以 10fps 播放。这种一般就写一行就行了，写多了后面的会覆盖前面的配置的。如果这个分辨率比屏幕分辨率低的话，就会居中显示。这个分辨率最好和图片一样，否则会缩放的（GL 的纹理贴图，默认代码 GL 纹理选项没开最好的过滤方式，所以最好不要缩放）。

xx xx xx xx： 4个参数的，是用来描述动画组成部分的（part），android 的开机动画 part 分为2种类型，一种是循环有限次数播放的，播放完指定次数这个 part 就结束了，进入到下一个 part；一种是无限循环播放的，直到开机初始化完成，bootanimation 进程结束（一般就2个 part，第一个不循环的，第二个循环的，应该可以写多与3个的 part，但是一般都不这么做）。第一个参数表示 part 是否必需要等到播放完成才能结束（就是说这个 part 能不能被中断，例如说开机初始化很快，动画还没播放就初始化好了，这个时候 bootanimation 进程会接收到上层 framework 请求终止的消息）， 填 ‘c’ 表示必需要等到这个 part 不能被终止（就是要必需被播放完，不能被提前终止）。填其它的表示可以被提前终止（一般填写 ‘p’）。第二参数表示这个 part 的循环次数。如果填 0 就表示是无限循环的，大于 0 就是循环次数。注意一下，不要第一个参数填 ‘c’，这个参数填 0，这样填写开机动画就真*无限循环了。第三个参数表示循环之间的等待时间（单位是以 fps 帧数来算的，例如10就是表示等待10帧），就是播放一次循环后，等多长时间开始下一次循环。第四个参数表示这个 part 使用的图片资源的路径。在 zip 包中不同的 part 要建立不同的文件夹（例如 part1/, part2/），图片以 frame 动画的编号命名放好，例如 f0000、f0001、f0002。例如：

<pre config="brush:bash;toolbar:false;">
p 1 0 part1
p 0 10 part2
</pre>

表示开机动画有 2个 part， 2个part 都可以被提前终止，第一个 part 循环一次（只播放一次），由于只播放一次所以循环等待时间填0，图片路径为 part1；第二个 part 无限循环，每个 part 之间等待时间为 10，图片路径为 part2 。

这个命名上面要注意一下，代码里面使用了一个以名字来排序的 Vector 来存储动画帧图片。所以名字可以是 `xx_0001`, `xx_0002`， 也可以是 x0001, x0002，只要是名字的字符能够正确的排序就行了，不要作死的搞一些奇怪的名字就行。

```cpp
    struct Animation {
        struct Frame {
            String8 name;
            FileMap* map;   
            mutable GLuint tid;
            // 以名字来排序
            bool operator < (const Frame& rhs) const {
                return name < rhs.name;
            }
        };
        struct Part {
            int count;
            int pause;
            String8 path;
            // 排序的 vector 
            SortedVector<Frame> frames;
            bool playUntilComplete;
        };
        int fps;
        int width;
        int height;
        Vector<Part> parts;
    };
```

当然，可以自己弄一些其他格式的，例如说只有一个参数的，代表背景颜色。这个主要是我弄的一个芯片 GPU 性能不行， 但是产品偏要用非黑色背景的 bootanimation，要用非黑色背景的只能把图片做成全屏的，但是 GPU 不行，绑纹理速度太慢了，贴全屏的图片卡得和幻灯片一样。所以我就多搞了一个配置，用来指定背景颜色（默认 GL clear color 是黑色的）。这里 GL 调用是很原始的，要自己手动调用 eglSwapBuffers(mDisplay, mSurface); 才会刷新。

```cpp
    for (;;) {
        const char* endl = strstr(s, "\n");
        if (!endl) break;
        String8 line(s, endl - s);     
        const char* l = line.string(); 
        int fps, width, height, count, pause;
        char path[256];       
        char pathType;
        // 3个参数的
        if (sscanf(l, "%d %d %d", &width, &height, &fps) == 3) { 
            //ALOGD("> w=%d, h=%d, fps=%d", width, height, fps);
            if (mReverseAxis) {            
                animation.width = height;      
                animation.height = width;      
            } else {
                animation.width = width;       
                animation.height = height;     
            }
            animation.fps = fps;           
        }
        // 4个参数的
        else if (sscanf(l, " %c %d %d %s", &pathType, &count, &pause, path) == 4) {
            //ALOGD("> type=%c, count=%d, pause=%d, path=%s", pathType, count, pause, path);
            Animation::Part part;          
            part.playUntilComplete = pathType == 'c';
            part.count = count;            
            part.pause = pause;            
            part.path = path;
            animation.parts.add(part);     
        }
        // 我自己加得一个参数的，指定背景颜色的
        else if (sscanf(l, "%d", &bkColor) == 1) {
            // add by hmm@dw.gdbb.com      
            // for support specific bootanimtion background color.
            // due our GPU is pool, it's can't use full screen(1280x800 or 1920x1080) bitmap as
            // bootanimtion frame, so we can set background color to reduce the bitmap size.
            // the background is 10 hex base on RGB(e.g: 0x028cd6 ==> 167162).
            bkR = (float)((bkColor & 0x00ff0000) >> 16) / 255.0f; 
            bkG = (float)((bkColor & 0x0000ff00) >>  8) / 255.0f; 
            bkB = (float)(bkColor & 0x000000ff)         / 255.0f;
            ALOGD("read bkColor=0x%x, r=%f, g=%f, b=%f", bkColor, bkR, bkG, bkB);
        }

        s = ++endl;
    }
```

特别注意一点，关于 bootanimation .zip 这个包打 zip 包的时候，里面的资源文件只能选择 store 模式（即存储模式），因为在代码里面， bootanimation 里面只认 zip 的 store 模式的文件（即不支持压缩）。打包的时候不要选择压缩文件，否则开机动画播不出来的。用 zip -r -0 bootanimation.zip part0/ part1/ desc.txt 就可以指定以存储方式打包，不压缩。

```cpp
    // read all the data structures
    const size_t pcount = animation.parts.size();
    for (size_t i=0 ; i<numEntries ; i++) {
        char name[256];
        ZipEntryRO entry = zip.findEntryByIndex(i);
        if (zip.getEntryFileName(entry, name, 256) == 0) {
            const String8 entryName(name);
            const String8 path(entryName.getPathDir());
            const String8 leaf(entryName.getPathLeaf());
            if (leaf.size() > 0) {
                for (int j=0 ; j<pcount ; j++) {
                    if (path == animation.parts[j].path) {
                        int method;
                        // 只支持存储格式的 png 文件
                        // supports only stored png files
                        if (zip.getEntryInfo(entry, &method, 0, 0, 0, 0, 0)) {
                            if (method == ZipFileRO::kCompressStored) {
                                FileMap* map = zip.createEntryFileMap(entry);
                                if (map) {
                                    Animation::Frame frame;
                                    frame.name = leaf;
                                    frame.map = map;
                                    Animation::Part& part(animation.parts.editItemAt(j));
                                    part.frames.add(frame);
                                }
                            }
                        }
                    }
                }
            }
        } 
    }
```

rk 的修改的 bootanimation 支持的开机音效 ogg 文件固定为/system/media/audio/boot.ogg ，打包 system 的时候放自己的 boot.ogg 到这个路径下面就有开机音效了，不放就没有（放的方法参照定制系统音效那里）。

制作好了 bootanimation.zip  在打包 system 的时候 copy 到system/media 下面。修改 device/rockchip/Hx/Hx.mk 添加
PRODUCT_COPY_FILES += \
    $(LOCAL_PATH)/bootanimation.zip:system/media/bootanimation.zip

当然上面这些框框条条可以自行修改 bootaniamtion 这个模块，增加新的参数，实现新的功能（以前君正的开机音效就是增加了一个参数，把音效 ogg 文件一起打包进 bootanimation.zip 里面去了）。

这里贴一下 rk 播音效的代码，可以参考一下， native 层调用 MediaPlayer 播放音频文件：

```cpp
void BootAnimation::playMusic()
{
    char property[PROPERTY_VALUE_MAX];
    bool enable = true;
    if ( property_get( BOOT_MUSIC_PROPERTY, property, "true" ) > 0 ){
        enable = !strcmp(property, "true");
    }

    ALOGD( "BootAnimation::playMusic...enable:%s", enable ? "true" : "false" );

    if( enable ){
        sp<MediaPlayer> mp = new MediaPlayer();
        // 这里好像是检测能否正常访问音频文件
        if ((0 == access(BOOTMUSIC_FILE, F_OK)) && mp != NULL) {
            mp->setDataSource(BOOTMUSIC_FILE, NULL);
            mp->prepare();
            mp->start();
        }
    }
}

```
 

