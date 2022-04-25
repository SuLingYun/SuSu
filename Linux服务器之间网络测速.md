## Linux服务器之间网络测速

### 服务端配置：

```sh
yum -y install iperf3
firewall-cmd --zone=public --add-port=5201/tcp
firewall-cmd --zone=public --add-port=5201/udp
iperf3 -s  //启动服务
```

### 客户端配置：

```sh
yum -y install iperf3
iperf3 -c  <服务端ip>
```