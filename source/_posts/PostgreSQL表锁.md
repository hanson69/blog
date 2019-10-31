---
title: PostgreSQL表锁
date: 2019-10-31 22:52:19
categories: PG基础
tags:
- 表锁
- 共享锁
- 排斥锁
- 意向锁
- lock
---

在PostgreSQL数据库中有两类锁：表级锁和行级锁。当要查询、插入、更新、删除表中的数据时，首先是要获得表上的锁，然后再获得行上的锁。

表级锁模式

|         锁类型         |                                                              使用场景                                                             |
|:----------------------:|:---------------------------------------------------------------------------------------------------------------------------------:|
| ACCESS SHARE           | SELECT操作。通常，任何只读取表而不修改它的查询都将获得这种锁模式。                                                                |
| ROW SHARE              | SELECT FOR UPDATE 和 SELECT FOR SHARE                                                                                             |
| ROW EXCLUSIVE          | UPDATE、DELETE 和 INSERT，任何修改表中数据的命令取得                                                                              |
| SHARE UPDATE EXCLUSIVE | VACUUM（不带FULL）、ANALYZE、CREATE INDEX CONCURRENTLY、 CREATE STATISTICS 和 ALTER TABLE VALIDATE以及其他 ALTER TABLE 的变体获得 |
| SHARE                  | CREATE INDEX（不带 CONCURRENTLY）                                                                                                 |
| SHARE ROW EXCLUSIVE    | CREATE COLLATION、CREATE TRIGGER和很多 ALTER TABLE 的很多形式所获得                                                               |
| EXCLUSIVE              | REFRESH MATERIALIZED VIEW CONCURRENTLY                                                                                            |
| ACCESS EXCLUSIVE       | ALTER TABLE、DROP TABLE、TRUNCATE、REINDEX、CLUSTER、VACUUM FULL 和 REFRESH MATERIALIZED VIEW（不带CONCURRENTLY）                 |


### 表级锁冲突矩阵

|   Requested Lock Mode  | ACCESS SHARE | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE |
|:----------------------:|:------------:|:---------:|:-------------:|:----------------------:|:-----:|:-------------------:|:---------:|:----------------:|
|      ACCESS SHARE      |              |           |               |                        |       |                     |           |         X        |
|        ROW SHARE       |              |           |               |                        |       |                     |     X     |         X        |
|      ROW EXCLUSIVE     |              |           |               |                        |   X   |          X          |     X     |         X        |
| SHARE UPDATE EXCLUSIVE |              |           |               |            X           |   X   |          X          |     X     |         X        |
|          SHARE         |              |           |       X       |            X           |       |          X          |     X     |         X        |
|   SHARE ROW EXCLUSIVE  |              |           |       X       |            X           |   X   |          X          |     X     |         X        |
|        EXCLUSIVE       |              |     X     |       X       |            X           |   X   |          X          |     X     |         X        |
|    ACCESS EXCLUSIVE    |       X      |     X     |       X       |            X           |   X   |          X          |     X     |         X        |

一旦被获取，一个锁通常将被持有直到事务结束。 但是如果在建立保存点之后才获得锁，那么在回滚到这个保存点的时候将立即释放该锁。

在表中，“X”表示这两种锁冲突，也就是不同的用户不能同时持有这两种锁，其它
表示可以兼容。

首先，表级锁只有“SHARE”和“EXCLUSIVE”这两种锁。很容易理解，这两种锁基本就是读、写锁的意思。加了“SHARE”锁后相当于加了读锁，表的内容就不能变化了。可为多个事务加上此锁，只要任意一个人不释放这个读锁，则其他人就不能修改这个表。加上了“EXCLUSIVE”后，则相当于加了写锁，这时别的进程既不能写也不能读这条数据。但是，后来数据库又加上了多版本的功能。有了该功能后，如果改某一行的数据，实际上并没有改原先那行数据，而是另复制出了一个新行，修改都在新行上，事务不提交，别人是看不到这条数据的。由于旧的那行数据没有变化，在修改过程中，读数据的人仍然可以读到旧的数据，这样就没有必要让别人不能读数据了。若是在多版本功能下，除了以前的“SHARE”
锁和“EXCLUSION”锁外，还需要增加两个锁，一个叫“ACCESS SHARE”，表明加上这个锁，即使正在修改数据的情况下也允许读数据，另一个锁叫作“ACCESS EXCLUSION”，意思是即使有多版本的功能，也不允许访问数据。至于“SHARE”锁和“EXCLUSIVE”锁，意思与原来差不多，就不再重复了。

