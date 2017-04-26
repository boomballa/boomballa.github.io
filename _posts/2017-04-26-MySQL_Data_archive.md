---
layout: post
title: "MySQL 数据定期归档及表空间整理"
date: 2017-04-26
tags: MySQL,Shell,Python 
---

### 开场白
  好久没来啦，年后一直在处理一下琐碎的问题，还有挺多新项目上线，所以一直木有时间更新，有点小冷清了。 

	
### 情况介绍	


  今天来分享一个，我从事MySQL以来，经常遇见的一个问题，就是关于MySQL数据库中大表的数据归档。	
  有时候我们库中会存在一些类似于日志表、流水表，还有些表中的数据时效性过了就会成为冷数据，那么这些表中的数据大多都是冷数据，有些数据甚至过后没有查一下的余地。	
  但是它们就在真真实实的占用着数据库的磁盘空间。 	
  如鲠在喉啊，有木有？ 删又不能删（开发有时候说有可能还会查），Truncate又不能Truncate。	


<img src="/images/posts/mysql_data_archive/ganga.jpg" height="222" width="237">

	
那今天我就介绍下我针对这种情况所采取的措施。 前方高能预警，请自带安全帽进入施工现场。

### 场景及准备工作

**数据库IP：172.16.3.88**

**数据库名称：institute**

**需要归档的表为： call_record , net_flow , sms_record**
	
**1.归档表的规则要提前和负责项目的开发人员沟通，确定热数据的时间范围，我这边的规则是10分钟以前的都可以归档。**

**2.既然是按时间归档，那么call_record , net_flow , sms_record三张表都要有create_time字段并且要有索引。**

**3.call_record , net_flow , sms_record三张表都要有id自增主键。（虽然像是废话，但是还要说一下，必须要有）**

**4.针对call_record , net_flow , sms_record 三张表，按照原表结构创建三张历史表 call_record_history , net_flow_history , sms_record_history 。**

**5.需要安装的工具： [percona-toolkit](https://www.percona.com/downloads/percona-toolkit/LATEST/)  [MySQL-python](https://pypi.python.org/pypi/MySQL-python/1.2.5) （版本自己控制就好）**

**6.创建归档数据用户restore，密码：pwd4mysql**
	
```
mysql> GRANT CREATE, DROP, PROCESS, ALTER, SUPER, REPLICATION SLAVE, TRIGGER ON *.* TO 'restore'@'172.16.3.88' IDENTIFIED BY PASSWORD '*D83D4673BB4CB13F4AE6255A00A71AA1A3CFE6B6';
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON `institute`.* TO 'restore'@'172.16.3.88'
```
**备注：这里提醒一下，开权限尽量开按IP地址开吗，脚本连数据库，虽说在本地也要按照IP去连，因为如果权限开到 'restore'@'localhost' ，连接数据库时会检查soscket地址，还需要另外做软链，很麻烦就对了。 如果服务器上面跑得不是一个实例，真的很难过。 总之绕开就好。 **
	
	
### 正文
    
** 一、首先是针对每张表用python脚本，查出10分钟前最小id，最大id。  然后取刚刚查出来的最小id和最大id的所有数据每次1000条，插入本库历史表。**
	
** 分享一下其中一张表的脚本，剩下两张同理 **
	
```
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
    
	 

完美 ~


### 结束语
  好啦，安装部署的部分就讲到这里了，博主这几天将继续对Group_replication进行研究，近期还会有关于此功能的文章出现，好啦，下班回家吃饭。