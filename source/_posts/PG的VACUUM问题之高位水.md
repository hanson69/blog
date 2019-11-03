---
title: PG的VACUUM问题之高位水
date: 2019-10-21 23:57:43
categories: PG基础
tags:
- VACUUM
- 死元组
- 高水位
- 存储
---


VACUUM之后的空间不会回收到文件系统中。只有将来的插入物可以使用此空间。

除非死元组超出高水位线，否则VACUUM通常不会将空间回收到文件系统。

为什么要这样做？

因为索引对应记录是通过记录物理位置来实现的，如果将表页前面的记录删除，在进行回收空间时，将表页后面的元组向前移，这时记录所在的物理块号发生变化，会引起索引和记录的对应关系发生错误，需要重建索引和记录的对应关系，这种开销就比较大了（vacuum full 开销大的原因）。但是如果删除的是表页后面的记录，在进行空间回收时，回收的只是表页后面的死元组，而活元组物理位置没有发生变化，不会引起索引和记录的对应关系发生错误，不需要对表页进行重构。

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

请注意，使用VACUUM FULL，会锁住表并且重写整个关系。如果是一个小的表，这可能不成问题。但如果表很大，这种表锁会影响用户数分钟之久！VACUUM FULL会阻塞即格到来的写操作并且有些人会觉得数据库看上去已经宕机了。

## 再看一个相似的问题

```SQL
postgres=# CREATE TABLE t_test(id int) WITH (autovacuum_enabled=off); 
postgres=# INSERT INTO t_test SELECT * FROM generate_series(1,100000);
```
创建一个包含100000行的简单表。注意可以对特定的表关闭autovacuum，但对于大部分应用这通常不是个好主意。不过，在一些极限情况中autovacuum_enabled=off是有意义的。只要考虑一个生命周期很短的表。如果开发者已经知道整个表将在数秒之后被删除，对它清理元组就没有意义了。在数据仓库中，如果用户把表用作暂存区就会是这样。在这个例子中，VACUUM被关闭以保证不会在后台发生其他事情，读者所看到的都是由笔者触发而不是被某个其他进程所执行。

首先检查该表的尺寸：

```SQL
postgres=# SELECT pg_size_pretty(pg_relation_size('t_test'));
 pg_size_pretty
----------------
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
postgres=# SELECT pg_size_pretty(pg_relation_size('t_test'));
 pg_size_pretty
----------------
 7080 kB
(1 row)
```

在UPDATE之后，人们可能会尝试把空间还给文件系统：

```SQL
postgres=# VACUUM VERBOSE t_test;
INFO:  vacuuming "public.t_test"
INFO:  "t_test": removed 100000 row versions in 443 pages
INFO:  "t_test": found 100000 removable, 100000 nonremovable row versions in 885 out of 885 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 568
There were 0 unused item pointers.
Skipped 0 pages due to buffer pins, 0 frozen pages.
0 pages are entirely empty.
CPU: user: 0.08 s, system: 0.00 s, elapsed: 0.09 s.
VACUUM
```

如前所述，大部分情况下VACUUM不把空间还给文件系统。相反，它允许空间被重用。因此，该表根本不会收缩：

```SQL
postgres=# SELECT pg_size_pretty(pg_relation_size('t_test'));
 pg_size_pretty
----------------
 7080 kB
(1 row)
```

不过，下一次UPDATE不会让该表长大，因为它会用掉该表中的空闲空间。只有第二次的UPDATE会让该表再次长大，因为所有的空间都用完了，需要额外的存储空间：

```SQL
postgres=# UPDATE t_test SET id = id +1;
UPDATE 100000

postgres=# SELECT pg_size_pretty(pg_relation_size('t_test'));
 pg_size_pretty
----------------
 7080 kB
(1 row)

postgres=# UPDATE t_test SET id = id +1;
UPDATE 100000
postgres=# SELECT pg_size_pretty(pg_relation_size('t_test'));
 pg_size_pretty
----------------
 10 MB
(1 row)
```

