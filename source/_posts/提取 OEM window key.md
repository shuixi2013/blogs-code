title: 提取 OEM window key
date: 2015-01-31 15:55:16
updated: 2015-01-31 15:55:16
categories: [Window]
tags: [window]
---

如果买带 window 的笔记本，自带的 window 都是 OEM 授权的正版来的，有 OEM key 的。但是这个 key OEM 厂商并没有主动显式的提供给用户。这样如果自己重新装系统就没办法正常激活正版 window 了。也许 OEM 就想着用户不要自己装系统。不过这个 key 都在我们的笔记本里了，咋还能拿不到么， JS 果然是没办法和广大网友斗的。

## 提取 OEM key

首先说下，这个 key 是写在 BIOS 里面的，每台笔记本唯一对应一个 key，所以这个 key 提取出来就被想着给别共享了，没用的，这个 key 只能自己用。首先用一个工具去把 BIOS 里面的 key 给读出来： [RwPortable-V1.6-x64](http://pan.baidu.com/s/1sjLNbDr "RwPortable-V1.6-x64")。当然网上还有其它的工具，但是我是这个的，感觉还挺好用的。

刚刚那个工具解压后打开 rw.exe 程序，参考下图依次点击 ACPI-->MSDM 选项。图中 Data 后面对应的字符就是电脑内置的 Win8/8.1 激活密钥啦。然后把这串东西自己拿个 txt 保存一下就好了（JS 让你奸）。

![](http://7u2hy4.com1.z0.glb.clouddn.com/window/dump-OEM-key/1.jpeg)

## 选择正确的 window 版本

提取了 key 之后，自己重新装系统的时候还要选择正确的 window 版本才能用 OEM key 激活的。因为 OEM key 是对应特定的 window 版本的，这个就是你笔记本预转的那个 window 版本。这个可以用 Win+R 打开运行，输入“slmgr.vbs /dlv”，回车即可看到预装系统版本信息（所以在没把信息收集全前，不要那么快把预装的系统干掉）。例如，Windows 8中文版查看结果如下图所示：

![](http://7u2hy4.com1.z0.glb.clouddn.com/window/dump-OEM-key/2.jpeg)

一般 win8 有如下几种版本：

<pre config="brush:bash;toolbar:false;">
Win8单语言版本（CoreSingleLanguage）
Win8特定国家版（CoreCountrySpecific，Win8中文版属于该版本）
Win8普通版（Core）
Win8专业版（Professional）
Win8专业版含媒体中心（ProfessionalWMC）
Win8企业版（Enterprise）
</pre>

预装的一般都是啥特定国家版（例如针对中国的版本），找到版本后，从网上找对应的版本的 iso 下就行了。然后在网上可以找得到这个版本得安装 key（注意是安装 key，这个 key 可以用来安装，但是装好的系统是非激活状态的，还要前面提取的 key 进行激活，前面的提取的 key 是不能用来安装的，有点麻烦的说）。其实没必要上啥旗舰版的（特定国家版好像就是高级家庭版加了点中文），预装版基本够用了（至少现在我没发现缺啥的东西）。关键正版还是挺爽的，不用担心被黑和去网上各种找 key，还是我还是喜欢官方原版的 OS，啥 xx 版的真的没原版舒服。

这里给个 win8.1 的版本下载地址噻： [Windows 8.1 正式版镜像下载大全](http://www.iruanmi.com/windows-8-1-rtm-iso-download/ "Windows 8.1 正式版镜像下载大全")


