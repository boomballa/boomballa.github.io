---
layout: post
title: "MySQL Group Replication的安装部署"
date: 2017-01-04
tags: MySQL
---

### 开场白
  这次给大家介绍下MySQL官方最新版本5.7.17中GA的新功能  **Group Replication** 。

**简介**

> Group Replication是一种可用于实现容错系统的技术。复制组是一组通过消息传递相互交互的服务器。通信层提供一组保证，例如原子消息和总订单消息传递。这些是非常强大的属性，可以转化为非常有用的抽象，人们可以诉诸构建更高级的数据库复制解决方案。MySQL组复制构建在这些属性和抽象之上，并实现多主复制协议的更新。实质上，复制组由多个服务器形成，并且组中的每个服务器可以独立地执行事务。但是所有读写（RW）事务只有在组被批准后才会提交。只读（RO）事务不需要在组内协调，因此立即提交。换句话说，对于任何RW事务，组需要决定是否提交，因此提交操作不是来自始发服务器的单向决定。准确地说，当事务准备好在始发服务器上提交时，服务器原子地广播写入值（已改变的行）和对应的写入集（已更新的行的唯一标识符）。然后为该交易建立全局总订单。最终，这意味着所有服务器以相同的顺序接收同一组事务。因此，所有服务器以相同的顺序应用相同的一组更改，因此它们在组内保持一致。

>  但是，在不同服务器上并发执行的事务之间可能存在冲突。通过在称为认证的过程中检查两个不同的并发事务的写集合来检测这样的冲突。如果在不同的服务器上执行的两个并发事务更新同一行，则会出现冲突。解析过程指出，首先订购的事务在所有服务器上提交，而顺序第二次中止的事务将在源服务器上回滚，并由组中的其他服务器删除。这实际上是一个分布式的第一个提交赢的规则。

 **MySQL组复制协议**
