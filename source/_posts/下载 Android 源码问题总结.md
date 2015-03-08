title: 下载 Android 源码问题总结
date: 2015-03-06 17:15:16
updated: 2015-03-06 17:15:16
categories: [Android Framework]
tags: [android, install]
---

## 多余临时文件

目前下载源码，最好找个稳定点的 VPN 和稳定点的网络环境下载。因为如果中途断掉的话，好像是没断点续传的。repo 会一个一个的下 git 仓库，但是如果某个仓库下载到一半断掉了，好像下次就得重新下这个仓库。所以这里又会导致另一个问题，如果你 repo sync 的时候经常中断，你会发先的你的源码文件夹最变得很大。那是因为目前的 android 源码的 repo 在下载 git 仓库的时候在创建一个 `tmp_pack_XXXX` 的 临时文件，然后在这个仓库下载完成后删除。但是由于前面说的仓库下载中断的话，会重头开始下，这个临时文件也会重新创建（2次的名字不一样， XXXX 那里不同），所以如果经常中断的话，这些临时文件会越来越多。可以在 repo sync 完成后，手动删除：

```bash
find . -iname tmp_pack_* | xargs rm
```

顺带说下这些 tmp_pack 文件在每个仓库的 objects/pack/ 下面，例如： device/lge/hammerhead-kernel.git/objects/pack/，可以直接 repo 的根目录敲上面的命令。再顺带提一下，如果过了几个大版本的话，可以考虑重新下载 android 的 mirror，因为可能会有比较大的文件结构调整，但是在老版本的基础上 repo sync 不会把以前一些不用的文件删掉，也是导致 repo 仓库变得很大。

## mirror 缺少某一个仓库

google 的人感觉有些时候也会犯一些错误。在使用最近新下载的 5.0 mirror 的时候报某几个仓库不存在。打开 mirror 的 .repo/manifests/default.xml 和指定 android 版本的（5.0.2-r1）的仓库的 .repo/manifests/default.xml 对比，发现 mirror 确实少了一些仓库。感觉应该是 google 的漏了几个仓库吧。不过还好我以前 4.4 的 mirror 还没删掉，去 4.4 的 mirror 的 .repo/manifests/default.xml 能找到缺少仓库以前的地址和分支，照着地址去 gogole 的源码服务器看一下，发现 git 仓库还在，那应该是漏写在 repo 的 xml 里面了。

那咋们还是有曲线救国的方法的：找出缺少的仓库后，然后去 4.4 的 mirror 的 .repo/manifests/default.xml 依葫芦画瓢的把地址和分支抄过来，然后再在 5.0 的仓库中 repo sync 一下就下载到缺少的仓库了。当然既然 4.4 的 mirror 还在，还可以利用下以前下载好的资源，可以去 4.4 的 mirror 把缺少的仓库整个 .git 文件夹 copy 到 5.0 的 mirror 对应的路径下。我发现 5.0 缺少的这几个仓库，好像和 4.4 是一样的，repo sync 并没多下载东西。下面附上我找到的 5.0.2-r1 缺少 git 仓库：

<pre>
platform/bootable/bootloader/legacy.git
platform/external/arduino.git
platform/external/chromium_org/third_party/openssl.git
platform/external/gcc-demangle.git
platform/external/google-diff-match-patch.git
platform/external/openfst.git
platform/external/qemu.git
platform/external/qemu-pc-bios.git
platform/external/smack.git
platform/external/stressapptest.git
platform/external/yaffs2.git
platform/prebuilts/gcc/darwin-x86/mips/mipsel-linux-android-4.8.git
platform/prebuilts/gcc/darwin-x86/mips/mips64el-linux-android-4.8.git
platform/prebuilts/gcc/linux-x86/mips/mipsel-linux-android-4.8.git
platform/prebuilts/gcc/linux-x86/mips/mips64el-linux-android-4.8.git
platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.6.git
platform/tools/tradefederation.git
</pre>

既然我们可以手动修复 mirror 缺少 git 仓库的问题，这里又引申出解决另外一个问题的办法。就是前面说的 git 仓库下载中断，又要从头开始下，如果 VPN 或是网络不稳定，下载某些很大的 git 仓库会十分痛苦，但是某些 git 仓库其实你并不会用到。例如上面说的那个 lg 的 kernel 的 git 仓库，有 10G 左右，但是如果你的 nexus 设备并不是 lg 的话，根本不需要这个仓库。所以如果实在下不下来的话，可以在 mirror 的 .repo/manifests/default.xml 中删掉这个 git 仓库。


