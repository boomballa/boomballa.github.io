---
layout:     post
title:      "MySQL历史数据归档及表空间整理"
subtitle:   "Hello World, Hello Blog"
date:       2017-04-26 00:17:03
author:     "Boomballa"
header-img: "img/home-bg.jpg"
catalog:    true
tags:
    - MySQL
    - Python
    - Percona-toolkit
---

### 开场白
  好久没来啦，年后一直在处理一些琐碎的问题，还有挺多新项目上线，所以一直木有时间更新。

	
### 情况介绍
  今天来分享一个，我从事MySQL以来，经常遇见的一个问题，就是关于MySQL数据库中大表的数据归档。   
  有时候我们库中会存在一些类似于日志表、流水表，还有些表中的数据时效性过了就会成为冷数据，那么这些表中的数据大多都是冷数据，有些数据甚至过后没有查一下的余地。   
  但是它们就在真真实实的占用着数据库的磁盘空间。    
  如鲠在喉啊，有木有？ 删又不能删（开发有时候说有可能还会查），Truncate又不能Truncate。（手动尴尬）
  
<img src="/img/in-post/mysql_data_archive/ganga.jpg" height="222" width="237">
	
那今天我就介绍下我针对这种情况所采取的措施。 前方高能预警，请自带安全帽进入施工现场。

### 场景及准备工作

数据库IP: 172.16.3.88    
历史库IP: 172.16.3.188    
数据库名称: institute    
需要归档的表为: call_record,net_flow,sms_record
	
