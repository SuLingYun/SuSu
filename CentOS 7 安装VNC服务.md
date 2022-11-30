## CentOS 7 安装VNC服务

### 

### 1.安装

```sh
yum -y install tigervnc-server tigervnc
```

### 2.创建vncserver@:1.service

```sh
cp /lib/systemd/system/vncserver@.service /lib/systemd/system/vncserver@:1.service
```

### 3.修改文件

```sh
vi /lib/systemd/system/vncserver@:1.service
```

### 将文件中的<USER>改成root或者你需要有登陆权限的其他用户

```sh
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=simple

# Clean any existing files in /tmp/.X11-unix environment
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/usr/bin/vncserver_wrapper root %i
ExecStop=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'

[Install]
WantedBy=multi-user.target
```

### 4.vncserver root用户密码

```sh
[root@localhost ~]# vncpasswd
Password:
Verify:
Would you like to enter a view-only password (y/n)? n    //不添加只读密码
A view-only password is not used
```

### 5.重载

```sh
systemctl daemon-reload
```

### 6.启动

```sh
systemctl start vncserver@:1.service  && systemctl status vncserver@:1.service && systemctl enable vncserver@:1.service
```

### 7.VNC连接

```sh
VNC连接工具
IP地址 端口号 5901
或者
192.168.0.1:1
```

## 如果想创建多个用户

### 1.配置该用户vncpasswd

```sh
su - test
 
vncpasswd
```

### 2.新建vncserver@:2.service，创建其他用户，以此类推

```sh
cp /lib/systemd/system/vncserver@.service /lib/systemd/system/vncserver@:2.service
```

### 3.修改<USER>为用户名 test

```sh
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=simple

# Clean any existing files in /tmp/.X11-unix environment
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/usr/bin/vncserver_wrapper test %i
ExecStop=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'

[Install]
WantedBy=multi-user.target
```

### 4.删除

```sh
rm -rf /lib/systemd/system/vncserver@.service
```

### 5.启动

```sh
systemctl start vncserver@:2.service  && systemctl status vncserver@:2.service && systemctl enable vncserver@:2.service
```