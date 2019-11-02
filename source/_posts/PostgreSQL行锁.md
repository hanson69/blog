---
title: PostgreSQL行锁
date: 2019-11-02 23:43:27
categories: PG基础
tags:
- 行锁
- pgrowlocks
---
在PostgreSQL数据库中要查询、插入、更新、删除表中的数据时，首先是要获得表上的锁，然后再获得行上的锁。

## 4个行锁

|       锁类型      |                                        使用场景                                       |
|:-----------------:|-------------------------------------------------------------------------------------|
|     FOR UPDATE    | UPDATE,  DELETE, SELECT FOR UPDATE 等操作将会获取此锁                                 |
| FOR NO KEY UPDATE | 对非键值更新将会获取此锁。                                                            |
|     FOR SHARE     | 阻塞其他事务在这些行上执行UPDATE，DELETE，SELECT FOR UPDATE或SELECT FOR NO KEY UPDATE |
|   FOR KEY SHARE   | 阻塞其他事务对键值进行删除更新                                                        |

## 使用场景举例

```SQL
postgres=# \d tb1
                Table "public.tb1"
 Column |  Type   | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 id     | integer |           | not null |
 name   | text    |           |          |
Indexes:
    "tb1_pkey" PRIMARY KEY, btree (id)

-- session A
postgres=# begin ;
BEGIN

postgres=# select * from tb1 where id =1 for SHARE;
 id | name
----+------
  1 | a
(1 row)

-- session B
postgres=# select * from pgrowlocks('tb1');
 locked_row | locker | multi | xids  |     modes     |  pids
------------+--------+-------+-------+---------------+---------
 (0,1)      |    605 | f     | {605} | {"For Share"} | {56422}
(1 row)

-- session A
-- 对非键值更新将获得 No Key Update 锁
postgres=# update tb1 set name = 'aa' where id = 1;
UPDATE 1

-- session B
postgres=# select * from pgrowlocks('tb1');
 locked_row | locker | multi | xids  |       modes       |  pids
------------+--------+-------+-------+-------------------+---------
 (0,1)      |    605 | f     | {605} | {"No Key Update"} | {56422}
(1 row)

-- session A 
-- 对键值更新将获得 Update 锁
postgres=# update tb1 set id = id+2 where id = 2;
UPDATE 1

-- session B
postgres=# select * from pgrowlocks('tb1');
 locked_row | locker | multi | xids  |       modes       |  pids
------------+--------+-------+-------+-------------------+---------
 (0,1)      |    605 | f     | {605} | {"No Key Update"} | {56422}
 (0,3)      |    605 | f     | {605} | {Update}          | {56422}
(2 rows)

```
#### **注意：**
> 1、在PostgreSQL中，由于有多版本的实现，所以实际读取行数据时，并不会在行上执行任何锁。
> 
> 2、PostgreSQL不会将已获得的行锁信息存储在内存中, 行锁信息存储在行的头部信息infomask中，换句话锁, 获得行锁时需要修改t_infomask。



## 行锁冲突矩阵
|                   | FOR KEY SHARE | FOR SHARE | FOR NO KEY UPDATE | FOR UPDATE |
|-------------------|---------------|-----------|-------------------|------------|
| FOR KEY SHARE     |               |           |                   | X          |
| FOR SHARE         |               |           | X                 | X          |
| FOR NO KEY UPDATE |               | X         | X                 | X          |
| FOR UPDATE        | X             | X         | X                 | X          |

## 行锁查看
一般在pg_locks中无法查看到行锁的信息。

如果在pg_locks中查看到了行锁信息, 并不代表行锁信息存储在内存中. 这种情况往往出现在行锁等待时。

如果需要查看行锁可以使用扩展模块pgrowlocks。 使用pgrowlocks模块查看行锁须知 :

1. 使用pgrowlocks查看行锁实际上是解析tuple的t_infomask信息, 所以表要能正常查询, 如果加了ACCESS Exclusive Lock, 那么正常的查询需要等待。

```
-- session A
postgres=# begin;
BEGIN

postgres=# lock table tb1 in access exclusive mode;
LOCK TABLE

-- session B
postgres=# select * from pgrowlocks('tb1');
-- 会话 B 处于等待状态.

```


2. pgrowlocks不提供一致性数据, 例如tuple a locked, 但是在查看到lock b的时候, tuple a 可能已经unlocked了， 也就是说在查询过程中行锁随时都可以发生变化。


