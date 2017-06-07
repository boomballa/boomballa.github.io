---
layout:     post
title:      "MySQL同步故障处理(Last_Error: 1864)"
subtitle:   "MySQL Sync Troubleshooting (Last_Error: 1864)"
date:       2016-11-30 22:57:07
author:     "Boomballa"
header-img: "img/home-bg.jpg"
catalog:    fasle
tags:
    - MySQL
---

  之前朋友给我提供了一个主从同步故障处理的案例，在此share给大家，希望能帮到一些有需要的人。

**MySQL slave同步报错：**

```
$ Last_Error:1864
  Last_Error: Cannot schedule event Rows_query, relay-log name ./db-s18-relay-bin.000448, position 419156572 to Worker thread because its size 18483519 exceeds 16777216 of slave_pending_jobs_size_max.
```

  从字面意思看了一下是因为slave_pending_jobs_size_max默认值为16777216（16MB），但是slave接收到的slave_pending_jobs_size_max为18483519；

  首先查一下slave_pending_jobs_size_max；

```
[test]>show global variables like '%slave_pending_jobs_size_max%';
+-----------------------------+----------+
| Variable_name               | Value    |
+-----------------------------+----------+
| slave_pending_jobs_size_max | 16777216 |
+-----------------------------+----------+
1 row in set (0.00 sec)
```

  果然是因为slave_pending_jobs_size_max这个值为16777216，
  果断:

```
[test]>stop slave；
[test]>set global slave_pending_jobs_size_max=20000000;
[test]>start slave；
```

```
[test]>show global variables like '%slave_pending_jobs_size_max%';
+-----------------------------+----------+
| Variable_name               | Value    |
+-----------------------------+----------+
| slave_pending_jobs_size_max | 19999744 |
+-----------------------------+----------+
1 row in set (0.00 sec)
```

  至此已经顺利解决掉了。

### 介绍一下slave_pending_jobs_size_max的用途：
  在多线程复制时，在队列中Pending的事件所占用的最大内存，默认为16M，如果内存富余，或者延迟较大时，可以适当调大;注意这个值要比主库的max_allowed_packet大。
