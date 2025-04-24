## CentOS 7 安装 Percona XtraDB Cluster + Keepalived + HAProxy 多主高可用集群 MySQL 5.7

### 一、规划说明

主机规划：

|节点|功能|IP地址|安装软件|
|:--:|:--:|:--:|:--:|
|PXC1|启动节点|1.10.10.2|Percona XtraDB Cluster 5.7.44  + Keepalived + HAProxy|
|PXC2|常规节点|1.10.10.3|Percona XtraDB Cluster 5.7.44  + Keepalived + HAProxy|
|PXC3|常规节点|1.10.10.4|Percona XtraDB Cluster 5.7.44  + Keepalived + HAProxy|

端口规划：

|软件|功能|端口|
|:--:|--|:--:|
|Percona XtraDB Cluster 5.7.44|数据库|3306 : MySQL 服务   4567 : Galera 集群通信   4568 : IST 增量传输   4444 : SST 全量传输|
|HAProxy|负载均衡|13306：负载端口  80：WEB服务端口|
|Keepalived|VRRP|VIP地址：1.10.10.5|

**数据库服务：1.10.10.5  端口号：13306**

**HAProxy 监控：1.10.10.5 端口号 80**

### 二、搭建Percona XtraDB Cluster集群

1、分别在3个节点设置主机hosts

```sh
cat > /etc/hosts <<EOF
# PXC 集群节点映射（IP + 短主机名 + 域名）
1.10.10.2  pxc1 pxc1.dxgs.com
1.10.10.3  pxc2 pxc2.dxgs.com
1.10.10.4  pxc3 pxc3.dxgs.com
EOF
```

2、分别在3个节点设置主机名

```sh
hostnamectl set-hostname pxc1

hostnamectl set-hostname pxc2

hostnamectl set-hostname pxc3
```

3、主机相关配置，3个节点都需执行

```sh
关闭防火墙：
systemctl stop firewalld.service && systemctl disable firewalld.service

临时关闭SELinux 后续配置永久关闭：
setenforce 0

永久关闭SELinux：
sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config && setenforce 0
```

4、分别在3个节点安装数据库

```sh
# 安装 percona-release 工具（用于管理 Percona 软件仓库）
yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm

# 启用 PXC 5.7 仓库（禁用其他仓库）
percona-release enable-only pxc-57 release

# 启用 Percona 工具仓库（可选，用于安装备份工具如 xtrabackup）
percona-release enable tools release

# 清理YUM缓存
yum clean all && yum makecache

# 安装 PXC 5.7 核心组件
yum install -y Percona-XtraDB-Cluster-57

# 检查安装的 PXC 版本
mysqld --version

# 若需安装 PXC 8.0，将 pxc-57 替换为 pxc-80：
percona-release enable-only pxc-80 release
```

国内镜像源配置：

```sh
yum -y install https://mirrors.tuna.tsinghua.edu.cn/percona/yum/percona-release-latest.noarch.rpm
percona-release enable-only pxc-57 release
percona-release enable tools release
sed -i 's|http://repo.percona.com|https://mirrors.tuna.tsinghua.edu.cn/percona|g' /etc/yum.repos.d/percona-pxc-57-release.repo
sed -i 's|http://repo.percona.com|https://mirrors.tuna.tsinghua.edu.cn/percona|g' /etc/yum.repos.d/percona-tools-release.repo
yum clean all && yum makecache
yum install -y Percona-XtraDB-Cluster-57
```

5、初始化集群，注意只在第一个节点执行【作为启动节点】

```sh
注意：如需更改数据目录
/etc/percona-xtradb-cluster.conf.d/mysqld.cnf
datadir=/var/lib/mysql 修改为 datadir=/data/mysql

# 启动 MySQL 服务
systemctl start mysql

# 获取临时 root 密码（若首次安装）
grep 'temporary password' /var/log/mysqld.log

# 配置MySQL用户、密码【注意HAporxy健康检测用户无密码】
mysql -uroot -p

ALTER USER 'root'@'localhost' IDENTIFIED BY 'sly2025';
update mysql.user set host ='%' where user ='root';
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'sly2025';
GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
CREATE USER 'haproxy_healthy'@'1.10.10.%' IDENTIFIED BY '';
GRANT USAGE ON *.* TO 'haproxy_healthy'@'1.10.10.%';
FLUSH PRIVILEGES;
exit

# 检查haproxy_healthy用户是否可以使用
mysql -h 1.10.10.2 -u haproxy_healthy -p -e "SHOW DATABASES;"

注意：以上执行完成后先停止数据库，因为下一步要用wsrep引导启动第一个节点
systemctl stop mysql
```

