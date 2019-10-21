---
title: PostgreSQL中的VACUUM
date: 2019-10-21 23:57:43
categories: DB基础
tags:
- VACUUM
- 死元祖
- 高水位
- 
- 
- 
---


表中已删除的tuple，但仍占用空间的记录称为Dead tuple。一旦已经运行的事务不再依赖于那些死元组，则不再需要死元组。因此，PostgreSQL在此类表上运行VACUUM，VACUUM回收这些死元组占用的存储。这些死元组占据的空间可以称为Bloat。VACUUM扫描页面中的死元组，并将它们标记到自由空间映射(FSM)中。除散列索引外，每个关系都有一个FSM，存储在称为<relation_oid> _fsm的单独文件中。


在使用VACUUM时，该空间不会回收到磁盘，但以后可以在此表上插入以重新使用。VACUUM将每个堆(或索引)页面上的可用空间存储到FSM文件中。

运行VACUUM是无阻塞操作。它永远不会在表上引起排他锁，这意味着VACUUM可以在生产中繁忙的事务表上运行。

对10条记录的更新会生成10个死元组。让我们看下面的日志，以了解VACUUM之后那些死元组会发生什么。

```SQL
postgres=# VACUUM employee ;
VACUUM
postgres=# SELECT t_xmin, t_xmax, tuple_data_split('employee'::regclass, t_data, t_infomask, t_infomask2, t_bits) FROM heap_page_items(get_raw_page('employee', 0));
 t_xmin | t_xmax |               tuple_data_split                
--------+--------+-----------------------------------------------
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
    673 |      0 | {"\\x01000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x02000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x03000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x04000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x05000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x06000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x07000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x08000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x09000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x0a000000","\\x0b61766969","\\x01000000"}
(20 rows)

postgres=# VACUUM employee ;
VACUUM
postgres=# SELECT t_xmin, t_xmax, tuple_data_split('employee'::regclass, t_data, t_infomask, t_infomask2, t_bits) FROM heap_page_items(get_raw_page('employee', 0));
 t_xmin | t_xmax |               tuple_data_split                
--------+--------+-----------------------------------------------
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
    673 |      0 | {"\\x01000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x02000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x03000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x04000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x05000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x06000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x07000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x08000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x09000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x0a000000","\\x0b61766969","\\x01000000"}
(20 rows)
```

可能会注意到死掉的元组已被删除，并且该空间可供重用。但是，VACUUM之后该空间不会回收到文件系统中。只有将来的插入物可以使用此空间。

VACUUM还要执行其他任务。过去插入并成功提交的所有行都标记为**冻结**，这表明它们对于所有当前和将来的事务都是可见的。

除非死元组超出高水位线，否则VACUUM通常不会将空间回收到文件系统。

让我们考虑以下示例，以了解VACUUM何时可以将空间释放到文件系统。

创建一个表并插入一些记录。记录是根据主键索引在磁盘上物理排序的。



```SQL
postgres=# CREATE TABLE employee (emp_id int PRIMARY KEY, name varchar(20), dept_id int);
CREATE TABLE
postgres=# INSERT INTO employee VALUES (generate_series(1,1000), 'avi', 1);
INSERT 0 1000

```

现在，在表上运行ANALYZE以更新其统计信息，并查看上述插入后向该表分配了多少页。



```SQL
postgres=# ANALYZE employee ;
ANALYZE
postgres=# select relpages, relpages*8192 as total_bytes, pg_relation_size('employee') as relsize 
FROM pg_class 
WHERE relname = 'employee';
relpages | total_bytes | relsize 
---------+-------------+---------
6        | 49152       | 49152
(1 row)
```

现在让我们看看当删除emp_id> 500的行时VACUUM的行为



```SQL
postgres=# DELETE from employee where emp_id > 500;
DELETE 500
postgres=# VACUUM ANALYZE employee ;
VACUUM
postgres=# select relpages, relpages*8192 as total_bytes, pg_relation_size('employee') as relsize 
FROM pg_class 
WHERE relname = 'employee';
relpages | total_bytes | relsize 
---------+-------------+---------
3        | 24576       | 24576
(1 row)
```

可以看到VACUUM已回收了文件系统一半的空间。之前，它占据了6页(每个8KB)。VACUUM之后，它已将3页释放到文件系统。

现在，我们通过删除emp_id <500的行来重复相同的操作



```SQL
postgres=# DELETE from employee ;
DELETE 500
postgres=# INSERT INTO employee VALUES (generate_series(1,1000), 'avi', 1);
INSERT 0 1000
postgres=# DELETE from employee where emp_id < 500;
DELETE 499
postgres=# VACUUM ANALYZE employee ;
VACUUM
postgres=# select relpages, relpages*8192 as total_bytes, pg_relation_size('employee') as relsize 
FROM pg_class 
WHERE relname = 'employee';
 relpages | total_bytes | relsize 
----------+-------------+---------
        6 |       49152 |   49152
(1 row)
```

在上面的示例中，从表中删除一半的记录后，页数仍然保持不变。这意味着，VACUUM这次没有释放空间到文件系统。

