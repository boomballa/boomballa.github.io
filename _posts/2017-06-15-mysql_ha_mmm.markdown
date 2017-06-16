---
layout:     post
title:      "MySQL MMM高可用架构的介绍与搭建"
subtitle:   "Introduction And Install of MySQL MMM High Availability Architecture"
date:       2017-06-16 00:47:03
author:     "Boomballa"
header-img: "img/home-bg.jpg"
catalog:    true
tags:
    - MySQL
    - High Availability
    - Perl
---
## 开场白
好久不见，最近又不知不觉的忙碌了起来，许多新项目的数据库已经投入使用了一段时间，需要整理，各种数据归档，更改数据流存储不合理的地方。接下来两篇文章来介绍下[MySQL](https://www.mysql.com/)的两种高可用方案，此篇文章来介绍我目前环境中运用最多的[MMM](http://mysql-mmm.org/doku.php)架构。

## MMM高可用架构介绍
MMM（Master-Master Replication Manager for MySQL）是一组灵活的脚本，用于执行MySQL Master-Master复制配置的监视/故障转移和管理（只有一个节点可以随时写入）。 该工具集还能够通过标准主/从架构配置任意数量的从库，达到读库负载平衡，因此您可以通过移动虚拟IP地址，来决定它们是否在复制集群中。 除此之外，它还具有数据备份，节点之间的重新同步等脚本。

MMM官网地址：[传送门](http://mysql-mmm.org/doku.php)
MMM使用手册地址：[传送门](http://mysql-mmm.org/mysql-mmm.html)

<img src="/img/in-post/mysql_ha_mmm/mysql_mmm.jpg" height="479" width="638">

## MMM高可用架构搭建

#### **搭建环境介绍**

| IP地址 | 节点角色 | 节点端口 |
| :-------------: |:-------------:| :--------: |
| 10.10.13.20 | master-1 | 3306 |
| 10.10.13.22 | master-2 | 3306 |
| 10.10.13.23 | monitor-slave | 3306 |
	
我这边直接就把`monitor`搭建在`slave`上了，在正式线上环境中，大家可以把`monitor`单独搭建在一台机器上。搭建主从我就不在文章中演示了，相信大家都可以信手拈来了。这里只需要注意一点，搭建主主环境的时候，在`master2`上执行`show master status\G`，随便找一个点change过去即可。

#### MMM高可用环境搭建


```
[shell ~]# cat /etc/mysql-mmm/mmm_common.conf 

active_master_role      writer

<host default>
    cluster_interface       bond0
    pid_path                /var/run/mysql-mmm/mmm_agentd.pid
    bin_path                /usr/libexec/mysql-mmm/
    replication_user        repl
    replication_password    repl
    agent_user              mmm_agent
    agent_password          mmm_agent
</host>

<host db1>
    ip      10.10.13.20
    mode    master
    peer    db2
</host>

<host db2>
    ip      10.10.13.22
    mode    master
    peer    db1
</host>

<host db3>
    ip      10.10.13.23
    mode    slave
</host>

<role writer>
    hosts   db1, db2
    ips     10.10.13.26
    mode    exclusive
</role>

<role reader>
    hosts   db2, db3
    ips     10.10.13.27
    mode    balanced
</role>
```

```
[shell ~]# cat /etc/mysql-mmm/mmm_agent.conf 

include mmm_common.conf

# The 'this' variable refers to this server.  Proper operation requires 
# that 'this' server (db1 by default), as well as all other servers, have the 
# proper IP addresses set in mmm_common.conf.
this db3
```

```
[shell ~]# cat /etc/mysql-mmm/mmm_mon.conf 

include mmm_common.conf

<monitor>
    ip                  127.0.0.1
    pid_path            /var/run/mysql-mmm/mmm_mond.pid
    bin_path            /usr/libexec/mysql-mmm
    status_path         /var/lib/mysql-mmm/mmm_mond.status
    ping_ips            10.10.13.20,10.10.13.22,10.10.13.23
    auto_set_online     60
    mode                active

    # The kill_host_bin does not exist by default, though the monitor will
    # throw a warning about it missing.  See the section 5.10 "Kill Host 
    # Functionality" in the PDF documentation.
    #
    # kill_host_bin     /usr/libexec/mysql-mmm/monitor/kill_host
    #
</monitor>

<host default>
    monitor_user        mmm_monitor
    monitor_password    mmm_monitor
</host>

debug 0
```

```
GRANT PROCESS, SUPER, REPLICATION CLIENT ON *.* TO 'mmm_agent'@'10.10.13.%' IDENTIFIED BY 'mmm_agent';
GRANT REPLICATION CLIENT ON *.* TO 'mmm_monitor'@'10.10.13.%' IDENTIFIED BY 'mmm_monitor';
```

```
[shell ~]# mmm_control checks
db2  ping         [last change: 2017/06/12 12:13:34]  OK
db2  mysql        [last change: 2017/06/12 12:13:34]  OK
db2  rep_threads  [last change: 2017/06/12 12:13:34]  OK
db2  rep_backlog  [last change: 2017/06/12 12:13:34]  OK: Backlog is null
db3  ping         [last change: 2017/06/12 12:13:34]  OK
db3  mysql        [last change: 2017/06/12 12:13:34]  OK
db3  rep_threads  [last change: 2017/06/12 12:13:34]  OK
db3  rep_backlog  [last change: 2017/06/12 12:13:34]  OK: Backlog is null
db1  ping         [last change: 2017/06/12 12:13:34]  OK
db1  mysql        [last change: 2017/06/12 12:13:34]  OK
db1  rep_threads  [last change: 2017/06/12 12:13:34]  OK
db1  rep_backlog  [last change: 2017/06/12 12:13:34]  OK: Backlog is null
```

```
[shell ~]# mmm_control ping
OK: Pinged successfully!
```
```
[shell ~]# mmm_control mode
ACTIVE
```

```
[shell ~]# mmm_control show
  db1(10.10.13.20) master/ONLINE. Roles: writer(10.10.13.26)
  db2(10.10.13.22) master/ONLINE. Roles: reader(10.10.13.27)
  db3(10.10.13.23) slave/ONLINE. Roles:
```

```
[shell ~]# mmm_control help  
Valid commands are:
    help                              - show this message
    ping                              - ping monitor
    show                              - show status
    checks [<host>|all [<check>|all]] - show checks status
    set_online <host>                 - set host <host> online
    set_offline <host>                - set host <host> offline
    mode                              - print current mode.
    set_active                        - switch into active mode.
    set_manual                        - switch into manual mode.
    set_passive                       - switch into passive mode.
    move_role [--force] <role> <host> - move exclusive role <role> to host <host>
                                        (Only use --force if you know what you are doing!)
    set_ip <ip> <host>                - set role with ip <ip> to host <host>
```





### 结束语
   妥啦，今天历史数据归档的话题，就分享到这，如果哪点分享的有问题或者有更好的方式解决此问题，欢迎雅正和探讨。            
   
