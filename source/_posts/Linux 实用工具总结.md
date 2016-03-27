title: Linux 实用工具总结
date: 2016-03-27 10:59:16
updated: 2016-03-27 10:59:16
categories: [Linux]
tags: [Linux]
---

现在我开发基本上都用 linux(ubuntu) 了，window 已经变成我的娱乐系统了。linux 下有不少比较实用的工具的，这里稍微记录一下。

## pngquant

pngquant 是一个图片压缩工具，可以压缩 png 图片，是**有损**的。虽然说是有损的，但是一般调节一下压缩参数，在手机上基本上看不出来，但是体积却可以小一半以上。别告诉我现在写 apk，还是美工给你多大的图片，你就塞多大的去 apk 里面，那太凹凸了。我个人很讨厌体积很大的 apk ... ... 。这个工具可以通过 apt-get 来安装：

<pre>
sudo apt-get install pngquant
</pre>

然后使用方法查一下 help 就指定怎么用了，我稍微写了一个脚本（png_quant）可以直接压缩指定文件夹下的所有 png：

```bash
#!/bin/bash

#
# author: mingming.killer@gmail.com
#

# $1: path
# $2: compress quality: 0 - 100
function func_compress()
{
    local path=$1
    local quality=$2
    local file=""
    ls $path | while read image
    do
        echo "compress $image to $quality"
        file=${path}"/"${image}
        #echo "resize $file to $size"
        pngquant -f --ext .png --quality $quality-$quality $file
    done
}

# ==================================
# main entry:
if [ "$1" = "" -o $# -lt 2 ]; then
    echo "Usage: $0 path quality"
    exit 1
else
    func_compress $1 $2
fi
# ==================================
```

用法就是： png_quant pics/ 75  ，就会把当前目录下 pics 文件夹下所有的 png 文件按 75% 的质量进行压缩（替换原文件）。有些时候一些 png 使用 75 会出现压缩无效的情况（就是图片大小没变），可以尝试调节下压缩比较到 60。

## convert

convert 是一个图片缩放工具。有些时候我们需要批量改变图片的大小，这个工具就十分有用了。也是可以通过 apt-get 来安装的：

<pre>
sudo apt-get install imagemagick
</pre>

同样也是写了一个脚本（image_resize）来实现缩放指定文件夹下所有图片的：

```bash
#!/bin/bash

#
# author: mingming.killer@gmail.com
#

# $1: path
# $2: resize pre
function func_resize()
{
    local path=$1
    local size=$2
    local file=""
    ls $path | while read image
    do
        echo "resize $image to $size"
        file=${path}"/"${image}
        #echo "resize $file to $size"
        convert -resize $sizex$size $file $file
    done
}

# ==================================
# main entry:
if [ "$1" = "" -o $# -lt 2 ]; then
    echo "Usage: $0 path resize"
    exit 1
else
    func_resize $1 $2
fi
# ==================================
```

用法就是： image_resize pics/ 60  ，就会把当前目录下 pics 文件夹下所有的图片件缩放到原来的 60%（替换原文件）。

## 音频转化工具

### lame

lame 是 mp3 转化 wav 工具，安装：
<pre>
sudo apt-get install lame
</pre>

用法：
<pre>
lame --decode sound.mp3 sound.wav
</pre>

### vorbis-tools

vorbis-tools 是 ogg 转 wav 工具，安装：
<pre>
sudo apt-get install vorbis-tools
</pre>

用法：
<pre>
oggdec sound.ogg -o sound.wav
</pre>