如前所述，如果在高水位标记之后的页面中没有更多的活动元组，则可以通过VACUUM将后续页面刷新到磁盘上。在第一种情况下，可以理解的是，第3页之后不再有活动元组。因此，第4、5和6页已刷新到磁盘。

但是，如果在我们删除了emp_id <500的所有记录的情况下需要在文件系统中回收空间，则可以运行VACUUM FULL。VACUUM FULL重建整个表并回收磁盘空间。



```SQL
postgres=# VACUUM FULL employee ;
VACUUM
postgres=# VACUUM ANALYZE employee ;
VACUUM
postgres=# select relpages, relpages*8192 as total_bytes, pg_relation_size('employee') as relsize 
FROM pg_class 
WHERE relname = 'employee';
 relpages | total_bytes | relsize 
----------+-------------+---------
        3 |       24576 |   24576
(1 row)
```

请注意，使用VACUUM FULL，会锁住表并且重写整个关系。如果是一个小的表，这可能不成问题。但如果表很大，这种表锁会影响用户数分钟之久！VACUUM FULL会阻塞即格到来的写操作并且有些人会觉得数据库看上去已经宿掉了。

第二个例子

```SQL
postgres=# CREATE TABLE t_test(id int) WITH (autovacuum_enabled=off); 
postgres=# INSERT INTO t_test SELECT * FROM generate_series(1,100000);
```
创建一个包含100000行的简单表。注意可以对特定的表关闭autovacuum，但对于大部分应用这通常不是个好主意。不过，在一些极限情况中autovacuum_enabled=off是有意义的。只要考虑一个生命周期很短的表。如果开发者已经知道整个表将在数秒之后被删除，对它清理元组就没有意义了。在数据仓库中，如果用户把表用作暂存区就会是这样。在这个例子中，VACUUM被关闭以保证不会在后台发生其他事情，读者所看到的都是由笔者触发而不是被某个其他进程所执行。

首先检查该表的尺寸：

```SQL
postgres=# SELECT pg_size_pretty(pg_relation_size("t_test"));
pg_size pretty
---------------
3544 kB
(1 row)
```

然后对表中的所有行都更新：

```SQL
postgres=# UPDATE t_test SET id = id +1;
UPDATE 100000
```

这里数据库引擎不得不复制所有的行。为什么会这样？首先，我们不知道该事务是否将会成功，因此这些数据不能被覆盖。第二个重要的方面是，可能有一个并发事务仍然在用这些数据的旧版本。

逻辑上，该表的尺寸将会在修改之后变得更大：

```SQL
postgres=# SELECT pg_size_pretty(pg_relation_size('t test'));
pg_size pretty
---------------
7080kB 
(1 row)
```

在UPDATE之后，人们可能会尝试把空间还给文件系统：

```SQL
postgres=# VACUUM t_test;
```

如前所述，大部分情况下VACUUM不把空间还给文件系统。相反，它允许空间被重用。因此，该表根本不会收缩：

```
postgres=# SELECT pg_size_pretty(pg_relation_size('t_test'));
pg_size pretty
---------------
7080 kB
(1row)
```

不过，下一次UPDATE不会让该表长大，因为它会用掉该表中的空闲空间。只有第二次的UPDATE会让该表再次长大，因为所有的空间都用完了，需要额外的存储空间：

```SQL
postgres=# UPDATEt test SET id = id + 1;
UPDATE 100000
postgres=# SELECT pg_size_pretty(pg_relation_size('t_test'));
pg_size pretty
---------------
7080 kB
(1 row)

postgres=#  UPDATE t_test SET id = id + 1; 
UPDATE 100000
postgres=#  SELECT pg_size_pretty(pg_relation_size("t_test')); 
pg_size_pretty
---------------
10 MB
(1 row)
```

运行更多的查询：

```SQL
postgres=# VACUUM t_test;
postgres=# UPDATE t_test SET id = id + 1;
postgres=# VACUUM t_test;
```
尺寸还是没有改变。让我们看看表里有什么：

```SQL
postgres=# SELECT ctid，*FROM t_test ORDER BY ctid DESC;
   ctid    | id
--------- -+--------
 (1327,46) | 112
 (1327,45) | 111
 (1327,44) | 110
 ...
 (884,20)  | 99798
 (884,19)  | 99797
... 
```

ctid是行在磁盘上的物理位置。通过使用ORDER BY ctid DESC，用户将按照物理顺序从后向前读到该表(为什么有这么多小值和大值在该表的未尾？在该表最初被100000个行填充之后，最后一个块并没有被完全填满，因此第一次的UPDATE将用更改填满最后一块。这自然把该表的末尾揽乱了一点。)。如果将末尾数据删除会发生什么？

```SQL
postgres=# DRLETE EROM t_test WHERE id >99000 oR id <1000;
postgres=# VACUUM t_test;
VACUUM 
postgres=# SBLRcT B9 size pretty(eg_relation size('t test，));
pg_size pretty
---------------
3504 kB
(1 row)
```

尽管只有2%的数据被删除，该表的尺寸却下降了三分之二。



