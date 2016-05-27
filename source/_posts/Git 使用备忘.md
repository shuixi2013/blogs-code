title: Git 使用备忘
date: 2015-01-13 20:13:16
updated: 2015-01-13 20:13:16
categories: [Linux]
tags: [linux]
---

简单的 git 使用教程。

##常用命令
* git clone

从服务拷贝代码副本到本地（类似 svn checkout）

* git add

添加本机文件到服务器

* 查看git仓库路径

这个好像没直接的命令可以查看，可以去 git 代码的根目录下的 .git/config 里去看。

* git checkout

切换分支。一般在切换分支前需要 git pull 更新到最新。这个命令，还有另外一个用处，当你想恢复一个文件时候，可以使用 git checkout xx （你还可以先把这个文件先删掉）。

* git branch
    *  git branch name ：创建新的分支（name）。
    *  git branch -a ：查看所有的分支信息。
    *  git branch -d name ：删除本地分支 name。
    *  git push origin :name ：删除远程分支 name。

* git diff

和 svn diff 类似的东西。

* git merge-base branch-A branch-B

找到2个分支最近一次的公共 commit 。

* git merge branch-A

把 branch-A 合并到当前分支。

* git init

创建仓库。如果是在远程的服务器上，一般要用 git init --bare 来创建仓库。 创建仓库的话需要配置一下仓库访问权限，否则别人无法提交代码到你创建的仓库。

1: 修改 config ： 加上 sharedrepositiory = 1 这个属性。

<pre config="brush:bash;toolbar:false;">
[core]
    repositoryformatversion = 0
    filemode = true
    bare = true
    sharedrepository = 1
[receive]
    enyNonFastforwards = true
</pre>

2: 把 object 和 refs 目录（当然你改全部的也行）权限改成其它的人（同一组或者指定组的人）可以有写的权限（chmod 777 就可以）。

* git log

git log 查看 log， git log -p xx 可以单独查看这个文件的修改记录。

* git tag

打标签，查看标签。

* git show

显示一个 commit 的详细信息，还可以显示一个 tag 的详细信息。

## 初始化配置

### 1. 
下面的命令将修改/home/[username]/.gitconfig文件，也就是说下面的配置只对每一个ssh的用户可见，所以每个人都需要做。

* 提交代码的log里面会显示提交者的信息
<pre>
[user]
    git config --global user.name [username]
    git config --global user.email [email]
</pre>

* 在git命令中开启颜色显示
<pre>
[color]
    git config --global color.ui true
</pre>

如果不太确定 .gitconfig 的位置的，可以用 git config  --global 命令来配置，git 会自动保存配置文件的。

### 2. 
下面的命令将修改/etc/gitconfig文件，这是全局配置，所以admin来做一次就可以了。
<pre config="brush:bash;toolbar:false;">
// 配置一些git的常用命令alias
sudo git config --system alias.st status     // git st
sudo git config --system alias.ci commit     // git commit
sudo git config --system alias.co checkout   // git co
sudo git config --system alias.br  branch    // git branch
</pre>

### 3.
也可以进入工作根目录，运行git config -e，这样就只会修改工作区的.git/config文件，但是暂时还用不着. git config文件的override顺序是3>1>2.

## 代码提交流程里
* 先确定下本地的修改： git status

* 看下diff: git diff

* 提交修改代码：git add xx

* 确认提交修改：git commit -m"xx" （-m 是注释信息，偷懒的话可以使用 git commit -am"xx"，可以把上面那一步也省了，不过好像不太好）

* 然后合并别人的代码：git pull （如果有冲突的话，需要解决冲突；有时候无法找到默认的分子，可以用 git pull origin xx 更新指定的分支，这个 origin 是仓库的名字，可以通过 git branch -a 看到，一般是 /remote/xx/branch，xx 就是，默认是 origin）

* 最后提交本地修改代码： git push (如果也是找不到默认分支的话，可以使用 git push origin xx(分支名)，如果不想每次都这么写，可以在第一次提交的时候使用 -u 参数，以后会默认提交到上次提交的分支 )

* 可以查看下提交记录： git log

## 忽略规则配置
在仓库代码目录下可以新建一个叫 .gitignore 的文件来配置提交代码时忽略的文件类型：

<pre>
*.class
*.apk
*.ap_
*.swp
tags
bin/
gen/
doc/
local.properties
proguard/
build.xml
</pre>

## 创建远程分支
要创建一个新的远程分支，首先要创建一个本地分支（git branch xx），例如 git branch test。然后再把这个本地分支 push 到服务器上去：

<pre>
# ：号左边的是本地的分支，右边的是远程分支
# 当然你可以把本地分支 push 到服务器用不同的名字，但是我觉得还是一样的比较好
git push origin test:test
</pre>

然后如果要删掉某个远程分支的话，这么弄就行了：

<pre>
# 把本地的留空就是删除了
git push origin :test
</pre>

## 删除误添加的文件
如果在 git all --all 不小心添加了不想提交的文件（在 git status 中可以可得到添加的文件）。可以使用 git rm --cached xx 删掉误添加的文件。当然这个是还没 commit 的时候，可以这样简单的就删除掉，如果已经 commit 了，就不是这么简单的弄了。所以每次提交的时候都要 status 检查下本次提交的内容。

## 撤销 add 的文件
使用 git reset xx ，撤销 add 的问题，特别适用于使用 add --all 但是发现多添加了文件。注意这个是 add 但是还没 commit 的时候。

## 撤销 commit
如果只是本地 commit 还没 push 到服务器，可以使用 git reset --hard xx（回到某个提交，xx 可以是 commit 的 id）。如果 push 到服务器的，要稍微麻烦点。但是最后每次提交前确定一下，这种操作还是少一点比较好。

## 合并其它仓库的 commit
有些时候，我们需要 merge 其它仓库的 commit 到自己的仓库，这种情况经常出现在基于某些开源的仓库自己定制，然后更新开源仓库的版本。git 提供了很好的功能：

<pre config="brush:bash;toolbar:false;">
# 添加其它仓库的源到自己的仓库
# other 是 path, 可以自己定义的，后面的 url 是其它仓库的地址
git remote add other git@github.com:mingming-killer/K7Utils.git

# pull 刚刚添加的仓库
git fetch other

# 合并其它仓库的指定分支到自己的仓库
# 这里的 merge 就和合并自己的仓库的 commit 一样，如果有冲突的话需要解决冲突
# 合并完成后，就可以 push 合并之后的结果到自己的远程仓库中去了
git merge other/master
</pre>

用 git 的这个功能能够很方便的更新其它仓库的代码，不需要人工搬运（如果没有冲突的话），并且还会保留合并仓库的 log 信息。

## push 其它仓库
上面的的是可以合并其它仓库。如果要更换 git 服务器的话，还可以直接把整个原来的仓库 push 到新服务器去，提交信息也都还在的：

1. 先在新服务器创建一个裸仓库
2. 然后在原来仓库上使用 git remote add other git@github.com:mingming-killer/K7Utils.git 命令添加远程仓库地址
3. 然后切到你要 push 的分支，直接 git push other branch 就可以 push 了。当然最后是裸仓库 push 或者是远程仓库还没修改的，不然有冲突是无法 push 的。






