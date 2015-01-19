title: 如何生成 Patch 补丁
date: 2015-01-19 10:12:16
tags: linux
---

一般有 2 种方式：

## 2个文件夹生成 patch

在2个文件夹之前生成 patch 使用 diff -Nur a b 命令（a、b 2个文件夹）。注意，这样生成 patch 的时候，要把一些不需要的临时文件清理掉（例如 tag、编译的中间文件等），这样才能保证 patch 是干净的。可以重定向到 xx.patch 文件，这样 patch 就生成好了。

## git 仓库不同版本生成 patch

在 git 仓库的不同版本间生成 patch 使用 git diff commita commitb 命令。commit 是每个 git 版本的那一串很长的数字。不过注意这个只能比较已经提交了的。例如说，这个命令如果什么都参数都不加的话，就是比较当前 git 版本和现在 git 仓库代码的区别。不过如果你当前新加入了一些文件，是 diff 不出来的，需要提交以后才能 diff 。这个也是可以通过重定向到 xx.patch 文件的。

## git commit 修改了二进制文件

上面那个 git diff 生成的 patch 是普通的文本 patch，用 vi 就可以看到 patch 的内容。但是这种方式不适用于修改了二进制文件的提交（例如修改了图片）。这种文本 patch 无法包含二进制文件的修改的。所以这个时候就使用：

<pre>
git diff --binary commita commitb > xx.patch 
</pre>

来生成二进制形式的 patch。这种 patch 要使用 git apply patch 来打。打之前可以使用 git apply --check patch 来检测 patch 是否有错误。有些时候打 patch 会出现一些空白符号的警告错误，不用理会就行。

有些时候 git diff a b 生成的 patch 会有点不对（你不加 binary 去看 patch 能看出来，patch 多改了一些东西）。如果只是单个 commit 的 patch 可以使用:

<pre>
git show --binary commit > xx.patch
</pre>

来生成 patch，这样就对了（其实只要能打出 diff 的命令都可以生成 patch）。

## 总结

以上 2 种方式打出来的 patch 都是 p1 的，还有这个 px 的问题，需要在打 patch 的时候注意。