6、配置文件修改

**/etc/percona-xtradb-cluster.conf.d/wsrep.cnf**

第一个节点【启动节点】

```sh
[mysqld]
wsrep_provider=/usr/lib64/galera3/libgalera_smm.so
wsrep_cluster_name=pxc-cluster
wsrep_cluster_address=gcomm://1.10.10.2,1.10.10.3,1.10.10.4  # 所有节点IP列表
wsrep_node_name=pxc1              # 修改为 pxc1（每个节点唯一）
wsrep_node_address=1.10.10.2     # 修改为第一个节点的真实 IP
wsrep_sst_method=xtrabackup-v2
wsrep_sst_auth=sstuser:sly2025   # 与第一个节点一致
pxc_strict_mode=ENFORCING
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
```

第二个节点

```sh
wsrep_node_name=pxc2
wsrep_node_address=1.10.10.3
```

第三个节点

```sh
wsrep_node_name=pxc3
wsrep_node_address=1.10.10.4
```

**/etc/percona-xtradb-cluster.conf.d/mysqld.cnf**

第一个节点

```sh
server-id=1
```

第二个节点

```sh
server-id=2
```

第三个节点

```sh
server-id=3
```

7、启动第一个节点【引导节点】

**systemctl start mysql@bootstrap.service**

```sh
# 检查集群状态
mysql> SHOW STATUS LIKE 'wsrep%';
mysql> SHOW STATUS LIKE 'wsrep_cluster_size';
```

8、启动第二节点、第三节点**【必须等启动节点启动完成后才能启动】**

```sh
systemctl start mysql && systemctl enable mysql && systemctl status mysql
```

### 三、安装 HAProxy 启用负载均衡

1、YUM安装haproxy：3个节点都执行

```sh
yum -y install haproxy
```

2、修改配置文件：3个节点都执行**【3个节点的配置一样】**

**/etc/haproxy/haproxy.cfg**

