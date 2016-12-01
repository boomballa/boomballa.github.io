---
layout: post
title: "Shell脚本加密"
date: 2016-12-02
tags: linux,shell 
---

### 开场白

 好吧，如果觉得前几篇都是MySQL有点枯燥的话，这把我们来点轻松的，摇身一变为猥琐小王子（手动猥琐脸），让我来介绍一个**shell**脚本加密的小**Tips**。 

 <img src="/images/posts/shell/shell.jpg" height="375" width="500">


### 应用场景介绍

  有些时候，并不是我们猥琐，而是因为某些脚本必须要加密，例如我大MySQL数据库登录、备份脚本，里面有时候会配置一些数据库帐号，而之所以配制成脚本，是因为有些生产环境的密码是随机生成的，并背不下来，所以既想方便，又要担心安全问题。今天我们就来讲一个方法，解决你的困扰。  
   首先我们要把思路理清楚，脚本仅仅是加密OK不？  答案是否定的，因为脚本仅仅加密，只是看不到shell的明文，但是脚本依然可以被执行，对不对？   
   所以我们需要两个步骤：

- **shell 脚本加入交互**
- **shell 脚本加密**  

### shell加入交互

**link_db.sh**脚本样例如下 ：

```
#!/bin/bash
comparepass='0932313'
read -s -p "Please enter the cipher:" importpass
if [ "$importpass" == "$comparepass" ];then
        /usr/local/bin/mysql -uroot -p'xxxxxxxxx' --socket=/tmp/mysql.sock --port=3306 
else
        echo -e "\n"
        echo "Sorry,U import a wrong cipher!"
fi
exit 0
```

脚本执行起来是酱婶儿的：

```
$  sh link_db.sh  
Please enter the cipher:
```

也就是和脚本进入问答环节，只有答对了"芝麻开门"（正确的口令）才可以连接进数据库（也就是被正确的执行），这时候只需输入上面我们设置的口令 **“0932313”**，即可执行数据库登入：

```
$ sh link_db.sh 
Please enter the cipher:Warning: 
Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6443684
Server version: 5.6.29-76.2-log Percona Server (GPL), Release 76.2, Revision ddf26fe

Copyright (c) 2009-2016 Percona LLC and/or its affiliates
Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

是不是很简单呢？  OK，下面我们进行要进行最神秘的一步。

### 加密脚本步骤

加密采用工具shc脚本加密工具，此工具安全性极高，破译极其困难。以下为下载、安装及加密的过程。

1.下载、安装
  
```
$ wget http://www.datsi.fi.upm.es/~frosal/sources/shc-3.8.9b.tgz
$ tar xvf shc-3.8.9b.tgz
$ cd shc-3.8.9b
$ mkdir -p /usr/local/man/man1/  # 创建目录这一步这个是必须的，没这个目录会报错
$ make install 
```

2.对脚本进行加密

安装后shc会放在/usr/local/bin/目录下。如想方便请做软链。

```
$ ln –s /usr/local/bin/shc /usr/bin/shc
```

就以刚刚的 **link_db.sh** 为例，进行加密。

**样例**

```
$ shc -v -r -T  -f /path/to/script/link_db.sh
```

**加密**

```
$ shc -v -r -T  -f link_db.sh
shc shll=bash
shc [-i]=-c
shc [-x]=exec '%s' "$@"
shc [-l]=
shc opts=
shc: cc  link_db.sh.x.c -o link_db.sh.x
shc: strip link_db.sh.x
shc: chmod go-r link_db.sh.x
```

执行后，会在脚本所在目录生成两个文件，分别为
- link_db.sh.x
- link_db.sh.x.c

**link_db.sh.x.c** 是脚本的源文件，可以直接删除。 link_db.sh.x就是原来脚本的可执行文件，可随意改名，不用赋权，shc处理的过程中有赋权这一步。

可随意把**link_db.sh.x**改成别的你想要的名字，例如 **linkdb**， 放入 **/usr/bin/**下面，试试直接输入**linkdb**然后执行：

```
$  linkdb 
Please enter the cipher:
```

### 结束语
  
  至此，这个小技巧就分享完毕了，学习此小技巧的目的是为了更好的保护我们程序和帐号，不被所谓的坏人窃取或者不正当使用，并不是让某些人把自己的任何脚本加密以达到拒绝分享的目的，搞技术最重要的就是分享，请谨慎的使用本功能，如果使用目的有悖博主初衷，请先过了自己心里那一关再进行使用吧，以上。