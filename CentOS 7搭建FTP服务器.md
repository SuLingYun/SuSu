## CentOS 7搭建FTP服务器

### 1.安装vsftpd

```sh
yum -y install vsftpd
```

### 2.修改vsftpd配置文件

```sh
> /etc/vsftpd/vsftpd.conf
cat >> /etc/vsftpd/vsftpd.conf << EOF
#是否允许匿名，默认no
anonymous_enable=NO

#这个设定值必须要为YES 时，在/etc/passwd内的账号才能以实体用户的方式登入我们的vsftpd主机
local_enable=YES

#具有写权限
write_enable=YES

#本地用户创建文件或目录的掩码
local_umask=022

#当dirmessage_enable=YES时，可以设定这个项目来让vsftpd寻找该档案来显示讯息！您也可以设定其它档名！
dirmessage_enable=YES

#当设定为YES时，使用者上传与下载日志都会被纪录起来。记录日志与下一个xferlog_file设定选项有关
xferlog_enable=YES
xferlog_std_format=YES

#开启20端口
connect_from_port_20=YES

#这个是pam模块的名称，我们放置在/etc/pam.d/vsftpd
pam_service_name=vsftpd

userlist_enable=YES

#当然我们都习惯支持TCP Wrappers的啦！
tcp_wrappers=YES

#通过搭配能实现以下几种效果： 
#①当chroot_list_enable=YES，chroot_local_user=YES时，在/etc/vsftpd.chroot_list文件中列出的用户，可以切换到其他目录；未在文件中列出的用户，不能切换到其他目录。 
#②当chroot_list_enable=YES，chroot_local_user=NO时，在/etc/vsftpd.chroot_list文件中列出的用户，不能切换到其他目录；未在文件中列出的用户，可以切换到其他目录。 
#③当chroot_list_enable=NO，chroot_local_user=YES时，所有的用户均不能切换到其他目录。 
#④当chroot_list_enable=NO，chroot_local_user=NO时，所有的用户均可以切换到其他目录。

chroot_local_user=YES
chroot_list_enable=NO

#不添加下面这个会报错：500 OOPS: vsftpd: refusing to run with writable root inside chroot()
allow_writeable_chroot=YES

#启动被动式联机(passivemode)
pasv_enable=YES
#上面两个是与passive mode 使用的 port number 有关，如果您想要使用65400到65410 这 11 个 port 来进行被动式资料的连接，可以这样设定
pasv_min_port=65400
pasv_max_port=65500

#FTP访问目录
local_root=/data/ftp/

EOF
```

### 3.创建FTP目录

```sh
mkdir -p /data/ftp/
```

### 4.创建ftp访问用户

```sh
useradd -d /data/ftp/ -s /sbin/nologin ftpuser
passwd ftpuser
chown -R ftpuser /data/ftp
```

### 5.vim /etc/pam.d/vsftpd

```sh
#auth       required    pam_shells.so
```

### 6.启动ftp服务

```sh
systemctl enable vsftpd.service && systemctl start vsftpd.service && systemctl status vsftpd.service 
```