表级锁加锁的对象是表，这使得加锁的范围太大，导致并发不高，于是人们提出了行级锁的概念，但行级锁与表级锁之间会产生冲突，这时就需要有一种机制来描述行级锁与表级锁之间的关系。在MySQL中是使用“意向锁”的概念来解决这个问题的，方法就是当我们要修改表中的某一行数据时，需要先在表上加一种锁，表示即将在表的部分行上加共享锁或排他锁。PostgreSQL也是这样实现的，如“ROW SHARE”和“ROWEXCLUSIVE”这两个锁，实际就对应MySQL中的共享意向锁（IS）和排它意向锁（IX）。从“意向锁”的概念出发，可以得到意向锁的下面两个特点。

> 口 意向锁之间总是不会发生冲突的，即使是“ROWEXCLUSIVE”之间也不会发生冲突，因为它们都只是“有意”要做什么，还没有真做，所以是可以兼容的。
> 
> 口 意向锁与其他非意向锁之间的关系和普通锁与普通锁之间的关系是相同的。举例：
> “X”与“X”锁是冲突的，所以“IX”锁与“X”是冲突的；“X”与“S”锁是冲突的，所以“IX”锁与“S”也是冲突的；“S”与“S”锁是不冲突的，所以“IS”
> 与“S”也是不冲突的。

如果把共享锁“SHARE”简写为“S”，排他锁“EXCLUSIVE”简写为“X”，把“ROW SHARE”简写为“IS”，把“ROWEXCLUSIVE”简写为“IX”，这四种锁之间的关系则如下表所示。
|    | X | S | IX | IS |
|----|---|---|----|----|
| X  | X | X | X  | X  |
| S  | X |   | X  |    |
| IX | X | X |    |    |
| IS | X |   |    |    |

我们知道，意向排他锁“IX”它们自己互相之间是不会冲突的，这时可能就需要一种稍严格的锁，就是这种锁自己之间也会冲突，至于和其他锁冲突的情况则与“IX”相同，这种锁在PostgreSQL中就叫“SHARE UPDATE EXCLUSIVE”。不带FULL选项的VACUUM、CREATE INDEX CONCURRENTLY命令都会请求这样的锁。这也很好理解，因为这些命令除了不允许在同一个表上并发执行本操作外，其他情况与更新表时对表加锁的需求是一样的。

在PostgreSQL中还有一种锁，称为“SHAREROWEXCLUSIVE”，这种锁可以看成是同时加了“S”锁和“IX”锁的结果，PostgreSQL命令不会自动请求这个锁模式，也就是说PostgreSQL内部目前不会使用这种锁。

总结一下，PostgreSQL中有8种表锁，最普通的是共享锁“SHARE”和排他锁“EXCLU-
SIVE”，因为多版本的原因，修改一条数据的同时，允许了读数据，所以为了处理这种情况，又加了两种锁“ACCESS SHARE”和“ACESSEXCLUSIVE”，锁中的关键字“ACCESS”
是与多版本读相关的。此外，为了处理表锁和行锁之间的关系，有了“意向锁”的概念，这时又加了两种锁，即意向共享锁和意向排他锁。由于意向锁之间不会产生冲突，而且意向排他锁互相之间也不会产生冲突，于是又需要更严格一些的锁，这样就产生了“SHARE UPDATE EXCLUSIVE”和“SHARE ROW EXCLUSIVE”这两种锁。于是，总共就有了8种锁。

在PostgreSQL中，显式在表上加锁的命令为“LOCKTABLE”命令，此命令的语法如下：

```
postgres=# \h lock
Command:     LOCK
Description: lock a table
Syntax:
LOCK [ TABLE ] [ ONLY ] name [ * ] [, ...] [ IN lockmode MODE ] [ NOWAIT ]

where lockmode is one of:

    ACCESS SHARE | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE
    | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE

```
NOWAIT：如果没有NOWAIT这个关键字时，当无法获得锁时，会一直等待，而如果加了NOWAIT关键字，在无法立即获取该锁时，此命令会立即退出并且发出一个错误信息。