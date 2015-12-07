---
layout: post
title: 试用kingshard
date: 2015-12-07 23:05:20
tags: engineering
---

最近在做一个信息查询的需求，由于没有现成的MySQL集群可以用，所以看了一些关于部署的资料。

## 1

需求：

1. 数据条数上亿，但是结构简单
2. 主要是读操作，偶尔批量更新
3. 需要可以横向扩展，短时间的数据不一致可以接受

实际上需要的就是两个特性——散表和读写分离。数据库使用简单的MySQL足矣，PostgreSQL和Cassandra这种“牛刀”不考虑。

读写分离，使用MySQL内置的Master-Slave特性，前面加一个Proxy即可，有些同步延迟可以接受。

散表，逻辑上很简单，写到业务代码里过于耦合，在此选择一个三方组件处理散表逻辑，同时做为读写分离的Proxy。这类组件中比较知名的是360的[Atlas](https://github.com/Qihoo360/Atlas)，但是在此试用了一个不知名的组件——[kingshard](https://github.com/flike/kingshard)，原因是这个实现比较简单，有问题可以自行修改。

## 2

使用mysqld_multi在一台机器上部署一个Master实例和一个Slave实例。

(部署起来真麻烦…… 感谢之前遇到的DBA们~)

初始化数据：

```plain
mysql_install_db --datadir=./mysql1
mysql_install_db --datadir=./mysql2
```

配置文件：

```plain
mysqld_multi --example > mysqld_multi.conf
```

```plain
[mysqld_multi]
mysqld     = /usr/bin/mysqld_safe
mysqladmin = /usr/bin/mysqladmin
log        = /home/xxx/mysql/mysqld_multi.log
user       = multi_admin
password   = multi_admin

[mysqld1]
socket     = /home/xxx/mysql/mysql.sock1
port       = 3307
pid-file   = /home/xxx/mysql/mysql1/hostname.pid1
datadir    = /home/xxx/mysql/mysql1
log-error  = /home/xxx/mysql/mysql1/mysql.err1
server-id  = 1
log-bin    = /home/xxx/mysql/mysql1/binlog
max_binlog_size = 500M
binlog_cache_size = 128K
binlog-do-db = xxx
binlog-ignore-db = mysql
log-slave-updates
binlog_format="MIXED"
auto-increment-increment=2
auto-increment-offset=1

[mysqld2]
socket     = /home/xxx/mysql/mysql.sock2
port       = 3308
pid-file   = /home/xxx/mysql/mysql2/hostname.pid2
datadir    = /home/xxx/mysql/mysql2
log-error  = /home/xxx/mysql/mysql1/mysql.err1
server-id  = 2
slave-skip-errors=1007,1008,1053,1062,1213,1158,1159
auto-increment-increment=2
auto-increment-offset=2
```

启动MySQL实例：

```plain
mysqld_multi --defaults-extra-file=./mysqld_multi.conf start

mysqld_multi --defaults-extra-file=./mysqld_multi.conf report
Reporting MySQL servers
MySQL server from group: mysqld1 is running
MySQL server from group: mysqld2 is running
```

设置密码和权限：

```plain
mysqladmin -u root -S ./mysql.sock1 password "root"
mysqladmin -u root -S ./mysql.sock2 password "root"

#master
DELETE FROM user WHERE User='';
GRANT SHUTDOWN ON *.* TO 'multi_admin'@'localhost' IDENTIFIED BY 'multi_admin';
GRANT SUPER, REPLICATION SLAVE ON *.* to 'repl'@'%' IDENTIFIED BY 'repl';

#slave
DELETE FROM user WHERE User='';
GRANT SHUTDOWN ON *.* TO 'multi_admin'@'localhost' IDENTIFIED BY 'multi_admin';

```

设置主从：

```plain
#master
show master status;
+---------------+----------+--------------+------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+---------------+----------+--------------+------------------+
| binlog.000001 |      758 | xxx          | mysql            |
+---------------+----------+--------------+------------------+
1 row in set (0.00 sec)

#slave
change master to master_host='127.0.0.1',master_port=3307,master_user='repl',master_password='repl',master_log_file='binlog.000001',master_log_pos=758;

start slave;

show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 127.0.0.1
                  Master_User: repl
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: binlog.000001
          Read_Master_Log_Pos: 758
               Relay_Log_File: hostname-relay-bin.000001
                Relay_Log_Pos: 526
        Relay_Master_Log_File: binlog.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 245
              Relay_Log_Space: 1104
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1

```
注意，**Slave\_IO\_Running**和**Slave\_SQL\_Running**都为**Yes**表示启动成功，然后就可以在master上建库建表了。

至于Master的高可用[MMM](http://mysql-mmm.org/)之类的在此就不折腾了……

## 3

然后部署[kingshard](https://github.com/flike/kingshard)

官方的文档还是挺清楚明了的——[kingshard简介](https://github.com/flike/kingshard/blob/master/README_ZH.md)，在此就不复述了，不过有一点需要注意：

```plain
schema :
    db : xxx
    nodes: [node1,]
    default: node1
    shard:
    -
        table: test_shard_hash
        key: key
        nodes: [node1,]
        type: hash
        locations: [4,]
```

虽然对于散表的key的数据类型没有什么限制，但是当执行INSERT语句时，散表的key必须放在第一个位置。

## 4

当然对于没有经过历史考验的程序，还是挺难让人放心的，所以最后结合使用[Supervisor](http://supervisord.org/)。

```plain
#/usr/local/etc/supervisor/conf.d/kingshard.conf

[program:kingshard]
command=/usr/local/bin/kingshard -config /usr/local/etc/kingshard/multi.yaml
autostart=true
autorestart=true
startretries=10
user=xxx
redirect_stderr=true
stdout_logfile=/usr/local/var/log/supervisor/kingshard.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
```

