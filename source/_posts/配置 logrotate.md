title: 配置 logrotate
date: 2016-02-19 23:39:16
updated: 2016-02-19 23:39:16
categories: [Server]
tags: [server]
---

对于服务器来说 log 是很重要的，通过 log 能够分析服务器运行是否正常，以及监控访问等。但是如果不加以限制的话，log 很容易就会爆满。例如说之前我们公司的 nginx 的访问 log 就把服务器的磁盘爆满了，导致磁盘没空间，然后访问各种异常。可能有人觉得 log 应该不会那么大，最开始我也是这么认为了，但是我们公司的 nginx 一天也就几十万次的请求，然后 nginx 默认会把每次请求记录下来，然后几个星期后，几十G 的 log 就产生了，加上我挪用了下服务器用来下 android 源码（大概 2、30G这样），服务器 128G 的 ssd 就爆了（我们公司买的 Linode 比较便宜的服务器作为一个节点）。

然后就会想能不能让 log 文件限制在一个可控的范围内。翻了下资料，就发现 linux 自带的工具 logrotate 就是干这个事情的。

## 简介

logrotate 从名字看叫 log 轮转，其实它的作用是，可以定时的把 log 转储成备份文件，然后重新开始记录 log，然后还能限制转储文件的数量。这样既能将 log 大小控制在一定范围内，又保留的最近的 log 记录，便于查看，分析。

## 配置

logrotate 的配置文件是 /etc/logrotate.conf:

```bash
# see "man logrotate" for details
# rotate log files weekly
weekly

# use the syslog group by default, since this is the owning group
# of /var/log/syslog.
su root syslog

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# uncomment this if you want your log files compressed
#compress

# packages drop log rotation information into this directory
include /etc/logrotate.d

# no packages own wtmp, or btmp -- we'll rotate them here
/var/log/wtmp {
    missingok
    monthly
    create 0664 root utmp
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0660 root utmp
    rotate 1
}

# system-specific logs may be configured here
```

我们先来介绍一下 logrotate 一些主要参数的含义：

* **compress**
通过 gzip 压缩转储以后的日志，加了这个参数生成的归档文件就是个 gzip 压缩包

* **nocompress**
不需要压缩时，用这个参数

* **copytruncate**
用于还在打开中的日志文件，把当前日志备份并截断
 
* **nocopytruncate** 
备份日志文件但是不截断

* **create mode owner group**
转储文件，使用指定的文件模式创建新的日志文件

* **nocreate** 
不建立新的日志文件

* **delaycompress**
和 compress 一起使用时，转储的日志文件到下一次转储时才压缩

* **nodelaycompress**
覆盖 delaycompress 选项，转储同时压缩。

* **errors address** 
专储时的错误信息发送到指定的Email 地址

* **ifempty**
即使是空文件也转储，这个是 logrotate 的缺省选项。

* **notifempty**
如果是空文件的话，不转储

* **mail address**
把转储的日志文件发送到指定的E-mail 地址

* **nomail**
转储时不发送日志文件

* **olddir directory**
转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统

* **noolddir**
转储后的日志文件和当前日志文件放在同一个目录下

* **prerotate/endscript** 
在转储以前需要执行的命令可以放入这个对，这两个关键字必须单独成行

* **postrotate/endscript**
在转储以后需要执行的命令可以放入这个对，这两个关键字必须单独成行

* **daily**
指定转储周期为每天

* **weekly**
指定转储周期为每周

* **monthly**
指定转储周期为每月

* **rotate xx**
指定日志文件删除之前转储的次数，0 指没有备份，5 指保留5 个备份

* **size xx**
当日志文件到达指定的大小时才转储，size 可以指定 bytes (缺省)以及KB (sizek)或者MB (sizem).

## 例子

我们先来看看上面 logrotate.conf 中的配置，默认写了一些，其中有一个 include /etc/logrotate.d 这就会引用 /etc/logrotate.d/ 下面的配置文件。其实应用都是把自己的配置文件写在 logrotate.d 下面的。像安装 nginx 的时候就会在 /etc/logrotate.d/ 下面生成一个默认的 nginx 文件的默认配置：

```bash
/var/log/nginx/*.log {
    weekly
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 0640 www-data adm 
    sharedscripts
    prerotate
        if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
            run-parts /etc/logrotate.d/httpd-prerotate; \
        fi \
    endscript
    postrotate
        [ -s /run/nginx.pid ] && kill -USR1 `cat /run/nginx.pid`
    endscript
}
```

logrotate 的配置，都是一个 log 的文件路径（这个路径就是你要配置的应用写 log 的路径，可以看到支持正则表达匹配的，同一个应用，不同类型的 log 可以用不同的配置），然后后面接一对 {} 做为一个配置。对照上面的参数含义，我们可以看到 nginx 默认是一个星期才转储一次，然后会保留 52 个转储文件，对于我们的服务器来说，一个星期的 nginx 的 log 就有十来G了，然后还保留将近2个月的，就算压缩过，我们那可怜的 128G ssd，也肯定撑不住。所以我决定给它改改。

然后上面还注意一下， prerotate/endscript 和 postrotate/endscript 这个参数。这个2个参数分别是能够在转储前和转储后，执行指定的操作，操作由特定的脚本语言描述，相当于是一个简单脚本语言了，这里不详细说明这个语言。就稍微解释下为什么需要这2个参数，是因为要转储的话，就需要移动原来正在写的 log 文件，然后让程序写新的 log，那这个功能肯定不同的程序不一样了，logrotate 不可能完成的，所以就留了一个接口（可编程的脚本语言）让每个程序自己实现。这里解释下上面 nginx 做的，转储前的话，是看下 /etc/logrotate.d/httpd-prerotate 这个文件是否存在，存在的话就运行一下？（好像是这个意思吧）。转储后，就找到 nginx 当前运行的 pid，然后发送一个信号量给 nginx 的进程，让其重启，就会写新的 log 文件了。

这里我把 nginx 的转储配置改为了每天转储一次，然后转储文件数量为3：

```bash
/var/log/nginx/*.log {
    daily
    missingok
    rotate 3
    compress
    delaycompress
    notifempty
    create 0640 www-data adm 
    sharedscripts
    prerotate
        if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
            run-parts /etc/logrotate.d/httpd-prerotate; \
        fi \
    endscript
    postrotate
        [ -s /run/nginx.pid ] && kill -USR1 `cat /run/nginx.pid`
    endscript
}
```

