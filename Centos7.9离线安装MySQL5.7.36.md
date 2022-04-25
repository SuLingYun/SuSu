## Centos7.9离线安装MySQL5.7.36

### 一、mysql-5.7.36-el7-x86_64.tar.gz安装包下载

```sh
https://downloads.mysql.com/archives/community/  //官网下载地址

Product Version:      5.7.36
Operating System:     Red Hat Enterprise Linux / Oracle Linux
OS Version:           Red Hat Enterprise Linux 7 / Oracle Linux 7 (x86, 64-bit)
```

### 二、卸载系统自带mariadb

```sh
[root@localhost ~]# rpm -qa|grep mariadb  //查询安装的mariadb软件
mariadb-libs-5.5.68-1.el7.x86_64
[root@localhost ~]# yum remove -y mariadb-libs-5.5.68-1.el7.x86_64  //删除软件包及相关依赖
[root@localhost ~]# rm -rf /etc/my.cnf  //删除遗留的目录
[root@localhost ~]# rm -rf /var/lib/mysql
```

### 三、创建安装用户和组

```sh
groupadd mysql  //创建mysql用户组
useradd -r -g mysql -s /bin/false mysql  //-s /bin/false参数指定mysql用户仅拥有所有权，而没有登录权限
```

### 四、创建安装目录，生产环境最好新加一块硬盘，保障系统崩溃情况下可以通过挂载硬盘的方式快速恢复数据库

```sh
[root@localhost /]# mkdir data  //创建安装目录
[root@localhost data]# ll
总用量 715496
-rw-r--r--. 1 root root 732667171 4月  23 17:45 mysql-5.7.36-el7-x86_64.tar.gz  //上传安装包到data
[root@localhost data]# tar -zxvf mysql-5.7.36-el7-x86_64.tar.gz  //解压安装包
[root@localhost data]# mv mysql-5.7.36-el7-x86_64/ mysql  //重命名
[root@localhost data]# cd mysql/
[root@localhost mysql]# mkdir {binlog,log}  //创建binlog,log目录
[root@localhost mysql]# cd log/
[root@localhost log]# touch mysqld.log   //创建mysql.log文件
[root@localhost log]# touch slow.log   //创建slow.log文件
[root@localhost data]# chown -R mysql:mysql /data/  //修改data文件夹包含子文件的用户、属组
[root@localhost data]# chmod 744 -R /data/  //修改data文件夹包含子文件的权限
```

### 五、创建/etc/my.cnf文件，填入以下内容，注意路径

```sh
[mysqld]
datadir=/data/mysql/data
socket=/data/mysql/mysql.sock
user=mysql
bind-address = 0.0.0.0
port=3306
symbolic-links=0
pid-file=/data/mysql/mysqld.pid
lc-messages-dir=/data/mysql/share/
server-id=1

max_connections=1000
character-set-server=utf8mb4
collation-server=utf8mb4_general_ci
default-storage-engine=INNODB
max_allowed_packet=16M
default-time_zone = '+8:00'
log-error=/data/mysql/log/mysqld.log
lower_case_table_names=1
log-bin = /data/mysql/binlog/mysql-bin.log
log-bin-index =/data/mysql/binlog/binlog.index
log_bin_trust_function_creators=1
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
innodb_buffer_pool_size = 6G
innodb_log_buffer_size = 64M
innodb_log_file_size = 256M
innodb_log_files_in_group = 2
innodb_file_per_table = 1
innodb_data_file_path = ibdata1:1G:autoextend
innodb_temp_data_file_path=ibtmp1:500M:autoextend:max:4096M
innodb_status_file = 1
max_connections=1000
max_connect_errors=100
max_user_connections=400
long_query_time = 120
slow_query_log=1 
log_output='File' 
slow_query_log_file=/data/mysql/log/slow.log

[mysqld_safe]
log-error=/data/mysql/log/mysqld.log
pid-file=/data/mysql/mysqld.pid

[mysql.server]
basedir=/data/mysql
default-character-set=utf8mb4
```

### 六、修改环境变量

```sh
[root@localhost ~]# vim /etc/profile
export PATH=$PATH:/data/mysql/bin  //末尾添加
[root@localhost ~]# source /etc/profile  //刷新环境变量
```

### 七、建立软连接

```sh
ln -s /data/mysql/mysql.sock /tmp/mysql.sock
```

### 八、初始化MySQL

```sh
[root@localhost ~]# cd /data/mysql/bin
[root@localhost bin]# mysqld --defaults-file=/etc/my.cnf --initialize-insecure --user=mysql

如果报错：
mysqld: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory

安装libaio.so.1的依赖库
yum -y install libaio
```

### 九、mysqld服务配置

```sh
vim /usr/lib/systemd/system/mysqld.service  //新建mysqld.service文件，填入以下内容
```

```sh
[Unit]
Description=MySQL Server
Documentation=man:mysqld(5.7)
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql

# Have mysqld write its state to the systemd notify socket
Type=forking

# Disable service start and stop timeout logic of systemd for mysqld service.
TimeoutSec=0

# Start main service
ExecStart=/data/mysql/bin/mysqld --defaults-file=/etc/my.cnf --daemonize 


# Sets open_files_limit
LimitNOFILE = 655350

Restart=on-failure

RestartPreventExitStatus=1

PrivateTmp=false
```

```sh
chmod 644 /usr/lib/systemd/system/mysqld.service  //添加权限
systemctl enable mysqld.service  //添加开机自启
systemctl start mysqld.service  //启动MySQL数据库 
systemctl status mysqld.service -l  //查看MySQL服务状态
systemctl stop firewalld  //关闭防火墙
或者
firewall-cmd --zone=public --add-port=3306/tcp --permanent  //在防火墙上添加端口3306
firewall-cmd --reload  //重新载入防火墙配置
firewall-cmd --zone=public --query-port=3306/tcp  //查看添加端口是否生效
```

### 十、登录数据库修改密码、配置远程连接

```sh
[root@localhost ~]# mysql  //登录数据库
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.36-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> alter user 'root'@'localhost' identified by '123456';  //修改数据库密码
Query OK, 0 rows affected (0.00 sec)

mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> update user set user.Host='%' where user.User='root';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select user,host from user;
+---------------+-----------+
| user          | host      |
+---------------+-----------+
| root          | %         |
| mysql.session | localhost |
| mysql.sys     | localhost |
+---------------+-----------+
3 rows in set (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)

mysql> 
```

```sh
alter user 'root'@'localhost' identified by '123456';
use mysql;
update user set user.Host='%' where user.User='root';
select user,host from user;
flush privileges;
```