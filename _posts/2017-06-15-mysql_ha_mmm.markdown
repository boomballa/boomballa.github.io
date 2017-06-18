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

## MMM高可用架构介绍及配置使用
MMM（Master-Master Replication Manager for MySQL）是一组灵活的脚本，用于执行MySQL Master-Master复制配置的监视/故障转移和管理（只有一个节点可以随时写入）。 该工具集还能够通过标准主/从架构配置任意数量的从库，达到读库负载平衡，因此您可以通过移动虚拟IP地址，来决定它们是否在复制集群中。 除此之外，它还具有数据备份，节点之间的重新同步等脚本。

MMM官网地址：[传送门](http://mysql-mmm.org/doku.php)    
MMM使用手册地址：[传送门](http://mysql-mmm.org/mysql-mmm.html)

## MMM高可用架构搭建

#### **搭建环境介绍**

| IP地址 | 节点角色 | 节点端口 |
| :-------------: |:-------------:| :--------: |
| 10.10.13.20 | master-1 | 3306 |
| 10.10.13.22 | master-2 | 3306 |
| 10.10.13.23 | monitor-slave | 3306 |
	
我这边直接就把`monitor`搭建在`slave`上了，在正式线上环境中，大家可以把`monitor`单独搭建在一台机器上。搭建主从我就不在文章中演示了，相信大家都可以信手拈来了。这里只需要注意一点，搭建主主环境的时候，在`master2`上执行`show master status\G`，随便找一个点change过去即可。

#### MMM monitor与agent的安装

首先安装一下`epel`的yum源，在`monitor-slave`上安装`monitor`和`agent`，在`master1`和`master2`上安装`agent`。

1.`monitor-slave`的安装
```
rpm -ivh http://mirror01.idc.hinet.net/EPEL/6/x86_64/epel-release-6-8.noarch.rpm
yum install mysql-mmm-monitor mysql-mmm-agent -y
```

2.`master1`/`master2`的安装
```
rpm -ivh http://mirror01.idc.hinet.net/EPEL/6/x86_64/epel-release-6-8.noarch.rpm
yum install mysql-mmm-agent -y
```
#### MMM 集群中各个机器上进行赋权

```
GRANT PROCESS, SUPER, REPLICATION CLIENT ON *.* TO 'mmm_agent'@'10.10.13.%' IDENTIFIED BY 'mmm_agent';
GRANT REPLICATION CLIENT ON *.* TO 'mmm_monitor'@'10.10.13.%' IDENTIFIED BY 'mmm_monitor';
```

#### MMM 配置文件修改及配置

在monitor那台机器上执行过以上操作后，开始修改配置文件，MMM的配置文件默认会放置在`/etc/mysql-mmm/`，配置文件包含以下几个：
- mmm_common.conf
- mmm_agent.conf
- mmm_mon.conf
- mmm_mon_log.conf

##### mmm_common.conf
`mmm_common.conf`是`monitor`和`agent`上最主要的配置文件，不管什么角色，都要有这个配置文件，内容如下:

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

配置文件`mmm_common.conf`中各项配置解读：
- active_master_role     在线主机的角色 writer/reader，设置为`writer`，就是为可写的主机，`reader`则会被设置为`read_only=1`    
    
`<host default>`   
- cluster_interface       集群绑定虚拟IP的网卡    
- pid_path                pid文件存放目录    
- bin_path                bin的存放目录    
- replication_user        集群复制账户名称    
- replication_password    集群复制账户密码    
- agent_user              agent用户账户名称    
- agent_password          agent用户账户密码    

`<host role_name>`    
- ip      角色IP地址    
- mode    角色模式 master/slave    
- peer    监视主机的角色，不要指定本机，指定除了本机以外的角色    
    
`<role writer>`    
- hosts   可以成为写入节点的IP地址
- ips     写入节点的虚拟IP，只可在一台master上面出现
- mode    写入角色的运行模式 exclusive(独占模式)/balanced(均衡模式)，写入节点一般设置单点写入exclusive(独占模式)。 

`<role reader>`
- hosts   读节点的所有IP地址
- ips     读角色虚拟IP地址，可以设置多个
- mode    读角色的运行模式 exclusive(独占模式)/balanced(均衡模式)，读节点一般设置轮询balanced(均衡模式)。   

##### mmm_agent.conf
`mmm_agent.conf`这个文件是指定该机器的角色的,`mmm_common.conf`中指定该台机器是什么角色，就在次配置文件配置，该台展示配置文件服务器角色为`db3`,内容如下：

```
[shell ~]# cat /etc/mysql-mmm/mmm_agent.conf 

include mmm_common.conf

# The 'this' variable refers to this server.  Proper operation requires 
# that 'this' server (db1 by default), as well as all other servers, have the 
# proper IP addresses set in mmm_common.conf.
this db3
```

配置文件`mmm_agent.conf`中配置解读，这里面只有两个有效配置：

- include 此项为默认，保持默认即可
- this 此处指定该机器角色即可 


##### mmm_mon.conf
`mmm_mon.conf`这个文件只需要在`monitor`机器上面配置，主要是为了指定`monitor`监控进程的。

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

配置文件`mmm_mon.conf`中各项配置解读：

- include mmm_common.conf，此项保持默认即可

`<monitor>`
- ip                  一般设置127.0.0.1，monitor本机就可以。
- pid_path            monitor进程pid文件位置
- bin_path            monitor进程bin文件位置
- status_path         monitor status文件存放位置
- ping_ips            设置集群中比较重要的数据库服务器节点IP地址即可
- auto_set_online     判断为online状态的轮询时间
- mode                monitor模式，active(自动)/manual(手动)/passive(被动)    

`<host default>`
- monitor_user        monitor用户名称
- monitor_password    monitor用户密码    
    
    
- debug 是否为调试模式，0(否)/1(是)    

##### mmm_mon_log.conf
`mmm_mon_log.conf`是指定`monitor`日志文件的配置文件，一般默认即可，需要个性化设置，可以进行配置。

```
[shell ~]# cat mmm_mon_log.conf 
#log4perl.logger = FATAL, MMMLog, MailFatal
log4perl.logger = FATAL, MMMLog

log4perl.appender.MMMLog = Log::Log4perl::Appender::File
log4perl.appender.MMMLog.Threshold = INFO
log4perl.appender.MMMLog.filename = /var/log/mysql-mmm/mmm_mond.log
log4perl.appender.MMMLog.recreate = 1
log4perl.appender.MMMLog.layout = PatternLayout
log4perl.appender.MMMLog.layout.ConversionPattern = %d %5p %m%n

#log4perl.appender.MailFatal = Log::Dispatch::Email::MailSender
#log4perl.appender.MailFatal.Threshold = FATAL
#log4perl.appender.MailFatal.from = mmm@example.com
#log4perl.appender.MailFatal.to = root
#log4perl.appender.MailFatal.buffered = 0
#log4perl.appender.MailFatal.subject = FATAL error in mysql-mmm-monitor
#log4perl.appender.MailFatal.layout = PatternLayout
#log4perl.appender.MailFatal.layout.ConversionPattern = %d %m%n
```

#### MMM 常用命令的使用和说明

##### mmm_control checks

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

##### mmm_control ping

```
[shell ~]# mmm_control ping
OK: Pinged successfully!
```

##### mmm_control mode

```
[shell ~]# mmm_control mode
ACTIVE
```
##### mmm_control show

```
[shell ~]# mmm_control show
  db1(10.10.13.20) master/ONLINE. Roles: writer(10.10.13.26)
  db2(10.10.13.22) master/ONLINE. Roles: reader(10.10.13.27)
  db3(10.10.13.23) slave/ONLINE. Roles:
```

##### mmm_control help  

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
    

## 结束语
   妥啦，今天MySQL数据库MMM架构，就分享到这。            
   