![](https://dev.mysql.com/doc/refman/5.7/en/images/gr-replication-diagram.png)

> 最后，组复制是一种无共享复制方案，其中每个服务器都有自己的整个数据副本。
上图描述了MySQL组复制协议，并通过将其与MySQL复制（或甚至MySQL半同步复制）进行比较，您可以看到一些差异。注意，为了清楚起见，这个图片中缺少一些基本的共识和Paxos相关消息。

介绍就到这，本文中我将一步一步的安装部署group_replication的三个节点，并让你看到它的功能和特性，如果看完全文，你十分的感兴趣的话，可以去[mysql](https://dev.mysql.com)的[Group Replication](https://dev.mysql.com/doc/refman/5.7/en/group-replication.html)主页去查看更详细的信息。

### 正文

**1. 环境介绍**

basedir = /usr/local/mysql 

(PS: 这点还是忍不住要吐槽一下，mysql官方basedir如果不在这个目录的话，mysql.server也不好使，这个实验也不成功，我们还是勉强先放在这)

| 端口号 | 数据及日志目录 | Group_replication 通讯端口 |
| :-------------: |:-------------:| :--------: |
| 3306 | /data/mysql/mysql_3306/{data,logs,tmp} | 33061 |
| 3307 | /data/mysql/mysql_3307/{data,logs,tmp} | 33071 |
| 3308 | /data/mysql/mysql_3308/{data,logs,tmp} | 33081 |

**2. 初始化**

①、下载and解压缩，并把mysql放到指定地方（标准目录： /usr/local/mysql）

```
shell> cd /opt/soft
shell> wget http://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz 
shell> tar -zxvf mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz -C /opt/app
shell> mv mysql-5.7.17-linux-glibc2.5-x86_64 mysql
shell> ln -s /opt/app/mysql /usr/local/mysql
```

②、 创建数据库需要的数据、日志和临时目录并赋权：
 
```
shell> mkdir -p /data/mysql/{mysql_3306,mysql_3307,mysql_3308}/{data,logs,tmp}
shell> chown -R mysql.mysql /data/mysql/
```

**3. 正式初始化并安装**

安装机器： 172.16.3.134 （单机多实例安装）
配置文件说明：

| 端口号 | 配置文件 |
| :-------------: | :-------------: |
| 3306 | /data/mysql/mysql_3306/my3306.cnf|
| 3307 | /data/mysql/mysql_3306/my3307.cnf|
| 3308 | /data/mysql/mysql_3306/my3308.cnf|

3306端口配置文件详情：

```
shell> cat /data/mysql/mysql_3306/my3306.cnf 

[client]
port            = 3306
socket          = /tmp/mysql3306.sock

[mysql]
prompt                         = mysql [\d]>
default_character_set          = utf8
no-auto-rehash

[mysqld]
#misc
user = mysql
basedir = /usr/local/mysql
datadir = /data/mysql/mysql_3306/data
port = 3306
socket = /tmp/mysql3306.sock
event_scheduler = 0

tmpdir=/data/mysql/mysql_3306/tmp
#timeout
interactive_timeout = 43200
wait_timeout = 43200

#character set
character-set-server = utf8

open_files_limit = 65535
max_connections = 100
max_connect_errors = 100000
#
explicit_defaults_for_timestamp
#logs
log-output=file
slow_query_log = 1
slow_query_log_file = slow.log
log-error = error.log
log_error_verbosity=3
pid-file = mysql.pid
long_query_time = 1
#log-slow-admin-statements = 1
#log-queries-not-using-indexes = 1
log-slow-slave-statements = 1

#binlog
binlog_format = row
server-id = 1343306
log-bin = /data/mysql/mysql_3306/logs/mysql-bin
binlog_cache_size = 1M
max_binlog_size = 200M
max_binlog_cache_size = 2G
sync_binlog = 0
expire_logs_days = 10


#group replication
server_id=1013306
gtid_mode=ON
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE
log_slave_updates=ON
binlog_format=ROW

transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="13855fca-d2ab-11e6-8f37-005056b8286c"
loose-group_replication_start_on_boot=off
loose-group_replication_local_address= "172.16.3.134:23306"
loose-group_replication_group_seeds= "172.16.3.134:33061,172.16.3.134:33071,172.16.3.134:33081"
loose-group_replication_bootstrap_group= off

loose-group_replication_single_primary_mode=off
loose-group_replication_enforce_update_everywhere_checks=on

#relay log
skip_slave_start = 1
max_relay_log_size = 500M
relay_log_purge = 1
relay_log_recovery = 1
#slave-skip-errors=1032,1053,1062

#buffers & cache
table_open_cache = 2048
table_definition_cache = 2048
table_open_cache = 2048
max_heap_table_size = 96M
sort_buffer_size = 2M
join_buffer_size = 2M
thread_cache_size = 256
query_cache_size = 0
query_cache_type = 0
query_cache_limit = 256K
query_cache_min_res_unit = 512
thread_stack = 192K
tmp_table_size = 96M
key_buffer_size = 8M
read_buffer_size = 2M
read_rnd_buffer_size = 16M
bulk_insert_buffer_size = 32M

#myisam
myisam_sort_buffer_size = 128M
myisam_max_sort_file_size = 10G
myisam_repair_threads = 1

#innodb
innodb_buffer_pool_size = 500M
innodb_buffer_pool_instances = 1
innodb_data_file_path = ibdata1:100M:autoextend
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 64M
innodb_log_file_size = 256M
innodb_log_files_in_group = 3
innodb_max_dirty_pages_pct = 90
innodb_file_per_table = 1
innodb_rollback_on_timeout
innodb_status_file = 1
innodb_io_capacity = 2000
transaction_isolation = READ-COMMITTED
innodb_flush_method = O_DIRECT

```

**3307**和**3308**端口的配置文件我就不贴了，替换下端口即可。

其中比较重要的地方，也是安装**Group_replication**必须要有的配置项，需要注意一下：

```
#group replication
server_id=1013306
gtid_mode=ON
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE
log_slave_updates=ON
binlog_format=ROW

transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="13855fca-d2ab-11e6-8f37-005056b8286c"
loose-group_replication_start_on_boot=off
loose-group_replication_local_address= "172.16.3.134:23306"
loose-group_replication_group_seeds= "172.16.3.134:33061,172.16.3.134:33071,172.16.3.134:33081"
loose-group_replication_bootstrap_group= off

loose-group_replication_single_primary_mode=off
loose-group_replication_enforce_update_everywhere_checks=on
```

这离说一下参数中带有loose-xxx-xxx ， 这个我也是头一次见，我们来看一下它的解释：


> 注意

> 用于上面group_replication变量的松散前缀指示服务器继续启动，如果在服务器启动时尚未加载组复制插件。




**4.启动Group_replication第一个节点：** 

①、初始化3306实例：

```
shell>  /usr/local/mysql/bin/mysqld --defaults-file=/data/mysql/mysql_3306/my3306.cnf --initialize-insecure
```

②、启动3306实例：

```
shell> /usr/local/mysql/bin/mysqld --defaults-file=/data/mysql/mysql_3306/my3306.cnf &
```
 
③、进入mysql进行change操作：

```
mysql> SET SQL_LOG_BIN=0;
mysql> CREATE USER rpl_user@'%';
mysql> GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%' IDENTIFIED BY 'rpl_pass';
mysql> SET SQL_LOG_BIN=1;
mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_pass' FOR CHANNEL 'group_replication_recovery'; 
```

> 注意

> 这里 SET SQL_LOG_BIN=0; 禁用二进制日志记录，创建具有正确权限的用户，并保存group_replication_recovery通道的凭据。

> 上面看到的最后一行配置此服务器使用给定凭据下次需要从另一个成员恢复其状态。分布式恢复是加入组的服务器执行的第一步。如果未正确设置这些凭据，则服务器无法运行恢复协议，并且最终无法加入组。


④、加载 group_replication的plugin：

```
mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so'; 
mysql> SHOW PLUGINS;
+----------------------------+----------+--------------------+----------------------+---------+
| Name                       | Status   | Type               | Library              | License |
+----------------------------+----------+--------------------+----------------------+---------+
| binlog                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| mysql_native_password      | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
|...                         | ...      | ...                |...                   | ...     |
| group_replication          | ACTIVE   | GROUP REPLICATION  | group_replication.so | GPL     |
+----------------------------+----------+--------------------+----------------------+---------+
```

⑤、启动第一个节点的Group_replication：

```
mysql> SET GLOBAL group_replication_bootstrap_group=ON;     #只在第一个节点使用
mysql> START GROUP_REPLICATION;
```

⑥、确认节点加入情况：

```
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST           | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
| group_replication_applier | 68ce93ca-d292-11e6-bdf9-005056b8286c | localhost.localdomain |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
1 rows in set (0.00 sec)
```

⑦、创建测试数据：

```
mysql> create database boom;
mysql> use boom;
mysql> create table boomballa(id int not null,name varchar(32),primary key(id));
mysql> insert into boomballa(id,name) values(1,'boomballa.top');
mysql> insert into boomballa(id,name) values(2,'myblog');  
```

**5. 启动Group_replication第二个节点：**

①、初始化并启动实例：

```
shell> /usr/local/mysql/bin/mysqld --defaults-file=/data/mysql/mysql_3307/my3307.cnf --initialize-insecure
shell> /usr/local/mysql/bin/mysqld --defaults-file=/data/mysql/mysql_3307/my3307.cnf &
```

②、安装插件并启动Group_replication:

```
mysql> SET SQL_LOG_BIN=0;
mysql> CREATE USER rpl_user@'%';
mysql> GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%' IDENTIFIED BY 'rpl_pass';
mysql> SET SQL_LOG_BIN=1;
mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_pass' FORCHANNEL 'group_replication_recovery';
mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';start group_replication;
mysql> START GROUP_REPLICATION;
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST           | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
| group_replication_applier | 5aaa8529-d296-11e6-a7be-005056b8286c | localhost.localdomain |        3307 | ONLINE       |
| group_replication_applier | 68ce93ca-d292-11e6-bdf9-005056b8286c | localhost.localdomain |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
2 rows in set (0.00 sec)
```

**6. 第三节点安装配置略过了，安装好了以后的状态应该是(三节点上查询结果都是如此)：**

```
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST           | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
| group_replication_applier | 5aaa8529-d296-11e6-a7be-005056b8286c | localhost.localdomain |        3307 | ONLINE       |
| group_replication_applier | 68ce93ca-d292-11e6-bdf9-005056b8286c | localhost.localdomain |        3306 | ONLINE       |
| group_replication_applier | af93afd1-d297-11e6-b8e9-005056b8286c | localhost.localdomain |        3308 | ONLINE       |
+---------------------------+--------------------------------------+-----------------------+-------------+--------------+
3 rows in set (0.00 sec)

mysql> use boom
Database changed
mysql> select * from boomballa;
+----+---------------+
| id | name          |
+----+---------------+
|  1 | boomballa.top |
|  2 | myblog        |
+----+---------------+
2 rows in set (0.00 sec)
```

确认一下：

```
[root@localhost ~]# echo "select * from boom.boomballa;"|/usr/local/mysql/bin/mysql -S /tmp/mysql3306.sock 
id      name
1       boomballa.top
2       myblog
[root@localhost ~]# echo "select * from boom.boomballa;"|/usr/local/mysql/bin/mysql -S /tmp/mysql3307.sock
id      name
1       boomballa.top
2       myblog
[root@localhost ~]# echo "select * from boom.boomballa;"|/usr/local/mysql/bin/mysql -S /tmp/mysql3308.sock
id      name
1       boomballa.top
2       myblog
```

完美 ~

<img src="/images/posts/mysql_gr_install/prefect.jpg" height="296" width="300">

### 结束语
  好啦，安装部署的部分就讲到这里了，博主这几天将继续对Group_replication进行研究，近期还会有关于此功能的文章出现，好啦，下班回家吃饭。