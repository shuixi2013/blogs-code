title: Rsa 验证简介
date: 2015-01-13 20:33:16
updated: 2015-01-13 20:33:16
categories: [Linux]
tags: [linux]
---

 如果不是用密码来进行验证的话，那么就可以使用 rsa 数据签名来进行验证。签名分为公钥和私钥2个。公钥是可以公开出来的，密钥是自己个人持有的。一般来使用 RSA 验证的话，是自己生成一对公钥/私钥。然后把公钥放置到服务器上，自己持有公钥和私钥，然后就可以通过公钥来验证了。下面上使用简介：
 

假设客户端的用户 charlee 要以 guest 用户登录到服务器上。首先在客户端执行下面的命令:

<pre>
// ssh-keygen: 
// -t 指定公钥类型，默认的是 rsa 的
// -c 是生成的公钥注释，如果不指定的话就是自己机子的终端显示的那个东西
// 其它的一些选项可以自己去查帮助文档
</pre>

<pre>
[charlee@client:~]$ ssh-keygen -t rsa -c "your_email@youremail.com"
Generating public/private rsa1 key pair.
Enter file in which to save the key (/home/charlee/.ssh/id_rsa): 回车 （这个是让你指定输出文件，直接敲回车就是括号里那个默认的路径、文件）
Enterpassphrase (empty for no passphrase):  回车 （这个是让你输入你公钥的密码，直接敲回车就不设定密码）
Enter same passphrase again:   回车
Your identification has been sabed in /home/charlee/.ssh/id_rsa
Your public key has been saved in /home/charlee/.ssh/id_rsa.pub
</pre>

生成的文件保存在主目录的 .ssh 目录下，id_rsa 为客户端密钥，id_rsa.pub 为客户端公钥。之后，通过 U 盘等方式将公钥 id_rsa.pub 复制到服务器上，并执行下列命令。

[guest@server:~]$ cat id_rsa.pub >> .ssh/authorized_keys

其实还可以执行 ssh-copy-id mingming@192.168.0.8 把公钥弄到服务器上。其中 id_rsa.pub 是客户端的用户 charlee 的公钥。这样在客户端即可通过以下的命令连接服务器。到此基本可以配对两台机的公钥了。

[charlee@client:~]$ ssh -l guest server

若不想每次登录服务器时都输入密码，可以先执行下列命令：

<pre>
// 如果提示 ssh not find agent 之类的，可以输入 eval `ssh-agent` 启动 ssh agent
[charlee@client:~]$ ssh-add
Enter passphrase for /home/charlee/.ssh/id_rsa: 输入密码
Identity added: /home/charlee/.ssh/id_rsa (/home/charlee/.ssh/id_rsa)
</pre>

以后登录服务器就不需要输入密码了。

如果出现： Permissions 0464 for '.ssh/id_rsa' are too open. 之类的错误，把私钥文件（id_rsa）权限改为 600 就可以了。

