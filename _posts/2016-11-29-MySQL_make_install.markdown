---
layout:     post
title:      "编译安装MySQL 5.6"
subtitle:   "Hello World, Hello Blog"
date:       2016-11-29 01:27:00
author:     "Boomballa"
header-img: "img/home-bg.jpg"
catalog:    true
tags:
    - MySQL
---


  第一次到自己的地盘写东西，感觉还真的有些紧张呢，进来主页空落落的感觉有些违和，之前在简介中也说过要把自己总结过的东西分享出来，虽说不一定很干。但是基本上都是自己在这条路上一步一步走过来的脚印，丰富Blog的同时，也算是对自己日常技术生活的归纳总结吧。 

  值得一说的是，我这个Blog是使用 Jekyll 搭建的，搭建过程参考了 @[潘伯信](http://baixin.io)在简书上的一篇文章 [Jekyll搭建个人博客](http://baixin.io/2016/10/jekyll_tutorials1/)，讲解的很详细，最后就是统计网站访问统计有点小问题，通过研究已经解决了，当然还有@[老司机](http://johnscott1989.cc/)的帮助，没有错，他的blog是HEXO搭建的。现在柏信的 [github](https://github.com/) 代码应该已经更新了，所以小伙伴们有兴趣的都可以尝试一下。


### 简介
  
  做为一名MySQL DBA，容我在开讲之前安利一下，简单的介绍一下MySQL。
  
  [MySQL](http://baike.baidu.com/subview/24816/15308361.htm)是一种[开放源代码](http://baike.baidu.com/view/1708.htm)的关系型[数据库管理](http://baike.baidu.com/view/600155.htm)系统（RDBMS），MySQL[数据库系统](http://baike.baidu.com/view/7809.htm)使用最常用的数据库管理语言--[结构化查询语言](http://baike.baidu.com/view/595350.htm)（SQL）进行数据库管理。
   
   好了，闲话不说了，下面入题。

### 编译安装MySQL  
1. 下载and解压缩
  本文以percona-server-5.6.29版本源码包为例。下载源码包我不赘述了，[mysql](http://www.mysql.com/)或者[percona](https://www.percona.com/)去下载。

```
$ tar -zxvf percona-server-5.6.29-76.2.tar.gz -C /path-you-want/
```

2. 安装必要软件包

```
$ yum -y install  gcc gcc-c++ gcc-g77 autoconf automake zlib* fiex* libxml* libmcrypt* libtool-ltdl-devel* make cmake readline-devel ncurses ncurses-devel openssl openssl-devel
```

3. 进入主目录 进行编译

```
$ cmake .
-DCMAKE_INSTALL_PREFIX=/opt/app/mysql \
-DMYSQL_DATADIR=/opt/app/mysql/data \
-DSYSCONFDIR=/etc \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_MEMORY_STORAGE_ENGINE=1 \
-DMYSQL_UNIX_ADDR=/opt/app/mysql/tmp/mysql.sock \
-DMYSQL_TCP_PORT=3306 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci
```

如果编译出错，需要重新编译，可以尝试

```
$ rm CMakeCache.txt
```

4. 上述步骤通过之后，进行 make 。

```
$ make&&make install
```

OK，这里执行完毕，就已经离成功很近了，这时候我们仅仅需要初始化数据库，就可以使用MySQL数据库了，这里有两种情况

1. 如果源码包是MySQL官方版本，可以直接进行初始化。
2. 如果Percona版本的源码包则需要额外安装一下东西。

```
$ yum install libaio numactl -y
```

5. 初始化

```
$ cd /opt/app/mysql

$ ./scripts/mysql_install_db --user=mysql --defaults-file=/opt/app/mysql/my.cnf --basedir=/opt/app/mysql --datadir=/opt/app/mysql/data --explicit_defaults_for_timestamp

$ mkdir -p {data,logs/tmp} && chown -R mysql.mysql {data,logs,tmp}
```

### 启动数据库
```
$ /etc/init.d/mysql start
```

至此，MySQL编译安装完成，有问题欢迎一起探讨。
