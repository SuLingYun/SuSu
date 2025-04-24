## NGINX+Keepalived 高可用配置

### **一、主备服务器完整配置文件**

#### **主服务器（1.10.10.2）配置：`/etc/keepalived/keepalived.conf`**

```nginx
global_defs {
   router_id nginx_master  # 唯一标识符，备机改为nginx_backup
}

# 定义Nginx健康检查脚本（无需外部文件）
vrrp_script chk_nginx {
   script "/usr/bin/killall -0 nginx"  # 检查Nginx进程是否存在（0表示存在）
   interval 2                          # 每2秒检查一次
   weight -20                          # 失败时优先级降低20（确保备机抢占）
   fall 2                              # 连续2次失败视为真
   rise 1                              # 1次成功即恢复
}

vrrp_instance VI_1 {
    state MASTER          # 主节点为MASTER，备机改为BACKUP
    interface ens192      # 绑定物理网卡（ifconfig查看实际名称）
    virtual_router_id 51  # 虚拟路由ID（集群内必须一致，范围0-255）
    priority 100          # 初始优先级（主高于备，例如主100，备90）
    advert_int 1          # VRRP心跳间隔（秒）

    # 单播模式配置（解决多播/组播网络限制问题）
    unicast_src_ip 1.10.10.2  # 本机IP
    unicast_peer {
        1.10.10.3            # 对端IP
    }

    # 认证配置（主备必须完全一致）
    authentication {
        auth_type PASS
        auth_pass Ng1x_HA_PSWD  # 建议更改为高强度密码
    }

    # 虚拟IP（VIP）配置
    virtual_ipaddress {
        1.10.10.5/24 dev ens192  # VIP绑定到ens192网卡
    }

    # 关联健康检查脚本
    track_script {
        chk_nginx  # 引用上方定义的脚本
    }
}
```

#### **备用服务器（1.10.10.3）配置：`/etc/keepalived/keepalived.conf`**

```nginx
global_defs {
   router_id nginx_backup  # 修改为备机标识
}

vrrp_script chk_nginx {    # 与主节点完全一致
   script "/usr/bin/killall -0 nginx"
   interval 2
   weight -20
   fall 2
   rise 1
}

vrrp_instance VI_1 {
    state BACKUP           # 改为BACKUP
    interface ens192
    virtual_router_id 51   # 必须与主节点一致
    priority 90            # 优先级低于主节点
    advert_int 1

    unicast_src_ip 1.10.10.3  # 备机IP
    unicast_peer {
        1.10.10.2            # 主节点IP
    }

    authentication {
        auth_type PASS
        auth_pass Ng1x_HA_PSWD  # 密码与主节点一致
    }

    virtual_ipaddress {
        1.10.10.5/24 dev ens192
    }

    track_script {
        chk_nginx
    }
}
```

---

### **二、详细部署步骤**

#### **1. 安装Keepalived和依赖**

```sh
# CentOS/RedHat
yum install -y keepalived psmisc  # psmisc提供killall命令
# Ubuntu/Debian
apt-get update && apt-get install -y keepalived psmisc
```

#### **2. 创建配置文件**

- 主备服务器分别编辑配置文件：
  ```sh
  vi /etc/keepalived/keepalived.conf
  ```
- **直接粘贴上方对应配置**，注意修改`router_id`、`state`、`priority`、`unicast_src_ip`

#### **3. 配置防火墙（关键！）**

```sh
# 放行VRRP协议（IP协议号112）
# 若使用firewalld（CentOS 7+）：
firewall-cmd --permanent --add-rich-rule='rule protocol value="vrrp" accept'
firewall-cmd --reload

# 若使用iptables（Ubuntu/CentOS 6）：
iptables -A INPUT -p vrrp -j ACCEPT
service iptables save
service iptables restart
```

#### **4. 启动服务并设置开机自启**

```sh
systemctl start keepalived
systemctl enable keepalived
```

#### **5. 验证VIP绑定**

```sh
ip addr show ens192 | grep 'inet'  # 查看ens192网卡是否出现1.10.10.5
```

#### **6. 模拟故障测试**

- **主节点操作**：
  ```sh
  systemctl stop keepalived  # 停止主节点服务
  ```
- **备节点检查**：
  ```sh
  ip addr show ens192        # 应看到VIP 1.10.10.5
  ```

---

### **三、关键原理说明**

1. **健康检查机制**：
   - `killall -0 nginx` 仅检查进程是否存在，如需更严格检测（如端口响应），可替换脚本：
     ```sh
     script "/usr/bin/curl -s http://localhost >/dev/null"  # 检测HTTP服务
     ```
2. **优先级权重**：
   - 主节点初始优先级为100，备节点为90。当主节点Nginx宕机时，权重-20使主优先级降至80，备节点90成为新主。
3. **日志追踪**：
   ```sh
   tail -f /var/log/messages  # CentOS
   tail -f /var/log/syslog    # Ubuntu
   ```

---

### **四、常见问题排查**

- **VIP未切换**：
  1. 检查防火墙是否放行VRRP协议。
  2. 确保主备`virtual_router_id`和`auth_pass`一致。
  3. 查看日志`journalctl -u keepalived`。
- **killall命令无效**：
  ```sh
  yum install psmisc  # CentOS
  apt-get install psmisc  # Ubuntu
  ```

---

### **五、最终架构效果**

```
客户端访问VIP 1.10.10.5
       |
       v
+-------------+     +-------------+
|  Master     |     |  Backup     |
| 1.10.10.2   |<--->| 1.10.10.3   |
| (优先级100) |     | (优先级90)  |
+-------------+     +-------------+
正常时VIP绑定在Master，当Master宕机后VIP自动漂移到Backup
```

按此配置部署后，您的Nginx服务将获得秒级故障切换能力，确保业务高可用。
