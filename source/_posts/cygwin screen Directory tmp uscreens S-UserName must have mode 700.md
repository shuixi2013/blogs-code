title: cygwin screen Directory tmp uscreens S-UserName must have mode 700
date: 2015-01-31 11:49:16
categories: [Window]
tags: [window]
---

win8.1 cygwin 目录权限问题很多。使用 cygwin 自带安装的 screen 会报上面的错误。S-UserName 就是你 cygwin 的用户名。有2种解决办法：

## 修改文件夹所有者

一开始我根据它的提示直接去改这个 S-UserName 这个文件夹的权限为 700 （chmod -R 700 S-UserName）。但是发现没用，还是报这个错。google 到一个老外说要把这个文件夹的所属用户组改一下：
   * chgrp Users S-UserName （cat /etc/group 可以看得到 cygwin 下面有一个 Users 组的）
   * chmod -R 700 S-UserName （改完用户组，再改一次权限就 OK 了）

##  修改源码

在这里 [screen-4.1.0-cygwin-sock-permission.patch](https://gist.github.com/pasela/3401354 "screen-4.1.0-cygwin-sock-permission.patch") 可以找得到 screen 4.1.0 的一个 patch，就是在 cygwin 的环境下忽略这个文件夹的权限检测：

```cpp
diff --git a/src/screen.c b/src/screen.c
index 6e19732..3a8ca3e 100644
--- a/src/screen.c
+++ b/src/screen.c
@@ -1102,8 +1102,10 @@ char **av;
 	      n = (eff_uid == 0 && (real_uid || (st.st_mode & 0775) != 0775)) ? 0755 :
 	          (eff_gid == (int)st.st_gid && eff_gid != real_gid) ? 0775 :
 		  0777;
+#if !defined(__CYGWIN__)
 	      if (((int)st.st_mode & 0777) != n)
 		Panic(0, "Directory '%s' must have mode %03o.", SockDir, n);
+#endif
 	    }
 	  sprintf(SockPath, "%s/S-%s", SockDir, LoginName);
 	  if (access(SockPath, F_OK))
@@ -1133,8 +1135,10 @@ char **av;
       if ((int)st.st_uid != real_uid)
 	Panic(0, "You are not the owner of %s.", SockPath);
     }
+#if !defined(__CYGWIN__)
   if ((st.st_mode & 0777) != 0700)
     Panic(0, "Directory %s must have mode 700.", SockPath);
+#endif
   if (SockMatch && index(SockMatch, '/'))
     Panic(0, "Bad session name '%s'", SockMatch);
   SockName = SockPath + strlen(SockPath) + 1;
```

可以在 cygwin 中下载 screen 4.1.0 的 source 下来（要编译的话还得把 devel 那一票工具下下来，automake、make 子类的，还得把 cygwin 的开发头文件下下来）。源码位置在 /usr/src/ 下面，是一个 tar.bz2 的压缩包，自己解压一下。照着打 patch，然后 autogen、configure、make，重新编译出 screen.exe ，然后把这个 exe 替换到 /usr/bin 下面就可以了。用是可以用了，不过好像我自己打过 patch 编出来的有些小问题，退出的时候老报个什么错误，快捷键也有点小错误。所以还是第一种方法靠谱点，这个纯粹是逗自己玩（正好我用的 cygwin screen 版本是 4.1.0 的）。


