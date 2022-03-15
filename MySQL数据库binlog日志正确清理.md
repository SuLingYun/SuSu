## MySQL数据库binlog日志正确清理

## 1、自动删除

永久生效：修改mysql的配置文件my.cnf，添加binlog过期时间的配置项：expire_logs_days=30，然后重启mysql，这个有个致命的缺点就是需要重启mysql。

```sh
expire_logs_days=30
```

临时生效：进入mysql，用以下命令设置全局的参数：set global expire_logs_days=30;

```sh
mysql> show variables like 'expire_logs_days';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| expire_logs_days | 0     |
+------------------+-------+
1 row in set (0.00 sec)
```

```sh
mysql> set global expire_logs_days = 30;
Query OK, 0 rows affected (0.00 sec)
```

```sh
mysql> show variables like 'expire_logs_days';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| expire_logs_days | 30    |
+------------------+-------+
1 row in set (0.00 sec)
```

## 2、手动删除

可以直接删除binlog文件，但是可以通过mysql提供的工具来删除更安全，因为purge会更新mysql-bin.index中的条目，而直接删除的话，mysql-bin.index文件不会更新。mysql-bin.index的作用是加快查找binlog文件的速度。

- **Linux找到MySQL的binlog目录直接rm -rf删除**

- **通过mysql提供的工具来删除**
  ```sh
  purge master logs before '2022-01-01 00:00:00';  //删除2022-01-01 00:00:00之前产生的所有日志
  purge master logs to 'mysql-bin.000002';   //删除mysql-bin.000002之前所有日志
  ```

## 3、其它命令

## 查看是否开启binlog

```sh
mysql> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.01 sec)
```

如果binlog没有开启，可以通过set sql_log_bin=1命令来启用;如果想停用binlog,可以使用set sql_log_bin=0。

## 获取binlog文件列表

```sh
mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000009 | 584660236 |
+------------------+-----------+
1 row in set (0.00 sec)
```

## 查看master上的binlog

```sh
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000009 | 584660236 |
+------------------+-----------+
1 row in set (0.00 sec)
```

## 4、binlog的扩展

当停止或重启服务器时，服务器会把日志文件记入下一个日志文件，Mysql会在重启时生成一个新的日志文件，文件序号递增；此外，如果日志文件超过max_binlog_size（默认值1G）系统变量配置的上限时，也会生成新的日志文件（在这里需要注意的是，如果你正使用大的事务，二进制日志还会超过max_binlog_size，不会生成新的日志文件，事务全写入一个二进制日志中,这种情况主要是为了保证事务的完整性）；日志被刷新时，新生成一个日志文件。

## 刷新binlog日志，重新生成一个日志文件

```sh
mysql> flush logs;
```