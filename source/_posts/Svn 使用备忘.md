title: Svn 使用备忘
date: 2016-12-07 17:24:16
updated: 2016-12-07 17:24:16
categories: [Linux]
tags: [linux]
---

我个人是觉得 git 比 svn 好用得多的 ... 但是公司版本管理用的是 svn，那也得用咯。

## 常用命令

* svn co

git checkout ... 可以接完整仓库的 url，例如： https://svn.yy.com/repos/src/dwmobile/kktest 。也可以接某个分支的 url，例如： https://svn.yy.com/repos/src/dwmobile/kktest/android/trunk 。一般开发用 trunk 的话，就只下载 trunk 就行了，不然时间久了，tags 和 branch 一多，一个 svn 的完整仓库能有几十G（svn 一个 tags 或是 branch 就是完整的 copy 整个项目文件过去的，和 git 比起来，简直就是原始时代） 。

如果要添加用户名和密码的话，可以接下面的参数：

<pre>
--username username --password passwd 
</pre>

如果要下载指定 commit 的可以加 -rxx ，例如说：

<pre>
svn co https://svn.yy.com/repos/src/dwmobile/kktest --username username --password passwd -r1395089
</pre>


* svn add, rm

git add， rm ... svn add 没 --all 命令，文件需要一个一个自己添加 ... 我都不想吐槽了 ... 还有一个 **rename** 命令，用来重命名文件名，如果你直接自己修改文件名，那么 svn 无法识别出这个仅仅是文件名变动了，需要 rm 旧的，再 add 新的（git 简直完爆） ... 


* svn info

查看 svn 仓库的信息 url，branch 等。


* svn log

git log ... 但是如果 commit 多了，速度非常慢，而且会从头一直显示到尾，简直就不想让在终端下用 ... 终端下的话，只要重定到文件，再查看（我已经不想说 git log 比这个好多少了 ... ）。

如果要想要 git log -p file 的效果（查看指定文件的修改记录列表），可以这么使用： **svn log --diff file**


* svn st

git status ... 这个终于相差不大了


* svn diff

git diff ... 这个也相差不大， svn diff file -rcommit1:commit2 这个可以查看指定文件2个版本之间的修改


* svn merge

git merge ... 简单的使用还是差不多的（目前我也没用到太复杂的）


* svn revert

撤销修改


## 代码提交流程里
* 先确定下本地的修改： svn st

* 看下diff: svn diff

* 提交修改代码：svn add xx/svn rm xx

* 确认提交修改：svn commit -m"xx"

* 如果没冲突的话就，提交上去了，svn up 更新一下版本号。如果别人有修改，就 svn up 更新一下，如果有冲突就解决冲突后再 commit

* 可以查看下提交记录： svn log


## 忽略规则配置
svn 的忽略规则比和 git 相比麻烦死了 ... 需要在对应的文件夹下编辑，例如说：

<pre>
svn propedit svn:ignore android/trunk/demo
</pre>

然后就会弹出一个编辑界面，就可以添加 ignore 文件了（规则和 git 的类似），用 vi 编辑保存退出后，提交就可以。**但是** svn 是分文件夹的，所以说你得为每个文件夹添加一次 ... 而且编辑的文件夹还必须已经纳入 svn 的管理（也就是文件夹要先提交到 svn）


## 分支管理

### 创建

在 svn 中主干代码一般是放置在 trunk 目录下的，如果要新建 branch 的话则放置在 branchs 目录下。(注意这是一种约定，svn 并不强制你这样做) 注意 branhs 和 trunk 目录要平级，不能有嵌套，要不会引起混乱。

<pre>
  myproject/
      trunk/
      branches/
      tags/
</pre>

但是我们公司移动的项目是这样的（其实也没太大关系）：

<pre>
  myproject/
	  android/
        trunk/
        branches/
        tags/
      ios/
        trunk/
        branches/
        tags/
      docs/
</pre>


创建一个 branch 也相当简单，只需要一条命令即可：

<pre>
svn copy http://example.com/repos/myproject/trunk http://example.com/repos/myproject/branches/releaseForAug -m 'create branch for release on August'
</pre>

从 trunk 中创建一个 releaseForAug 的分支。之后你就可以通过 svn co http://example.com/repos/myproject/branches/releaseForAug 来迁出你的 branch 源文件，在上面进行修改和提交了。

其实 svn 并没有 branch 的内部概念。我们只是创建了一个 repo 的副本，并自己赋予这个副本作为 branch 的意义，所以这与 git 中的 branch 有很大不同。需要注意的是 branch 和 trunk 使用同一套版本号，也就是说无论在 branch 还是 trunk 的提交都会引起主版本号的增加。这是因为 svn copy 只支持同一个 repo 内的文件 copy，并不支持跨 repo 的 copy，所以新创建的 branch 和 trunk 都属于同一个 repo。

### 合并

既然要创建分支也需要合并分支。基本的合并也是蛮简单的。假设现在 branch 上 fix 了一系列的 bug，现在我们想把针对 branch 的改变同步到 trunk 上，那么应该怎么做那？保证当前 branch 分支是 clean 的，也就是说使用 svn st 看不到任何的本地修改。命令行下切换到 trunk 目录中，使用 svn merge http://example.com/repos/myproject/branches/releaseForAug 来将 branch 分支上的改动 merge 回 trunk 下。

如果出现 merge 冲突则进行解决，然后执行svn ci -m 'description' 来提交变动。当然在 merge 你也可以指定 branch 上那些版本变更可以合并到 trunk 中:

<pre>
svn merge  http://example.com/repos/myproject/branches/releaseForAug -r150:HEAD
</pre>