1.归档表的规则要提前和负责项目的开发人员沟通，确定热数据的时间范围，我这边的规则是10分钟以前的都可以归档。    
2.既然是按时间归档，那么call_record , net_flow , sms_record三张表都要有create_time字段并且要有索引。    
3.call_record,net_flow,sms_record三张表都要有id自增主键。（虽然像是废话，但是还要说一下，必须要有）    
4.针对call_record,net_flow,sms_record 三张表，按照原表结构创建三张历史表 call_record_history , net_flow_history , sms_record_history 。    
5.需要安装的工具：[percona-toolkit](https://www.percona.com/downloads/percona-toolkit/LATEST/)  [MySQL-python](https://pypi.python.org/pypi/MySQL-python/1.2.5) （版本自己控制就好）    
6.搭建TokuDB历史库(172.16.3.188) ，这个自己选择，可以不搭建。  
7.主库和历史库创建归档数据用户restore，密码：pwd4mysql    
    
```mysql
mysql> GRANT CREATE, DROP, PROCESS, ALTER, SUPER, REPLICATION SLAVE, TRIGGER ON *.* TO 'restore'@'172.16.3.%' IDENTIFIED BY PASSWORD '*D83D4673BB4CB13F4AE6255A00A71AA1A3CFE6B6';
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON `institute`.* TO 'restore'@'172.16.3.%';
```

> **备注：**    
> **这里提醒一下，开权限尽量开按IP地址开吗，脚本连数据库，虽说在本地也要按照IP去连，因为如果权限开到'restore'@'localhost'，连接数据库时会检查soscket地址，还需要另外做软链，很麻烦就对了。 如果服务器上面跑得不是一个实例，真的很难过。 总之绕开就好。**


### 正文
    
**首先是针对每张表用python脚本，查出10分钟前最小id，最大id。  然后取刚刚查出来的最小id和最大id的所有数据每次1000条，插入本库历史表。**
	
**分享一下其中一张表的脚本，剩下两张同理。**
	
```python
#!/usr/bin/python
#encoding=utf-8
import sys,time
import MySQLdb
from __builtin__ import str

reload(sys)
sys.setdefaultencoding('utf-8')

def restore_call_record_data():
    list_id = find_min_max_id('call_record')
    print list_id
    start_value=list_id[0]
    max_value=list_id[1]
    step=1000

    while (start_value < max_value):

        end_value = start_value + step
        if (start_value+step)>max_value :
                end_value = max_value

        conn=MySQLdb.connect(host="172.16.3.88",user="restore",passwd="pwd4mysql",db="institute_call_record",charset="utf8")
        cursor =conn.cursor()
        count_sql="select count(1) from call_record t where created_time<date_sub(now(),interval 10 MINUTE) and id<="+str(end_value)+" and id>="+str(start_value)+";"
        sql ="select * from call_record t where created_time<date_sub(now(), interval 10 MINUTE) and id<="+str(end_value)+" and id>="+str(start_value)+";"
        print sql
        cursor.execute(count_sql)
        print cursor.fetchone()
        cursor.execute(sql)
        rows=cursor.fetchall()
        if rows:
            sql_ids=''
            flag=1
            for row in rows:
                if flag==1:
                    sql_ids=str(row[0])
                else:
                    sql_ids=sql_ids+','+str(row[0])
                flag=flag+1

            sql_insert="replace into call_record_history select * from call_record where id in("+sql_ids+");"
            #print sql_insert
            sql_delete="delete from call_record where id in("+sql_ids+");"
            #print sql_delete
            try:
                cursor.execute(sql_insert)
                cursor.execute(sql_delete)
                cursor.execute("commit;")
                time.sleep(0.5)
                #print "call_record-->start_value="+str(start_value)+"end_value="+str(end_value)
            except:
                print "call_record id="+str(row[0])+"!!!"
                cursor.execute("rollback;")
                cursor.close()
                conn.close()
                sys.exit(0)


            start_value = end_value+1
        else:
            start_value = end_value+1
            continue
            time.sleep(0.01)
        cursor.close()
        conn.close()

def find_min_max_id(table_name):
    find_sql="select ifnull(min(id),0),ifnull(max(id),0) from "+table_name+" where created_time<date_sub(now(),interval 10 MINUTE);"
    print find_sql
    conn=MySQLdb.connect(host="172.16.3.88",user="restore",passwd="pwd4mysql",db="institute",port=3306,charset="utf8")
	cursor =conn.cursor()
    cursor.execute(find_sql)
    row=cursor.fetchone()

    list_id = [0,0]
    list_id[0]=int(row[0])
    list_id[1]=int(row[1])

    cursor.close()
    conn.close()
    return list_id


if __name__ == '__main__':
    print "call_record_data---------"
    ISOTIMEFORMAT='%Y-%m-%d %X'
    print time.strftime( ISOTIMEFORMAT, time.localtime( time.time() ) )+"-------------------开始restore-----------------------"
    restore_call_record_data()
    print time.strftime( ISOTIMEFORMAT, time.localtime( time.time() ) )+"restory执行完毕。。。。。。。。。。。。。。。。。。。。"
```
    
**然后是表空间整理和数据归档**    
  以上步骤的脚本是处理表call_record的,net_flow和sms_record表同理，再编辑出两个脚本，下一步就是关键的整理表，那么我问题来了，为什么要整理表呢？    
  是因为原表删除过数据以后，原来数据的表空间依然保留，delete的操作只是在行数据上打上了delete的标签，并没有真正意义上删除。整理表大家都会：

```mysql
mysql> alter table call_record engiine='innodb';
mysql> alter table net_flow engiine='innodb';
mysql> alter table sms_record engiine='innodb';
```

是不是 ？

但是这个DDL操作会给表上锁，在正常生产环境中极有可能出现源数据锁，造成阻塞。而在我们脚本运行的时候，我们可能还在美梦中。 那画面太美，我真的不太敢想。 你懂的...

所以这时候就该使用到我们刚才所准备的[percona](https://www.percona.com/)工具了。[pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/2.2/pt-online-schema-change.html) 还有[pt-archiver](https://www.percona.com/doc/percona-toolkit/2.2/pt-archiver.html) 。他们两个都是干啥用的呢？

> **pt-online-schema-change 功能简介**    
  功能为在alter操作更改表结构的时候不用锁定表，也就是说执行alter的时候不会阻塞写和读取操作，注意执行这个工具的时候必须做好备份，操作之前最好详细读一下官方文档。        
  工作原理是创建一个和你要执行alter操作的表一样的空表结构，执行表结构修改，然后从原表中copy原始数据到表结构修改后的表，当数据copy完成以后就会将原表移走，用新表代替原表，默认动作是将原表drop掉。在copy数据的过程中，任何在原表的更新操作都会更新到新表，因为这个工具在会在原表上创建触发器，触发器会将在原表上更新的内容更新到新表。如果表中已经定义了触发器这个工具就不能工作了。

> **pt-archiver 功能简介**    
  该工具的目标是把线上的老数据转移,在转移过程中,不会对服务器产生任何冲击,同时也不会影响写入和查询.你可以把这些数据写入到另外一台MySQL,或者写到一个文件里面(该可以使用LOAD DATA INFILE语句导入),或者也可以直接做清理。
    
**可能看到这有点乱，咱缕缕:**

**1.通过脚本整理数据，把没用的数据归档到本实例上的该表的历史表。**    
**2.使用工具pt-archiver，将历史表的数据全部归档到历史库。待归档完成后，然后使用pt-archiverpt-online-schema-change整理历史表。（如果没有历史库，也可以使用Mysqldump把历史表dump出来之后，把历史表Truncate掉，也可以达到减少使用空间的目的）**    
**3.使用工具pt-archiverpt-online-schema-change，整理源表，将表空间释放，达到降低磁盘空间的目的。**

这里在解释一下，我这边生产环境是用[TokuDB](https://www.percona.com/software/mysql-database/percona-tokudb)引擎搭建的历史库，大概能比innodb引擎的库省下60%-70%的磁盘空间吧，而且TokuDB优秀的插入和查询速度正好符合我们使用历史库的初衷，我这边就不再赘述了，想要深入了解TokuDB，可以去[percona官网](https://www.percona.com/software/mysql-database/percona-tokudb)自己看一下它的优缺点，从而更好的使用它。
 
**整理后的shell脚本如下，我自己加入了邮件设置，这个大家自己选择性的去配置，以下仅供大家参考：**    

```shell
#!/bin/bash
to=songpeng@xxx.com
today=`date +%Y-%m-%d`

python /opt/shells/restore_call_record.py >>/tmp/restore_institute_data.log
date +"call_record表清理结束时间：%Y-%m-%d %H:%M:%S" >>/tmp/restore_institute_data.log
python /opt/shells/restore_net_flow.py >>/tmp/restore_institute_data.log
date +"net_flow表清理结束时间：%Y-%m-%d %H:%M:%S" >>/tmp/restore_institute_data.log
python /opt/shells/restore_sms_record.py >>/tmp/restore_institute_data.log
date +"sms_record表清理结束时间：%Y-%m-%d %H:%M:%S" >>/tmp/restore_institute_data.log

#整理表
#date +"整理表开始时间：%Y-%m-%d %H:%M:%S" >> /tmp/restore_institute_data.log
if [ $? -eq 0 ];  then

#向历史库归档历史表
pt-archiver --source h=172.16.3.88,P=3306,u=restore,p='pwd4mysql',D=institute,t=call_record_history,A=utf8 --dest h=172.16.3.188,P=3306,u=restore,p='pwd4mysql',D=institute,t=order_history,A=utf8 --where="1=1" --limit 1000 --replace
pt-archiver --source h=172.16.3.88,P=3306,u=restore,p='pwd4mysql',D=institute,t=net_flow_history,A=utf8 --dest h=172.16.3.188,P=3306,u=restore,p='pwd4mysql',D=institute,t=order_history,A=utf8 --where="1=1" --limit 1000 --replace
pt-archiver --source h=172.16.3.88,P=3306,u=restore,p='pwd4mysql',D=institute,t=order_history,A=utf8 --dest h=172.16.3.188,P=3306,u=restore,p='pwd4mysql',D=institute,t=sms_record_history,A=utf8 --where="1=1" --limit 1000 --replace

#历史表归档完毕，整理历史表空间
pt-online-schema-change  -urestore -ppwd4mysql --socket=/opt/app/mysql/tmp/mysql.sock --port=3306 --alter='engine="innodb"' --recurse=0 --execute D=institute,t=call_record_history  -A utf8 --print
echo "call_record_history表 整理完毕..." >> /tmp/restore_institute_data.log
pt-online-schema-change  -urestore -ppwd4mysql --socket=/opt/app/mysql/tmp/mysql.sock --port=3306 --alter='engine="innodb"' --recurse=0 --execute D=institute,t=net_flow_history  -A utf8 --print
echo "net_flow_history表 整理完毕..." >> /tmp/restore_institute_data.log
pt-online-schema-change  -urestore -ppwd4mysql --socket=/opt/app/mysql/tmp/mysql.sock --port=3306 --alter='engine="innodb"' --recurse=0 --execute D=institute,t=sms_record_history -A utf8 --print
echo "sms_record_history表 整理完毕..." >> /tmp/restore_institute_data.log

#整理源表空间
pt-online-schema-change  -urestore -ppwd4mysql --socket=/opt/app/mysql/tmp/mysql.sock --port=3306 --alter='engine="innodb"' --recurse=0 --execute D=institute,t=call_record  -A utf8 --print
echo "call_record表 整理完毕..." >> /tmp/restore_institute_data.log
pt-online-schema-change  -urestore -ppwd4mysql --socket=/opt/app/mysql/tmp/mysql.sock --port=3306 --alter='engine="innodb"' --recurse=0 --execute D=institute,t=net_flow  -A utf8 --print
echo "net_flow表 整理完毕..." >> /tmp/restore_institute_data.log
pt-online-schema-change  -urestore -ppwd4mysql --socket=/opt/app/mysql/tmp/mysql.sock --port=3306 --alter='engine="innodb"' --recurse=0 --execute D=institute,t=sms_record -A utf8 --print
echo "sms_record表 整理完毕..." >> /tmp/restore_institute_data.log
echo "institute库清理结束" >/tmp/archiver_result.log
date +"结束时间：%Y-%m-%d %H:%M:%S" >>/tmp/archiver_result.log
cat /tmp/archiver_result.log | mutt -s "institute库清理 状态：成功 $today" $to
else
echo "oh no~ institute库清理出现问题" | mutt -s "institute库清理 状态：失败 $today" $to
```


 
### 结束语
   妥啦，今天历史数据归档的话题，就分享到这，如果哪点分享的有问题或者有更好的方式解决此问题，欢迎雅正和探讨。            
   
