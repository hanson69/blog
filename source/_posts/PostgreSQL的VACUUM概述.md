---
title: PostgreSQL的VACUUM概述
date: 2019-11-03, 21:08:59
categories: PG基础
tags:
- VACUUM 
- 死元组 
- FSM 
- VM
---
PostgreSQL的VACUUM主要有两个任务： 删除死元组和冻结旧元组的事务ID。

**死元组**：表中已删除的tuple，但仍占用空间的记录称为Dead tuple，这些死元组占据的空间可以称为Bloat。一旦已经运行的事务不再依赖于那些死元组，则不再需要死元组，而VACUUM就负责回收这些死元组占用的存储。

VACUUM分为2种：

> 1、标准的VACUUM
> 
> 2、VACUUM FULL 

#### 标准的VACUUM

标准的VACUUM执行过程如下：

> （1）从指定的表集中获取每个表。
> 
> （2）获取表的ShareUpdateExclusiveLock锁。此锁允许其他事务对该表进行SELECT、INSERT、UPDATE、DELETE等操作。
> 
> （3）扫描表中所有页面，以获取所有的死元组，并在必要时冻结旧的元组。
> 
> （4）删除指向各个死元组的索引元组（如果存在）。
> 
> （5）对表的每一页执行以下任务，即步骤（6）和（7）。
> 
> （6）删除死元组。
> 
> （7）更新目标表的相应FSM和VM。
> 
> （8）如果最后一页没有任何元组，则截断最后一页。
> 
> （9）更新与目标表清理有关的统计信息和系统视图。
> 
> （10）更新与清理有关的统计信息和系统视图。
> 
> （11）删除CLOG中不必要的文件和页面。

在使用标准的VACUUM时，该空间一般不会回收到磁盘（高水位的元组除外），但以后可以在此表上插入以重新使用。VACUUM将每个堆(或索引)页面上的可用空间存储到FSM文件中。

运行标准的VACUUM是无阻塞操作。它不会在表上加排他锁，这意味着VACUUM可以与SELECT语句和DML语句（如INSERT、UPDATE、DELETE命令）并行执行，但是若在清理该表，不能使用如ALTERTABLE这样的DDL语句来修改表定义。

#### VACUUM FULL
VACUUM FULL不能与其他使用该表的语句并行执行，但是VACUUM FULL可以回收更多的磁盘空间，当然它的运行速度也要慢得多。

VACUUM FULL执行过程如下：
```
(1)  FOR each table
(2)       Acquire AccessExclusiveLock lock for the table
(3)       Create a new table file
(4)       FOR each live tuple in the old table
(5)            Copy the live tuple to the new table file
(6)            Freeze the tuple IF necessary
          END FOR
(7)       Remove the old table file
(8)       Rebuild all indexes
(9)       Update FSM and VM
(10)      Update statistics
          Release AccessExclusiveLock lock
       END FOR
(11)  Remove unnecessary clog files and pages if possible
```
【1】当对表执行VACUUM FULL命令时，PostgreSQL首先获取该表的AccessExclusiveLock锁，然后创建一个大小为8KB的新表文件。AccessExclusiveLock锁不允许访问。

【2】将旧表文件中的活元组复制到新表中。

【3】复制所有活元组后，PostgreSQL删除旧文件，重建所有关联的表索引，更新该表的FSM和VM，并更新关联的统计信息和系统视图。

#### VACUUM实践
对比VACUUM的两种模式的区别。

```SQL
postgres=# select * from tb2;
 id | name
----+------
  1 | a
  2 | b
  3 | c
  4 | d
  5 | e
(5 rows)

postgres=# select * from heap_page_items(get_raw_page('tb2', 0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    610 |      0 |        0 | (0,1)  |           2 |       2306 |     24 |        |       | \x010000000561
  2 |   8128 |        1 |     30 |    611 |      0 |        0 | (0,2)  |           2 |       2306 |     24 |        |       | \x020000000562
  3 |   8096 |        1 |     30 |    612 |      0 |        0 | (0,3)  |           2 |       2306 |     24 |        |       | \x030000000563
  4 |   8064 |        1 |     30 |    613 |      0 |        0 | (0,4)  |           2 |       2306 |     24 |        |       | \x040000000564
  5 |   8032 |        1 |     30 |    614 |      0 |        0 | (0,5)  |           2 |       2306 |     24 |        |       | \x050000000565
(5 rows)

postgres=# delete from tb2 where id in (2, 4);
DELETE 2
postgres=# select * from tb2;
 id | name
----+------
  1 | a
  3 | c
  5 | e
(3 rows)

-- 查看页面，delete的元组并没有在页面中真正删除，只是t_xmax被修改的事务号标记了
postgres=# select * from heap_page_items(get_raw_page('tb2', 0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    610 |      0 |        0 | (0,1)  |           2 |       2306 |     24 |        |       | \x010000000561
  2 |   8128 |        1 |     30 |    611 |    615 |        0 | (0,2)  |        8194 |       1282 |     24 |        |       | \x020000000562
  3 |   8096 |        1 |     30 |    612 |      0 |        0 | (0,3)  |           2 |       2306 |     24 |        |       | \x030000000563
  4 |   8064 |        1 |     30 |    613 |    615 |        0 | (0,4)  |        8194 |       1282 |     24 |        |       | \x040000000564
  5 |   8032 |        1 |     30 |    614 |      0 |        0 | (0,5)  |           2 |       2306 |     24 |        |       | \x050000000565
(5 rows)

-- 执行vaccum
postgres=# vacuum tb2;
VACUUM

-- delete的元组空间被腾出，但是这块空间还在页面保留，其它元组的物理位置没有发生改变
postgres=# select * from heap_page_items(get_raw_page('tb2', 0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    610 |      0 |        0 | (0,1)  |           2 |       2306 |     24 |        |       | \x010000000561
  2 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       |
  3 |   8128 |        1 |     30 |    612 |      0 |        0 | (0,3)  |           2 |       2306 |     24 |        |       | \x030000000563
  4 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       |
  5 |   8096 |        1 |     30 |    614 |      0 |        0 | (0,5)  |           2 |       2306 |     24 |        |       | \x050000000565
(5 rows)

-- 执行 vacuum full
postgres=# vacuum FULL VERBOSE tb2;
INFO:  vacuuming "public.tb2"
INFO:  "tb2": found 0 removable, 3 nonremovable row versions in 1 pages
DETAIL:  0 dead row versions cannot be removed yet.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s.
VACUUM

-- vacuum full后页面中的元组物理位置发生了变化
postgres=# select * from heap_page_items(get_raw_page('tb2', 0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    610 |      0 |        0 | (0,1)  |           2 |       2818 |     24 |        |       | \x010000000561
  2 |   8128 |        1 |     30 |    612 |      0 |        0 | (0,2)  |           2 |       2818 |     24 |        |       | \x030000000563
  3 |   8096 |        1 |     30 |    614 |      0 |        0 | (0,3)  |           2 |       2818 |     24 |        |       | \x050000000565
(3 rows)

```


## 参考：

http://www.interdb.jp/pg/pgsql06.html

https://www.postgresql.org/docs/12/sql-vacuum.html

