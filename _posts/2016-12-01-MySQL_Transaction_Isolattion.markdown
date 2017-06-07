---
layout:     post
title:      "MySQL事务隔离级别详解"
subtitle:   "MySQL transaction isolation level is explained in detail"
date:       2016-12-01 00:47:02
author:     "Boomballa"
header-img: "img/home-bg.jpg"
catalog:    true
tags:
    - MySQL
---


### 前言
  
这篇文章是之前写在我的[csdn](http://write.blog.csdn.net/postlist?t=unlock&id=46049297)上面的，还是蛮重要的，拿过来吧，隔离级别在MySQL数据库中还是比较重要的，下边做了点小实验，分享给大家。

### 介绍一下隔离级别：

  SQL标准定义了4类隔离级别，包括了一些具体规则，用来限定事务内外的哪些改变是可见的，哪些是不可见的。低级别的隔离级一般支持更高的并发处理，并拥有更低的系统开销。

下面介绍一下这4种隔离级别：

- **READ UNCOMMITTED         读未提交**

  一个事务中，可以读取到其他事务未提交的变更。该级别允许脏读发生。

- **READ COMMITTED           读已提交**

  一个事务中，可以读取到其他事务已经提交的变更。该级别允许  幻读、不可重复读  发生。 

- **REPEATABLE READ        可重复读**  

  一个事务中，直到事务结束前，都可以反复读取到事务一开始看到的数据，不会发生变化。该级别可保证事务一致性。

- **SERIALIZABLE            串行**

  串行化读，即便每次读都需要获得表级共享锁，每次写都加表级排它锁，两个会话间的读写会相互阻塞。该级别会导致**InnoDB**表的并发特性丧失，变成和**MyISAM**一样。

###  在MySQL中，实现了这四种隔离级别，随着并发事务会带来一些问题：

  也就是：
  
- **Dirty read（脏读）**
  
- **Unrepeatable read（不可重复读）**
  
- **Phantom read（幻读）**

下面把会出现的这几个问题解释一下：

**Dirty read（脏读）**：脏读就是指当一个事务正在访问数据，并且对数据进行了修改，而这种修改还没有提交到[数据库](http://lib.csdn.net/base/14)中，这时，另外一个事务也访问这个数据，然后使用了这个数据。

**Unrepeatable read（不可重复读）**：是指在一个事务内，多次读同一数据。在这个事务还没有结束时，另外一个事务也访问该同一数据。那么，在第一个事务中的两 次读数据之间，由于第二个事务的修改，那么第一个事务两次读到的的数据可能是不一样的。这样就发生了在一个事务内两次读到的数据是不一样的，因此称为是不 可重复读。例如，一个编辑人员两次读取同一文档，但在两次读取之间，作者重写了该文档。当编辑人员第二次读取文档时，文档已更改。原始读取不可重复。如果 只有在作者全部完成编写后编辑人员才可以读取文档，则可以避免该问题。

**Phantom read（幻读）**：是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行。 同时，第二个事务也修改这个表中的数据，这种修改是向表中插入一行新数据。那么，以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行，就好象 发生了幻觉一样。例如，一个编辑人员更改作者提交的文档，但当生产部门将其更改内容合并到该文档的主复本时，发现作者已将未编辑的新材料添加到该文档中。 如果在编辑人员和生产部门完成对原始文档的处理之前，任何人都不能将新材料添加到文档中，则可以避免该问题。

### 实践出真知，我们来测试：

下面我参照 [JAVA夜无眠](http://xm-king.iteye.com/blog/770721) 的博客做了几个实验，运用**A**、**B**端两个事务分别测试几种隔离级别。测试数据库为test，表为t_test；表结构：

```
[test]>show create table t_test;
+--------+-------------------------------------------------------------------------------------------------------------------+
| Table  | Create Table                                                                                                      |
+--------+-------------------------------------------------------------------------------------------------------------------+
| t_test | CREATE TABLE `t_test` (
  `id` int(11) NOT NULL,
  `no` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+--------+-------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

**(一)、将A的隔离级别设置为Read Uncommitted(未提交读)**
事务**A**端**：**

```
[test]>SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

[test]>SELECT @@TX_ISOLATION;      
+------------------+
| @@TX_ISOLATION   |
+------------------+
| READ-UNCOMMITTED |
+------------------+
1 row in set (0.00 sec)
```

在**B**端未更新数据之前：
事务**A**端**：**

```
[test]>SELECT @@TX_ISOLATION;        
+------------------+
| @@TX_ISOLATION   |
+------------------+
| READ-UNCOMMITTED |
+------------------+
1 row in set (0.00 sec)

[test]>start transaction;
Query OK, 0 rows affected (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

**B**端更新数据：事务**B**端**：**

```
[test]>SET AUTOCOMMIT=0;  

[test]>update t_test set no=888 where id =1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

[test]>select * from t_test;         
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>rollback;
Query OK, 0 rows affected (0.06 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

事务**A**端**：**

```
[test]>start transaction;
Query OK, 0 rows affected (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

经过上面的实验可以得出结论，事务**B**端更新了一条记录，但是没有提交，此时事务**A**可以查询出未提交记录。造成脏读现象。未提交读是最低的隔离级别。

**(二)、将事务**A**端的事务隔离级别设置为Read Committed(已提交读)**

在**B**端未更新数据之前：
事务**A**端**：**

```
[test]>SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

[test]>SELECT @@TX_ISOLATION;        
+----------------+
| @@TX_ISOLATION |
+----------------+
| READ-COMMITTED |
+----------------+
1 row in set (0.00 sec)

[test]>start transaction;
Query OK, 0 rows affected (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

**B**端更新数据：
事务**B**端**：**

```
[test]>SET AUTOCOMMIT=0;  

[test]>update t_test set no=888 where id =1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>commit;
Query OK, 0 rows affected (0.23 sec)
```

事务**A**端**：**

```
[test]>start transaction;
Query OK, 0 rows affected (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

经过上面的实验可以得出结论，已提交读隔离级别解决了脏读的问题，但是出现了不可重复读的问题，即事务**A**端在两次查询的数据不一致，因为在两次查询之间事务**B**端更新了一条数据。已提交读只允许读取已提交的记录，但不要求可重复读。


**(三)、将A的隔离级别设置为Repeatable Read(可重复读)**

在**B**端未更新数据之前：
事务**A**端**：**

```
[test]>SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
 
[test]>SELECT @@TX_ISOLATION;        
 +-----------------+
| @@TX_ISOLATION  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

[test]>start transaction;            
Query OK, 0 rows affected (0.05 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

**B**端更新数据：
事务**B**端**：**

```
[test]>SET AUTOCOMMIT=0;

[test]>start transaction;
Query OK, 0 rows affected (0.00 sec)

[test]>update t_test set no=888 where id =1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

[test]>select * from t_test;         
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>commit;                
Query OK, 0 rows affected (0.06 sec)
```

事务**A**端**：**

```
[test]>start transaction;            
Query OK, 0 rows affected (0.05 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

**B**端插入数据：
事务**B**端**：**

```
[test]>insert into t_test(no)value(999);
Query OK, 1 row affected, 1 warning (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
|  0 |  999 |
+----+------+
5 rows in set (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
|  0 |  999 |
+----+------+
5 rows in set (0.00 sec)
```

事务**A**端**：**

```
[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)

[test]>commit;
Query OK, 0 rows affected (0.00 sec)

[test]>select * from t_test;
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

由以上的实验可以得出结论，可重复读隔离级别只允许读取已提交记录，而且在一个事务两次读取一个记录期间，其他事务部的更新该记录。但该事务不要求与其他事务可串行化。例如，当一个事务可以找到由一个已提交事务更新的记录，但是可能产生幻读问题(注意是可能，因为数据库对隔离级别的实现有所差别)。像以上的实验，就没有出现数据幻读的问题。

**(四)、将**A**端的隔离级别设置为 Serializable (可串行化)**

**A**端打开事务，**B**端插入一条记录
**A**端**：**

```
[test]>SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

[test]>SELECT @@TX_ISOLATION;     
+----------------+
| @@TX_ISOLATION |
+----------------+
| SERIALIZABLE   |
+----------------+
1 row in set (0.00 sec)

[test]>select * from t_test;      
+----+------+
| id | no   |
+----+------+
|  1 |  888 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
4 rows in set (0.00 sec)
```

事务**B**端**：**

```
[test]>SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
Query OK, 0 rows affected (0.00 sec)

[test]>SELECT @@TX_ISOLATION;        
+-----------------+
| @@TX_ISOLATION  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

[test]>insert into t_test(no)value(999);
```

因为此时事务**A**的隔离级别设置为serializable，开始事务后，并没有提交，所以事务**B**只能等待。

**A**端提交事务：
**A**端**：**

```
[test]>commit;
Query OK, 0 rows affected (0.00 sec)
```

事务**B**端**：**

```
[test]>insert into t_test(no)value(999);
Query OK, 0 rows affected (0.00 sec)
```

Serializable完全锁定字段，若一个事务来查询同一份数据就必须等待，直到前一个事务完成并解除锁定为止 。是完整的隔离级别，会锁定对应的数据表格，因而会有效率的问题。


### 归纳
  实验做完了，是的，正如你所看到的，MySQL的事物隔离级别和锁有着不可分割的关系，不知道介绍的详细不详细，希望可以帮助到大家。有疑问欢迎留言一起探讨。