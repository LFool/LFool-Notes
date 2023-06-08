# MySQL 搭建主从复制

本篇文章从零开始搭建 MySQL 主从个复制的架构～～

由于搭建过程与 MySQL 版本有很大的关系，所以本文基于 MySQL 8.0.33

**主库 IP：**192.168.1.1

**从库 IP：**192.168.1.2

### <font color=#1FA774>配置主库</font>

下面步骤都在主库中操作～

#### <font color=#9933FF>赋予从库复制权限</font>

```mysql
# 为从库创建一个账号 repl，密码为 123456，从库 IP 没有限制
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY '123456';

# 赋予该账号复制权限
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

# 刷新权限生效
FLUSH PRIVILEGES;
```

#### <font color=#9933FF>修改主库配置文件</font>

在主库的`/etc/my.cnf`文件中添加下面内容：

```properties
[mysqld]
# binlog 日志存储位置
log-bin=/opt/homebrew/var/mysql/binlog
# master id，必须不重复
server-id = 1
# binlog 记录的格式为 row
binlog_format = row
# 同步的数据库
binlog-do-db = study
# 不同步的数据库
binlog-ignore-db = mysql
```

配置好后重启主库 MySQL，执行`SHOW MASTER STATUS;`可以查看主库中 binlog 文件名和位置 Position 的值，这部分数据在配置从库时会用到：

```mysql
mysql> show master status;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| binlog.000014 |     1361 | study        | mysql            |                   |
+---------------+----------+--------------+------------------+-------------------+
1 row in set (0.01 sec)
```

### <font color=#1FA774>配置从库</font>

#### <font color=#9933FF>修改从库配置文件</font>

在从库的`/etc/my.cnf`文件中添加下面内容：

```properties
[mysqld]
# binlog 日志存储位置
log-bin=/var/lib/mysql/binlog
# slave id，必须不重复
server-id = 2
# binlog 记录的格式为 row
binlog_format = row
# 同步的数据库
binlog-do-db = study
# 不同步的数据库
binlog-ignore-db = mysql
# 从库只可以读
read_only = ON
```

配置好后重启从库 MySQL，然后执行下面语句和主库建立连接：

```mysql
# SOURCE_HOST 主库 ip
# SOURCE_USER、SOURCE_PASSWORD 主库授权给从库的账号和密码
# SOURCE_LOG_FILE 主库中 binlog 文件名
# SOURCE_LOG_POS 主库中 binlog 位置的值 Position
CHANGE REPLICATION SOURCE TO SOURCE_HOST='192.168.1.1',SOURCE_PORT=3306,SOURCE_USER='repl',SOURCE_PASSWORD='123456',SOURCE_LOG_FILE='binlog.000014',SOURCE_LOG_POS=1361;
```

#### <font color=#9933FF>控制主从复制相关操作</font>

```mysql
# 启动主从复制
START REPLICA;
# 停止主从复制
STOP REPLICA;
# 重启主从复制工作
RESTART REPLICA;
# 清除从库主从复制关系
RESET REPLICA;
```

除此之外，还可以通过`SHOW REPLICA STATUS\G;`查看从库主从复制的状态：

```mysql
mysql> SHOW REPLICA STATUS\G;
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 192.168.124.13
                  Source_User: rep1
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: binlog.000014
          Read_Source_Log_Pos: 1361
               Relay_Log_File: LFool-relay-bin.000003
                Relay_Log_Pos: 624
        Relay_Source_Log_File: binlog.000014
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Source_Log_Pos: 1361
              Relay_Log_Space: 1602
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Source_SSL_Allowed: No
           Source_SSL_CA_File: 
           Source_SSL_CA_Path: 
              Source_SSL_Cert: 
            Source_SSL_Cipher: 
               Source_SSL_Key: 
        Seconds_Behind_Source: 0
Source_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Source_Server_Id: 2
                  Source_UUID: 93c73bc6-eb9f-11ed-b7cf-52361ce7344f
             Source_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Source_Retry_Count: 86400
                  Source_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Source_SSL_Crl: 
           Source_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Source_TLS_Version: 
       Source_public_key_path: 
        Get_Source_public_key: 0
            Network_Namespace: 
1 row in set (0.00 sec)
```

### <font color=#1FA774>注意事项</font>

从库相应的数据库和表需要手动提前创建好，主从复制不会自动创建数据库和表

主从复制只会自动同步开始复制之后的数据，而开始复制之前的数据需要手动复制到从库

主库和从库的 server-id 必须不重复

### <font color=#1FA774>参考文章</font>

- **[mysql主从同步配置](https://www.cnblogs.com/zhoujie/p/mysql1.html)**
- **[MySQL8.0主从复制的配置](https://blog.csdn.net/u013068184/article/details/107691389)**