运行更多的查询：

```SQL
postgres=# VACUUM VERBOSE t_test;
INFO:  vacuuming "public.t_test"
INFO:  "t_test": removed 100000 row versions in 885 pages
INFO:  "t_test": found 100000 removable, 100000 nonremovable row versions in 1328 out of 1328 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 570
There were 118 unused item pointers.
Skipped 0 pages due to buffer pins, 0 frozen pages.
0 pages are entirely empty.
CPU: user: 0.08 s, system: 0.00 s, elapsed: 0.08 s.
VACUUM


postgres=# UPDATE t_test SET id = id + 1;
UPDATE 100000

postgres=# VACUUM VERBOSE t_test;
INFO:  vacuuming "public.t_test"
INFO:  "t_test": removed 100000 row versions in 442 pages
INFO:  "t_test": found 100000 removable, 100000 nonremovable row versions in 887 out of 1328 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 571
There were 357 unused item pointers.
Skipped 0 pages due to buffer pins, 441 frozen pages.
0 pages are entirely empty.
CPU: user: 0.07 s, system: 0.00 s, elapsed: 0.07 s.
VACUUM

```
尺寸还是没有改变。让我们看看表里有什么：

```SQL
postgres=# SELECT ctid, * FROM t_test ORDER BY ctid DESC;
   ctid    |   id
-----------+--------
 (1327,46) |    112
 (1327,45) |    111
 ...
 (1327,25) |     91
 (1327,24) |     90
 (884,20)  |  99798
 (884,19)  |  99797
 ...
 (884,12)  |  99790
 (884,11)  |  99789
 (442,85)  |     89
 (442,84)  |     88
 ...
 (442,2)   |      6
 (442,1)   |      5
 (441,216) | 100004
 (441,215) | 100003
... 
```

ctid是行在磁盘上的物理位置。通过使用ORDER BY ctid DESC，用户将按照物理顺序从后向前读到该表(为什么有这么多小值和大值在该表的未尾？在该表最初被100000个行填充之后，最后一个块并没有被完全填满，因此第一次的UPDATE将用更改填满最后一块。这自然把该表的末尾揽乱了一点。)。

这里的表记录在物理上存储的布局如下：

序号| ctid | id
---|---|---
 1| (0, 1) ~ (440, 226) | 123 ~ 99788
 2| (441, 1) ~ (441, 10) | 113 ~ 122
 3| (441,11) ~ (441,216) | 99799 ~ 100000
 4|(442, 1) ~ (442, 85) | 5 ~ 89
5| (884, 11) ~ (884, 20) | 99789 ~ 99798
6| (1327, 24) ~ (1327, 46) | 90 ~ 112




如果将末尾数据删除会发生什么？

```SQL
postgres=# DELETE FROM t_test WHERE id > 99000 OR id < 1000;
DELETE 1999

postgres=# VACUUM VERBOSE t_test;
INFO:  vacuuming "public.t_test"
INFO:  "t_test": removed 1999 row versions in 12 pages
INFO:  "t_test": found 1999 removable, 143 nonremovable row versions in 12 out of 1328 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 572
There were 465 unused item pointers.
Skipped 0 pages due to buffer pins, 883 frozen pages.
0 pages are entirely empty.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s.
INFO:  "t_test": truncated 1328 to 438 pages
DETAIL:  CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.01 s
VACUUM

-- 显示 "t_test": truncated 1328 to 438 pages

postgres=# SELECT pg_size_pretty(pg_relation_size('t_test'));
 pg_size_pretty
----------------
 3504 kB
(1 row)
```

尽管只有2%的数据被删除，该表的尺寸却下降了三分之二。因为 id > 99000 OR id < 1000 的记录很多都位于物理存储的后面（序号2 - 6），这时如果将它们删除，执行vacuum会将表中位于物理存储后面的死元组全部回收（序号2 - 6）。


### 参考
https://www.percona.com/blog/2018/08/06/basic-understanding-bloat-vacuum-postgresql-mvcc/