```sh
# ---------------------------------------- 全局配置 ----------------------------------------
global
    # 日志配置：发送到本地127.0.0.1的syslog服务，使用local2设备记录日志
    log         127.0.0.1 local2

    # 安全设置：将进程限制在/var/lib/haproxy目录下运行
    chroot      /var/lib/haproxy
    # PID文件路径
    pidfile     /var/run/haproxy.pid
    # 最大并发连接数
    maxconn     50000
    # 运行用户和组
    user        haproxy
    group       haproxy
    # 以守护进程模式运行
    daemon

    # 启用统计信息套接字（用于动态配置和监控）
    stats socket /var/lib/haproxy/stats

    # 工作进程数量（建议与CPU核心数匹配）
    nbproc      1

    # SSL安全配置：设置默认Diffie-Hellman参数长度
    tune.ssl.default-dh-param 2048


# ---------------------------------------- 默认配置 ----------------------------------------
defaults
    # 默认工作模式为HTTP七层代理
    mode                    http
    # 继承全局日志配置
    log                     global
    # 启用详细HTTP请求日志
    option                  httplog
    # 不记录空连接日志
    option                  dontlognull
    # 启用HTTP长连接优化
    option http-server-close
    # 添加X-Forwarded-For头记录客户端真实IP
    option forwardfor       header HTTP_X_REAL_FORWARDED_FOR
    # 当服务器不可用时重新分配请求
    option                  redispatch
    # 记录健康检查日志
    option log-health-checks
    # 请求失败重试次数
    retries                 3

    # 各类超时设置（单位：秒）
    timeout http-request    30s    # 最大HTTP请求等待时间
    timeout queue           1m     # 最大排队时间
    timeout connect         60s    # 最大连接建立时间
    timeout client          5m     # 客户端最大空闲时间
    timeout server          5m     # 服务器端最大空闲时间
    timeout http-keep-alive 30s    # 保持连接超时时间
    timeout check           10s    # 健康检查超时时间
    # 默认最大连接数
    maxconn                 50000

    # 自定义日志格式（详细记录请求信息）
    log-format %ci:%cp\ [%t]\ %ft\ %b/%s\ %Tq/%Tw/%Tc/%Tr/%Tt\ %ST\ %B\ %CC\ %CS\ %tsc\ %ac/%fc/%bc/%sc/%rc\ %sq/%bq\ %hr\ %hs\ %{+Q}r

    # 启用压缩（gzip算法，压缩指定MIME类型）
    compression algo gzip
    compression type text/plain text/css text/xml text/javascript application/javascript application/x-javascript application/xml application/json image/jpeg image/gif image/png


# ---------------------------------------- MySQL负载均衡配置 ----------------------------------------
listen mysql
    # 监听13306端口（MySQL协议）
    bind :13306
    # 使用TCP四层代理模式
    mode tcp
    # 启用TCP日志
    option tcplog
    # 保持TCP连接活跃
    option tcpka
    # 负载均衡策略：轮询算法
    balance roundrobin
    # MySQL健康检查方法（使用特定用户检测）
    option mysql-check user haproxy_healthy
    
    # 定义后端服务器集群（Percona XtraDB Cluster节点）
    server PXC1_10.2 1.10.10.2:3306 check inter 2000 rise 3 fall 2 weight 2
    server PXC2_10.3 1.10.10.3:3306 check inter 2000 rise 3 fall 2 weight 2
    server PXC3_10.4 1.10.10.4:3306 check inter 2000 rise 3 fall 2 weight 2
    # 参数说明：
    # check       启用健康检查
    # inter 2000  每2秒检查一次
    # rise 3      连续3次成功标记为健康
    # fall 2      连续2次失败标记为异常
    # weight 2    服务器权重


# ---------------------------------------- 状态监控页面配置 ----------------------------------------
listen status *:80
    # 启用统计页面
    stats enable
    # 统计页面访问路径
    stats uri /stats
    # 基础监控页面路径
    monitor-uri /mstats
    # 隐藏HAProxy版本号
    stats hide-version
    # 自动刷新间隔
    stats refresh 60s
    
    # 访问控制列表：
    acl AUTH       http_auth(stats-auth)        # 需要认证
    acl AUTH_ADMIN http_auth_group(stats-auth) admin  # 管理员组权限
    
    # 访问控制规则：
    stats http-request auth unless AUTH        # 未认证用户需登录
    stats admin if AUTH_ADMIN                  # 仅允许admin组用户进行管理操作


# ---------------------------------------- 用户权限配置 ----------------------------------------
userlist stats-auth
    # 用户组定义：
    group admin    users admin     # 管理员组包含admin用户
    group readonly users haproxy   # 只读组包含haproxy用户
    
    # 用户凭证配置：
    user  admin    insecure-password Szm2025  # 管理员账号（建议使用加密密码）
    user  haproxy  insecure-password Szm2025  # 只读账号（建议使用加密密码）
    # 注意：生产环境建议使用加密密码（通过mkpasswd生成）
```

后续检查：

```sh
# 检查配置文件语法：
haproxy -c -f /etc/haproxy/haproxy.cfg
# 启动并配置systemd服务
systemctl start haproxy.service && systemctl enable haproxy.service && systemctl status haproxy.service
```

### 四、安装 Keepalived 启用 VIP 地址、配置检测脚本

1、YUM安装 Keepalived ：3个节点都执行

```sh
yum -y install keepalived
```

2、修改**/etc/keepalived/keepalived.conf**配置文件**【注意3个节点不同】**

第一个节点：**MASTER**

