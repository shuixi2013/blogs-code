title: ubuntu 开启 ftp 服务
date: 2015-01-19 10:03:16
tags: [linux]
---

## 安装vsftpd 
直接从源里面安装：
<pre>
sudo apt-get install vsftpd
</pre>

安装完毕后或许会自动生成一个帐户"ftp"，/home下也会增加一个文件夹。
如果没有生成这个用户的话可以手动来，生成了就不用了：
<pre>
sudo useradd -m ftp
sudo passwd ftp
</pre>

有"ftp"帐户后还要更改权限：
<pre>
sudo chmod 777 /home/ftp
</pre>

在这个目录下我建立一个文件夹专门保存需要共享的内容

## 配置文件
通过sudo gedit /etc/vsftpd.conf修改。配置文件比较简单，如下：

<pre config="brush:bash;toolbar:false;">
#独立模式启动
listen=YES

#同时允许4客户端连入，每个IP最多5个进程
max_clients=200
max_per_ip=4

#不允许匿名用户访问，允许本地（系统）用户登录
anonymous_enable=NO
local_enable=YES
write_enable=NO

#是否采用端口20进行数据传输
connect_from_port_20=YES

#生成日志
xferlog_enable=YES

#指定登录转向目录
local_root=/home/ftp/ftp
</pre>

这样，在同局域网的电脑上，用我的IP地址，用帐号"ftp"和对应密码就可以登录了，密码是第一步里面passwd那句指定的。就这样就结束了，请大家拍砖！！对了，更改配置后不要忘了重启ftp服务：
<pre>
sudo /etc/init.d/vsftpd restart
</pre>

此外还有开启关闭服务的命令：
<pre>
sudo /etc/init.d/vsftpd start
sudo /etc/init.d/vsftpd stop
</pre>

