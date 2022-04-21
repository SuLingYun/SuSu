# kali系统docker安装、docker安装awvs、nessus

## 安装docker

### 第一步：添加Docker官方的GPG密钥

```sh
curl -fsSL https://download.docker.com/linux/debian/gpg |apt-key add
```

```sh
┌──(root㉿kali)-[~]
└─# curl -fsSL https://download.docker.com/linux/debian/gpg |apt-key add
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
OK
```

### 第二步：更新源

```sh
echo 'deb https://download.docker.com/linux/debian stretch stable'> /etc/apt/sources.list.d/docker.list
```

### 第三步：直接导入证书

```sh
apt-get install apt-transport-https  ca-certificates  curl  gnupg2  software-properties-common
```

### 第四步：系统更新

```sh
apt-get update && apt-get upgrade
```

### 第五步：安装docker 

```sh
apt install docker.io
```

### 第六步：检查docker安装是否成功

```sh
docker -v
```

### 第七步：docker安装成功后，需要安装compose

```sh
apt install docker-compose
查看是否安装成功docker-compose
```

## docker常用命令

```sh
service docker start   //启动docker服务
 
docker version   //查看docker版本信息
 
docker ps  //查看容器
 
docker  images   //查看已有的镜像

systemctl enable docker   //开机自启

docker search awvs    //搜索需要的镜像

docker pull xiaomimi8/awvs14-log4j-2022  //拉取镜像文件

docker inspect stupefied_ishizaka    //查看镜像详细信息

docker run -d -p 3443:3443 --restart=always  xiaomimi8/awvs14-log4j-2022
//启动镜像文件、把端口映射到本地服务器、开机自动启动镜像

1. 删除指定容器
docker rm -f <containerid>

2. 删除未启动成功的容器
docker rm $(docker ps -a|grep Created|awk '{print $1}')
或者
docker rm $(docker ps -qf status=created)

3. 删除退出状态的容器
docker rm $(docker ps -a|grep Exited|awk '{print $1}')
或者
docker rm $(docker ps -qf status=exited)

4. 删除所有未运行的容器
docker rm $(docker ps -a -q) #正在运行的删除不了，所有未运行的都被删除了
或者
docker container prune #Docker 1.13版本以后，可以使用 docker containers prune 命令，删除孤立的容器
```

```sh
┌──(root㉿kali)-[~]
└─# docker ps    //查看镜像                                                           
CONTAINER ID   IMAGE                         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
c6cc271173b5   xiaomimi8/awvs14-log4j-2022   "/bin/sh -c 'echo 12…"   37 minutes ago   Up 35 minutes   0.0.0.0:3443->3443/tcp, :::3443->3443/tcp   stupefied_ishizaka
                                                                                                                                                                              
┌──(root㉿kali)-[~]
└─# docker stop stupefied_ishizaka    //关闭镜像

删除容器
首先使用docker stop 容器ID 命令停止容器，再docker rm -f 容器ID 进行删除
```

## 安装awvs

```sh
docker search awvs    //搜索需要的镜像
docker pull xiaomimi8/awvs14-log4j-2022  //拉取镜像文件
docker run -d -p 3443:3443 --restart=always  xiaomimi8/awvs14-log4j-2022
//启动镜像文件、把端口映射到本地服务器、开机自动启动镜像
awvs默认的用户名和密码
username: admin@admin.com
password: Admin123
```

## 安装nessus

```sh
docker pull leishianquan/awvs-nessus:v03    //拉取镜像文件
docker run -it -d -p 8834:8834 --restart=always  leishianquan/awvs-nessus:v03
docker ps –a    //查看容器
docker start 容器id    //启动容器
docker exec –it 容器id /bin/bash    //进入容器
/etc/init.d/nessusd start    //启动nessus服务

nessus username:leishi
nessus password:leishianquan
```