```sh
global_defs {
    router_id LVS_DEVEL           # 路由标识符，集群内唯一，通常用主机名或IP标识
    vrrp_skip_check_adv_addr      # 跳过检查通告地址的合法性（某些网络环境需要）
    vrrp_garp_interval 0          # 禁止发送GARP消息（避免网络干扰）
    vrrp_gna_interval 0           # 禁止发送GNA消息（避免网络干扰）
    script_user root              # 执行检查脚本的用户身份
}

# 定义健康检查脚本（检测HAProxy服务状态）
vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"  # 脚本路径
    interval 2                     # 检查间隔（秒）
    weight -30                     # 检查失败时优先级降低30
    fall 2                         # 连续2次检查失败视为故障
    rise 1                         # 1次检查成功视为恢复
}

vrrp_instance VI_1 {
    state MASTER                   # 初始角色为【主节点】
    interface ens192               # 绑定VIP的物理网卡
    virtual_router_id 100          # 虚拟路由ID，集群内所有节点必须一致（范围1-255）
    priority 110                   # 优先级（主节点最高）
    advert_int 1                   # VRRP通告间隔（秒）
    garp_master_delay 3            # 成为Master后延迟3秒发送GARP包
    garp_master_refresh 5          # Master每隔5秒发送一次GARP包

    # 认证配置（所有节点必须一致）
    authentication {
        auth_type PASS             # 认证类型为密码
        auth_pass Fnn2id8Mp        # 密码（建议用随机字符串）
    }

    # 跟踪健康检查脚本
    track_script {
        chk_haproxy               # 引用上方定义的检查脚本
    }

    # 单播通信配置（主节点声明自己的IP和对端IP）
    unicast_src_ip 1.10.10.2       # 本节点IP地址
    unicast_peer {
        1.10.10.3                 # 对端节点1的IP
        1.10.10.4                 # 对端节点2的IP
    }

    # 跟踪网卡状态（如果网卡故障，放弃Master角色）
    track_interface {
        ens192                    # 监听的网卡名称
    }

    # 虚拟IP配置（VIP）
    virtual_ipaddress {
        1.10.10.5/24              # VIP地址和子网掩码
        brd 1.10.10.255           # 广播地址
        dev ens192                # 绑定的网卡
        label ens192:pxc          # 别名（便于识别）
    }

    # 状态切换通知脚本
    notify_master "/etc/keepalived/notify_action.sh MASTER"   # 成为Master时执行
    notify_backup "/etc/keepalived/notify_action.sh BACKUP"   # 成为Backup时执行
    notify_fault "/etc/keepalived/notify_action.sh FAULT"     # 进入故障状态时执行
    notify_stop "/etc/keepalived/notify_action.sh STOP"       # keepalived停止时执行
}
```

第二个节点：**BACKUP**

```sh
global_defs {
    router_id LVS_DEVEL
    vrrp_skip_check_adv_addr
    vrrp_garp_interval 0
    vrrp_gna_interval 0
    script_user root
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight -30
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state BACKUP                   # 初始角色为【备用节点】
    interface ens192
    virtual_router_id 100
    priority 100                   # 优先级低于主节点
    nopreempt                      # 非抢占模式（即使优先级更高也不抢VIP）
    advert_int 1
    garp_master_delay 3
    garp_master_refresh 5

    authentication {
        auth_type PASS
        auth_pass Fnn2id8Mp
    }

    track_script {
        chk_haproxy
    }

    unicast_src_ip 1.10.10.3       # 本节点IP
    unicast_peer {
        1.10.10.2                 # 主节点IP
        1.10.10.4                 # 其他备用节点IP
    }

    track_interface {
        ens192
    }
    virtual_ipaddress {
        1.10.10.5/24 brd 1.10.10.255 dev ens192 label ens192:pxc
    }
    notify_master "/etc/keepalived/notify_action.sh MASTER"
    notify_backup "/etc/keepalived/notify_action.sh BACKUP"
    notify_fault "/etc/keepalived/notify_action.sh FAULT"
    notify_stop "/etc/keepalived/notify_action.sh STOP"
}
```

第三个节点：**BACKUP**

```sh
global_defs {
    router_id LVS_DEVEL
    vrrp_skip_check_adv_addr
    vrrp_garp_interval 0
    vrrp_gna_interval 0
    script_user root
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight -30
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state BACKUP                   # 初始角色为【备用节点】
    interface ens192
    virtual_router_id 100
    priority 90                    # 最低优先级
    nopreempt                      # 非抢占模式
    advert_int 1
    garp_master_delay 3
    garp_master_refresh 5

    authentication {
        auth_type PASS
        auth_pass Fnn2id8Mp
    }

    track_script {
        chk_haproxy
    }

    unicast_src_ip 1.10.10.4       # 本节点IP
    unicast_peer {
        1.10.10.2                 # 主节点IP
        1.10.10.3                 # 其他备用节点IP
    }

    track_interface {
        ens192
    }
    virtual_ipaddress {
        1.10.10.5/24 brd 1.10.10.255 dev ens192 label ens192:pxc
    }
    notify_master "/etc/keepalived/notify_action.sh MASTER"
    notify_backup "/etc/keepalived/notify_action.sh BACKUP"
    notify_fault "/etc/keepalived/notify_action.sh FAULT"
    notify_stop "/etc/keepalived/notify_action.sh STOP"
}
```

3、创建 HAporxy 检测脚本**【3个节点都一样】**

**/etc/keepalived/check_haproxy.sh**

```sh
#!/bin/bash

# 检查 HAProxy 进程是否存活
count=$(ps -C haproxy --no-header | wc -l)  # 统计 HAProxy 进程数量（不显示表头）

# 判断 HAProxy 进程是否存在
if [ $count -eq 0 ]; then
    # 若 HAProxy 进程数为 0，执行以下操作：
    systemctl stop haproxy         # 停止 HAProxy 服务（避免残留进程干扰）
    systemctl stop keepalived      # 停止 Keepalived 服务（让当前节点放弃 Master 角色）
    exit 1                         # 返回状态码 1（告知 Keepalived 检查失败）
else
    exit 0                         # 返回状态码 0（检查正常）
fi

```

