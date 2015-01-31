title: window 在 cygwin 中使用 repo ssh
date: 2015-01-31 11:42:16
categories: [Window]
tags: [window, install]
---

其实在 cygwin 中弄 ssh 是为使用在 window 中使用 repo 。如果单纯为了在 window 上 ssh 登陆 linux 服务器的话，直接用 putty 就行了，简单方便，无需安装、配置。

## 安装 cygwin

从[cygwin web](http://cygwin.com/setup.exe "cygwin web") 下载一个很小的 setup.exe ，然后第一选从网上安装（第一次装好后，会把安装文件缓存到你指定的目录，下次就可以选中从本地安装了）。如果选择从网上安装的话，之后会让你选择安装源（我选的163的那个，看上去就像是国内的源）。之后就会解析源信息（无论是在线的、还是本地的），如果出现说：提示你这个 setup.exe 有新版，然后出现解析失败的话，先要去它提示你的网址下载最新的 setup.exe，然后就会解析成功了。

最后一步是让你选择要安装的包，如果你硬盘够大，又不在意安装时间的话，可以全部选择安装（在那些分类上点击出现  install 就行了）。默认是 default，default 的话，它会安装一些最基本的 linux 的包（例如 bash 这些），然后如果是之前安装过的，default 是会保留你之前安装的包。所以如果你要卸载的话 ，就把它点成 uninstall，还有一个 reinstall 是重新安装。

这些我只是要用 repo 和 git （svn也算上吧）而已，所以把 Net 大类和 Python （repo要用 python）大类点上（点成 install 状态，然后点开看到具体的软件的版本号就算行了），然后在搜索栏上敲 ssh，把出来的也点上（一般就是  openssh），然后还可以点些自己需要的东西，例如 vim、unzip、sqlite 等。我为了省空间强制干掉了 X11、Gonme、KDE、Graphic、Audio 的东西，然后也没点啥编译的东西（gcc、make 之类的玩意，这些东西在 Devel 这个类下，想在 window 下要编 NDK 的要勾上这些了）。

选好后，点下一步就等进度条就行了。

## 配置 cygwin

cygwin 是个在 window 上模拟的 linux 环境，很多和 linux 差不多。我们可以先配一下，以后用着顺手。一开桌面上的快捷方式，它会在你的安装目录下以你当前 window 用户创建一个用户目录，例如我的安装目录是 C:\cygwin ，我当前的 window 用户是 Administrator，那就会创建这么目录。然后在这个目录下会自动创建 .bashrc_histroy 等一些文件。顺带说一下，这个目录就是你的 home 目录了，然后在安装目录下就有 etc、usr、lib、dev、bin 等 linux 的目录。然后我们可以自己用得习惯的 bash 配置文件 .bashrc 放到自己的 home 目录（还可以放一些别的软件配置文件，例如 .srceenrc, .vimrc 等）。不过别以直接丢 .bashrc 到 home 就完事了，你再次打开 bash 发现是无效的。因为 cygwin bash 的配置文件不在这个 home 目录下，而是在 etc 下的 bash.bashrc 。当然你可以在 bash.bashrc  里改，但是我建议还是把你自定义的配置文件放到 home 目录下，然后在 bash.bashrc 的最后加上 source ~/.bashrc 就 OK 了。这样做的好处的能最大的兼容 cygwin ，因为你打开 bash.bashrc 会发现它其实设置了一些东西的，所以最好不要乱动别人的东西。

## 配置 sshd 

ssh 的客户端不需要配置什么东西，直接 ssh username@host 就可以用。不过一上来就这么用的话，会发现会报啥 connect to xx port  22: Conection  refused 之类的错误的。这个是因为你的 ssh 服务的守护进程没有启动（也就是 sshd）。我这里的配置大多数是网上的：

如果之前有配置过，但是配错了，想重新配置，就直接敲 ssh-host-config 就行了。但是如果想重新安装，可以用 sc delete sshd 这个命令先把之前安装的 sshd 服务给卸掉。如果创建了一些不想用的window用户，可以去 window 的控制面板去闪除。如果是 cygwin 的用户的话，你直接把 /etc/passwd 和 /etc/group 给删掉就好了，然后再用 mkpasswd -l > /etc/passwd 和 mkgroup -l > /etc/group 重新恢复默认用户和用户组就行了。记得重新分组后，要 chmod +r /ect/passwd 和 chmod +r /ect/group 改下权限，然后 rm -rf /var ，把这些临时文件给删掉。

其实就是直接敲 ssh-host-config ，只不过会一些选项让你选择而已：

```bash
$ ssh-host-config
*** Info: Generating /etc/ssh_host_key
*** Info: Generating /etc/ssh_host_rsa_key
*** Info: Generating /etc/ssh_host_dsa_key
*** Info: Creating default /etc/ssh_config file（如果这里之前安装过，会问你是否覆盖，选 yes 覆盖）
*** Info: Creating default /etc/sshd_config file （如果这里之前安装过，会问你是否覆盖，选 yes 覆盖）

*** Info: Privilege separation is set to yes by default since OpenSSH 3.3.
*** Info: However, this requires a non-privileged account called 'sshd'.
*** Info: For more info on privilege separation read /usr/share/doc/openssh/REAME.privsep.
*** Query: Should privilege separation be used? (yes/no) yes（这里的意思开启 sshd 需要一个有系统管理员权限的用户，说的是 window 的；问你是不是要单独创建一个这样的window用户，网上有说选no，有说选 yes 的，我是选 yes 的，可以启动 sshd）

*** Info: Updating /etc/sshd_config file
*** Info: Creating default /etc/inetd.d/sshd-inetd file
*** Info: Updated /etc/inetd.d/sshd-inetd
*** Warning: The following functions require administrator privileges!
*** Query: Do you want to install sshd as a service?
*** Query: (Say "no" if it is already installed as a service) (yes/no) yes（这里是问你是不是要安装 sshd 服务，肯定要选 yes，必须要装的）
*** Info: Note that the CYGWIN variable must contain at least "ntsec"
*** Info: for sshd to be able to change user context without password.

*** Query: Enter the value of CYGWIN for the daemon: [] ntsec tty（这里好像问的你是使用的终端连接方式吧，照网上的说法填 ntsec tty）
*** Info: On Windows Server 2003, Windows Vista, and above, the
*** Info: SYSTEM account cannot setuid to other users -- a capability
*** Info: sshd requires.  You need to have or to create a privileged
*** Info: account.  This script will help you do so.
*** Info: You appear to be running Windows 2003 Server or later.  On 2003
*** Info: and later systems, it's not possible to use the LocalSystem
*** Info: account for services that can change the user id without an
*** Info: explicit password (such as passwordless logins [e.g. public key
*** Info: authentication] via sshd).
*** Info: If you want to enable that functionality, it's required to create
*** Info: a new account with special privileges (unless a similar account
*** Info: already exists). This account is then used to run these special
*** Info: servers.
*** Info: Note that creating a new user requires that the current account（这里第一次还会有个问题的，就是说如果你用的高于 window xp 的版本，就需要你重新创建一个有管理权限的用户，这个用户是 cygwin 的，这里选 yes，创建）

*** Info: have Administrator privileges itself.
*** Info: No privileged account could be found.
*** Info: This script plans to use 'cyg_server'.
*** Info: 'cyg_server' will only be used by registered services.
*** Query: Do you want to use a different name? (yes/no) no（这里创建的用户默认名字是 cyg_server 问你是不是要改名字，一般不用改）
*** Warning: Privileged account 'cyg_server' was specified,
*** Warning: but it does not have the necessary privileges.
*** Warning: Continuing, but will probably use a different account.
*** Warning: The specified account 'cyg_server' does not have the
*** Warning: required permissions or group memberships. This may
*** Warning: cause problems if not corrected; continuing...

*** Query: Please enter the password for user 'cyg_server': ******（然后让你为刚刚创建的 cyg_server 设密码）
*** Query: Reenter: ******
*** Info: The sshd service has been installed under the 'cyg_server'
*** Info: account.  To start the service now, call `net start sshd' or
*** Info: `cygrunsrv -S sshd'.  Otherwise, it will start automatically
*** Info: after the next reboot.
*** Info: Host configuration finished. Have fun!
```

最后出现这个提示就表示配置成功了。因为新创建了2个用户（一个 window 的，一个 cygwin 的），再用 mkpasswd -l > /etc/passwd 和 mkgroup -l > /etc/group 重新设置下用户信息。如果配置的过程中，出现创建用户不成功的提示，就用最开始说的，把之前创建的用户给删掉；还有在重新配置前先要停掉正在运行的 sshd 服务（net stop sshd）。

然后按提示的，使用 net start sshd 就可以启动 sshd 服务了。成功的话，会出成功的提示（cygwin bash 中文乱码的话，右键鼠标 --> options --> text 设置下编码可以解决）。然后在 window 的控制面板中的 系统安全 --> 管理工具 --> 服务 中可以看到一个叫 CYGWIN sshd 的服务，点开属性，登陆那里可以看到是使用刚刚创建的用户在启动的。

然后就可以使用 ssh user@host 来登陆了。如果使用密码登陆，记得把服务器的 /etc/ssh/sshd_config 中的 PasswordAuthentication 设置成 yes，如果使用密钥进行登陆就把 RSAAuthentication 和 PubkeyAuthentication 设置成 yes （具体使用密钥的方式，可以参看我的另外一篇叫 Rsa 验证的笔记）。

然后再记下，window 的 hosts 文件的位置在 C:\Windows\System32\drivers\etc 下，默认是隐藏、只读的系统文件。

最后，搞好 ssh 后，终于可以在 window 下用 repo 和 git pull、push 或是 sync 代码了。


