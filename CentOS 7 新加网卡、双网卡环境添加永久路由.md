## CentOS 7 新加网卡、双网卡环境添加永久路由

### 1.添加网卡【添加了ens192】

```sh
[root@localhost ~]# ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:9b:f7:ba brd ff:ff:ff:ff:ff:ff
    inet 1.10.10.249/24 brd 1.10.10.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::c126:ec44:b6be:d28b/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:9b:32:cc brd ff:ff:ff:ff:ff:ff
    inet 1.10.1.249/24 brd 1.10.1.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::5277:bf15:4385:f112/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

```

### 网络显示、但是配置文件没有。

<br/>

### 2.查看网卡配置信息

```sh
[root@localhost ~]# nmcli con show
NAME         UUID                                  TYPE      DEVICE 
ens160       e808f8a5-6bdc-48b4-b3db-48c88715989d  ethernet  ens160 
有线连接1    3eca808d-113e-4439-b2af-f3665005506a  ethernet  ens192 

```

<br/>

### 3.生成网卡信息

```sh
nmcli con add con-name ens192 type ethernet ifname ens192

执行后会自动生成ifcfg-ens192配置文件
```

<br/>

### 4.修改配置文件、重启网络

```sh
systemctl restart network
```

<br/>

<br/>

## 双网卡添加路由

### 1.临时添加

```sh
route add -net 1.10.1.0/24 gw 1.10.1.1 dev ens192
```

<br/>

### 2.永久添加

### 新建对应网卡的路由配置文件

```sh
vim /etc/sysconfig/network-scripts/route-ens192
```

### 编辑配置文件

```sh
1.10.1.0/24 via 1.10.1.1 dev ens192
1.10.2.0/24 via 1.10.1.1 dev ens192
1.10.3.0/24 via 1.10.1.1 dev ens192
1.10.4.0/24 via 1.10.1.1 dev ens192
1.10.5.0/24 via 1.10.1.1 dev ens192

```

### 重启网络、查看路由

```sh
systemctl restart network
route -n
```