4、创建 Keepalived 状态监控脚本**【3个节点都一样】**

**/etc/keepalived/notify_action.sh**

```sh
#!/bin/bash

# 定义日志文件路径
log_file=/var/log/keepalived.log

# 日志记录函数
log_write() {
    echo "[$(date '+%Y-%m-%d %T')] $1" >> $log_file  # 写入带时间戳的日志
}

# 创建状态文件目录（若不存在）
[ ! -d /var/keepalived/ ] && mkdir -p /var/keepalived/

# 根据传入的状态参数执行对应操作
case "$1" in
    "MASTER" )
        # 成为 Master 时的操作：
        echo -n "$1" > /var/keepalived/state       # 将状态 "MASTER" 写入文件
        log_write " notify_master"                 # 记录日志
        echo -n "0" > /var/keepalived/vip_check_failed_count  # 重置 VIP 检查失败计数器
        ;;

    "BACKUP" )
        # 成为 Backup 时的操作：
        echo -n "$1" > /var/keepalived/state       # 将状态 "BACKUP" 写入文件
        log_write " notify_backup"                 # 记录日志
        ;;

    "FAULT" )
        # 进入故障状态时的操作：
        echo -n "$1" > /var/keepalived/state       # 将状态 "FAULT" 写入文件
        log_write " notify_fault"                  # 记录日志
        ;;

    "STOP" )
        # Keepalived 停止时的操作：
        echo -n "$1" > /var/keepalived/state       # 将状态 "STOP" 写入文件
        log_write " notify_stop"                   # 记录日志
        ;;
    *)
        # 未知状态时的处理：
        log_write "notify_action.sh: STATE ERROR!!!"  # 记录错误日志
        ;;
esac

```

5、添加脚本执行权限、启动 Keepalived 

```sh
# 添加执行权限
chmod +x check_haproxy.sh notify_action.sh

# 启动并配置systemd服务
systemctl enable keepalived.service && systemctl start keepalived.service && systemctl status keepalived.service
```

### 五、日常维护

```sh
# 负载均衡测试:应轮询返回不同节点名称
mysql -h1.10.10.5 -P13306 -uroot -p -e "SHOW VARIABLES LIKE 'wsrep_node_name'"

# 重启其中一个节点
systemctl start mysql
# 异常宕机重启整个集群，为保障数据保持最新，在最后关闭的节点上执行
systemctl start mysql@bootstrap.service

# 查看集群状态
SHOW STATUS LIKE 'wsrep%';

# 查看集群配置参数
SHOW VARIABLES LIKE 'wsrep%';

wsrep_cluster_size: 当前集群中的节点数量。
wsrep_cluster_status: 集群状态，通常为 Primary。
wsrep_ready: 节点是否准备好接收请求，ON 表示正常。
wsrep_connected: 节点是否连接到集群，ON 表示正常。

wsrep_local_state_comment：
Synced: 节点已与集群同步。
Joining: 节点正在加入集群。
Donor/Desynced: 节点正在为其他节点提供 SST（State Snapshot Transfer）。

# 在维护或重启前，安全停止节点以确保数据一致性。
mysql -u root -p -e "SET GLOBAL wsrep_on='OFF';"

执行全量备份：
xtrabackup --backup --target-dir=/backup/full/ --user=backupuser --password=yourpassword
备份后准备：
xtrabackup --prepare --target-dir=/backup/full/

恢复备份：
systemctl stop mysql
sudo rm -rf /data/mysqldata/*
sudo xtrabackup --copy-back --target-dir=/backup/full/
sudo chown -R mysql:mysql /data/mysqldata/

## 备份脚本
#!/bin/bash
BACKUP_DIR=/backup/$(date +%F)
mkdir -p $BACKUP_DIR

xtrabackup --backup --target-dir=$BACKUP_DIR --user=backupuser --password=yourpassword

# 删除7天前的备份
find /backup/ -type d -mtime +7 -exec rm -rf {} \;
```

**日志查看：**

```sh
tail -f /var/log/keepalived.log
tail -f /var/log/haproxy.log
tail -f /var/keepalived/state
tail -f /var/keepalived/vip_check_failed_count
```