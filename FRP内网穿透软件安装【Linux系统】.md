## FRP内网穿透软件安装【Linux系统】

### 

### 查看当前Linux主机的硬件架构类型

```sh
[root@susu ~]# arch
x86_64
```

### 

### x86_64对应下载

### frp_0.43.0_linux_amd64.tar.gz

<br/>

### 软件下载地址

### https://github.com/fatedier/frp/releases

<br/>

### 服务端安装  vim frps.ini

```sh
[common]
#1、这个是你当前的frp服务器ip，默认0.0.0.0就是接受所有来者
bind_addr = 0.0.0.0

#2、这个默认端口是7000，就是客户端和服务端通信的，不是你转发的那个端口
# 如果这个要改了，客户端的7000也要改成和这里一样。
bind_port = 7000

#3、这个是frp的web管理控制台的用户名密码，可以通过127.0.0.1:7500登录
dashboard_user = admin
dashboard_pwd = admin

#4、管理控制台的默认端口
dashboard_port = 7500

#5、转发后的服务端口：
[TOMCAT]
vhost_tomcat_port = 8888
[SSH]
vhost_ssh_port = 2222

#6、心跳连接：必须得有，frp 0.43.0版本如果不加，60秒就会自动断开连接！
# 服务器就加这一条，客户机每台都要加。
heartbeat_timeout = 30

```

### 启动服务端

```sh
nohup ./frps -c frps.ini    //nohup表示不中断服务，后台运行
```

<br/>

### 客户端配置  vim frpc.ini

```sh
[common]
#1、你frp服务器的公网ip地址
server_addr = xx.xx.xx.xx
#2、你frp服务器的通信端口，默认是7000，如果你服务器端改了，这里也要跟着改。
server_port = 7000 

[range:tcp]
#3、协议类型
type = tcp
#4、你当前内网服务器的网卡IP地址，不要用127.0.0.1
local_ip = 192.168.126.128
#5、你要转发的服务端口
local_port = 80,22
#6、你要映射到公网上的那个端口
remote_port = 8888,2222

#7、服务器与客户机之间的心跳连接：如果没有，每隔60秒就会自动断开连接！！！
heartbeat_timeout = 30

```

### 启动客户端

```sh
nohup ./frpc -c frpc.ini
```