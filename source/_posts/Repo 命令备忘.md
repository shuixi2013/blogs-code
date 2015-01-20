title: Repo 命令备忘
date: 2015-01-13 20:37:16
tags: [linux]
---

repo 其实本身就是一个 git 仓库，只不过这个仓库记录了别的 git 仓库的地址，版本号信息而已。这个 git 仓库在项目的根目录的 .repo/mainfests 下面。其实就是一个 xml 文件，然后里面记录了各个项目的版本号而已，repo 自身是一个脚本。

## download repo
 下载 repo 的地址: http://android.git.kernel.org/repo ，可以用 wget http://android.git.kernel.org/repo 或者 curl http://android.git.kernel.org/repo>~/bin/repo  来下载 repo , Repo脚本授权：chmod a+x ~/bin/repo

## repo ini
repo init -u URL ,  在当前目录安装 repository ，会在当前目录创建一个目录 ".repo"  -u 参数指定一个URL，从这个URL 中取得repository 的 manifest 文件。  

<pre>
repo init -u git://android.git.kernel.org/platform/manifest.git  
</pre>

可以用 -m 参数来选择 repository 中的某一个特定的 manifest 文件，如果不具体指定，那么表示为默认的 namifest 文件(default.xml)    repoinit -u git://android.git.kernel.org/platform/manifest.git -m dalvik-plus.xml 可以用 -b 参数来指定某个manifest 分支。

<pre>
repo init -u git://android.git.kernel.org/platform/manifest.git -b release-1.0
</pre>

可以用命令: repo help init 来获取 repo init 的其他用法。

## 查看 repo 可用的版本信息
先 repo init 把 repo 仓库安装下来，然后 cd ./repo/manifests 下，然后 git branch -a 就可以看到了。

repo branch 也可以看到当前 repo 用的分支。

## repo sync [project-list]
下载最新本地工作文件，更新成功，这本地文件和repository 中的代码是一样的。可以指定需要更新的project ，如果不指定任何参数，会同步整个所有的项目。如果是第一次运行 repo sync ，则这个命令相当于 git clone ，会把 repository 中的所有内容都拷贝到本地。如果不是第一次运行 repo sync ，则相当于 git remote update ;  git rebaseorigin/branch .  repo sync 会更新 .repo 下面的文件。如果在merge 的过程中出现冲突，这需要手动运行  git  rebase --continue

## repo update[ project-list ]
上传修改的代码，如果你本地的代码有所修改，那么在运行 repo sync 的时候，会提示你上传修改的代码，所有修改的代码分支会上传到 Gerrit (基于web 的代码review 系统), Gerrit 受到上传的代码，会转换为一个个变更，从而可以让人们来review 修改的代码。

## repo diff [ project-list ]
显示提交的代码和当前工作目录代码之间的差异。

## repo download  target revision
下载特定的修改版本到本地，例如:  repo downloadpltform/frameworks/base 1241 下载修改版本为 1241 的代码

## repo start newbranchname
创建新的branch分支。 "." 代表当前工作的branch 分支。

##  repo prune [project list]
删除已经merge 的 project

## repo forall [ project-lists] -c command
对每一个 project 运行 command 命令。例如你要用 repo 创建一个新的分支（其实就是让每个项目创建一个分支）： 

<pre>
# repo 本地创建一个 xx 分支
repo forall -c git branch xx
# repo 创建远程分支（把本地分支 push 到服务器上去）
# origin 是默认的远程路径名字
repo forall -c git push origin xx:xx
</pre>

## repo status
显示 project 的状态

## 签名错误
有些时候 repo 的时候会出现

<pre>
gpg: 于 2013年10月02日 星期三 00时44分27秒 CST 创建的签名，使用 RSA，钥匙号 692B382C
gpg: 无法检查签名：找不到公钥
</pre>

这样的错误。这是因为前后 key 不一样导致的，把 home 目录下的 .repoconfig 删掉，让 repo 自动导入一次 key 就可以了。

## 创建新的远程分支
首先要创建对应的 git 的各个仓库的分支，可以参加前面 repo forall 命令的使用。然后去 .repo/mainfests 这个仓库下创建自己的 repo 分支。然后可以自己创建分支所用的 xml 文件，提交到刚刚新创建的 repo 分支就可以了。然后就可以用下面的命令 down 新分支代码了：

<pre>
repo init -u git://android.git.kernel.org/platform/manifest.git -b my-branch -m my-branch.xml
repo sync
</pre>

## 创建远程仓库
1. 先本地库的创建，在一个本地一个目录把要创建仓库的目录创建好：
<pre>
# 在本地仓库目录，创建裸仓库
git init
# 把本地仓库先提交到一个分支，一般开始是 master
git add
git commit
</pre>

2. 提交远程服务器
命令： git remote add <name> <url>
如:
<pre> 
git remote add origin gitrepo:android/android4.1/platform/external/abc.git
</pre>
origin 是远程仓库的别名（路径），然后后面的 url 一定要是远程服务器的 url 。弄好后可以用 git remote -v 或是 git config -l 查看。如果一开始远程别名弄错了，可以用 git remote rename old new 来重命名。如果 url 错了，可以用 git remote set-url origin xx 来重新指定远程服务器 url 。确保正确后，用 git push origin master:master 把本地仓库提交远程服务器。

3. 在 repo 中添加新的仓库
进入 .repo/manifests 这个 repo 仓库，编辑 default.xml (视自己项目使用的 xml 不用，编辑不同的 xml 文件)。照着原有的模版添加刚刚新创建的仓库就好。例如： `<project path="external/abc" name="platform/external/abc" />`。然后把修改提交到自己项目的 repo 仓库分支就 OK 了，例如: git push origin default:default。

