---
title: PG事务隔离级别
date: 2019-10-20 23:51:24
categories: PG基础
tags:
- 事务隔离级别
- MVCC
---


- PostgreSQL 更新和删除数据时, 并不是直接删除行的数据, 而是更新行的头部信息中的xmax和infomask掩码。事务提交后更新当前数据库集群的事务状态和pg_clog中的事务提交状态。

- PostgreSQL多版本并发控制不需要UNDO表空间。

- PostgreSQL的repeatable read隔离级别不会产生幻像读

## READ UNCOMMITTED 测试

```SQL
-- session A
postgres=# begin isolation level READ UNCOMMITTED;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,3) |  498 |    0 |  1 | A
(2 行记录)

postgres=# update tb1 set name='a' where id=1;
UPDATE 1

postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |    0 |  1 | a
(2 行记录)


-- session B
-- 没有读出session A 的更改
postgres=# begin isolation level READ UNCOMMITTED;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,3) |  498 |    0 |  1 | A
(2 行记录)

-- session A
postgres=# commit;

-- session B
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |    0 |  1 | a
(2 行记录)

```
从上面测试可以看出，PostgreSQL不支持read uncommitted事务隔离级别。


## READ COMMITTED 测试
```sql

postgres=# create table tb1(id int, name text);
CREATE TABLE
postgres=# insert into tb1 values(1, 'a');
postgres=# insert into tb1 values(2, 'b');


-- session A
postgres=# begin;
BEGIN
-- a, b分别是事务id 495和497创建
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,1) |  495 |    0 |  1 | a
 (0,2) |  497 |    0 |  2 | b
(2 行记录)


-- session B
postgres=# begin;
BEGIN

-- 会话B创建事务块，id为498
postgres=# select  txid_current();
 txid_current
--------------
          498
(1 行记录)

-- 更新id=1的name='A'
postgres=# update tb1 set name='A' where id=1;
UPDATE 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,3) |  498 |    0 |  1 | A
(2 行记录)

-- session A
-- 会话A看不到事务B的更改（RC），但是id=1这一行xmax已经变为会话A的事务id，说明改行记录已经被事务号为498更改
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,1) |  495 |  498 |  1 | a
 (0,2) |  497 |    0 |  2 | b
(2 行记录)

-- 事务B提交
-- session B
postgres=# commit;
COMMIT

-- 事务A能够看到事务B提交后的更改数据，并且改行记录的xmax为0
session A
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,3) |  498 |    0 |  1 | A
(2 行记录)


postgres=# select * from heap_page_items(get_raw_page('tb1', 0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    495 |    498 |        0 | (0,3)  |       16386 |       1282 |     24 |        |       | \x010000000561
  2 |   8128 |        1 |     30 |    497 |      0 |        0 | (0,2)  |           2 |       2306 |     24 |        |       | \x020000000562
  3 |   8096 |        1 |     30 |    498 |      0 |        0 | (0,3)  |       32770 |      10498 |     24 |        |       | \x010000000541
(3 行记录)

```
PostgreSQL 更新和删除数据时, 并不是直接删除行的数据, 而是更新行的头部信息中的xmax和infomask掩码。

## REPEATABLE READ 测试
```SQL
-- session A
postgres=# begin isolation level REPEATABLE READ;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |    0 |  1 | a
(2 行记录)

-- session B
-- session B 修改数据, 并提交
postgres=# begin;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |    0 |  1 | a
(2 行记录)

postgres=# update tb1 set name='A' where id=1;
UPDATE 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,5) |  501 |    0 |  1 | A
(2 行记录)

postgres=# commit;
COMMIT

-- session A
-- 未出现不可重复读现象
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |  501 |  1 | a
(2 行记录)

-- session C 新增数据并提交
postgres=# begin;
BEGIN
postgres=# insert into tb1 values(3, 'c');
INSERT 0 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,5) |  501 |    0 |  1 | A
 (0,6) |  501 |    0 |  3 | c
(3 行记录)

postgres=# commit;
COMMIT

-- session A
-- 未出现幻象读
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,4) |  500 |  501 |  1 | a
(2 行记录)

```


## PostgresSQL 幻读情景
PostgresSQL 的 REPEATABLE READ 不会出现幻读，可以通过 READ COMMITTED 测试幻读

```
-- session A
postgres=# begin ISOLATION LEVEL READ COMMITTED;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,5) |  501 |    0 |  1 | A
 (0,6) |  501 |    0 |  3 | c
(3 行记录)

-- session B 删除一条记录并提交
postgres=# begin;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,5) |  501 |    0 |  1 | A
 (0,6) |  501 |    0 |  3 | c
(3 行记录)


postgres=# delete from tb1 where id=1;
DELETE 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
(2 行记录)

postgres=# commit;

-- session A 幻读（刚刚读的时候3条记录，现在是2条记录）
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
(2 行记录)


-- session C 插入一条记录
postgres=# begin;
BEGIN
postgres=# insert into tb1 values(4, 'c');
INSERT 0 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |    0 |  4 | c
(3 行记录)

postgres=# commit;


-- session A 再次幻读（刚刚读取的记录中没有id=4的记录）
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |    0 |  4 | c
(3 行记录)

```

## REPEATABLE READ 异常测试

```sql
-- session A
postgres=# begin ISOLATION LEVEL REPEATABLE READ;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |    0 |  4 | c
(3 行记录)

-- session B 更新或者删除id=4记录, 并提交.
postgres=# begin;
BEGIN
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |    0 |  4 | c
(3 行记录)

postgres=# delete from tb1 where id=4;
DELETE 1
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
(2 行记录)

postgres=# commit;

-- session A 更新或者删除id=4这条记录. 会报错回滚
postgres=# select ctid, xmin, xmax, * from tb1;
 ctid  | xmin | xmax | id | name
-------+------+------+----+------
 (0,2) |  497 |    0 |  2 | b
 (0,6) |  501 |    0 |  3 | c
 (0,7) |  503 |  504 |  4 | c
(3 行记录)

postgres=# update tb1 set name='d' where id=4;
ERROR:  could not serialize access due to concurrent delete


```


## SERIALIZABLE 测试
```
-- session A
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
          24300
(1 行记录)

postgres=# truncate tb1;
TRUNCATE TABLE
postgres=# insert into tb1 select generate_series(1,100000);
INSERT 0 100000
postgres=#  begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=# select sum(id) from tb1 where id=100;
 sum
-----
 100
(1 行记录)


-- session B
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
          10456
(1 行记录)


-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456);
 relation |  locktype  |  pid  |      mode       | granted
----------+------------+-------+-----------------+---------
 tb1      | relation   | 24300 | AccessShareLock | t
          | virtualxid | 24300 | ExclusiveLock   | t
 tb1      | relation   | 24300 | SIReadLock      | t
(3 行记录)


-- session B
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=# select sum(id) from tb1 where id=10;
 sum
-----
  10
(1 行记录)


-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456);
 relation |  locktype  |  pid  |      mode       | granted
----------+------------+-------+-----------------+---------
 tb1      | relation   | 10456 | AccessShareLock | t
          | virtualxid | 10456 | ExclusiveLock   | t
 tb1      | relation   | 24300 | AccessShareLock | t
          | virtualxid | 24300 | ExclusiveLock   | t
 tb1      | relation   | 10456 | SIReadLock      | t
 tb1      | relation   | 24300 | SIReadLock      | t
(6 行记录)


-- session A
postgres=# insert into tb1 values(1, 'a');
INSERT 0 1

-- session B
postgres=# insert into tb1 values(2, 'b');
INSERT 0 1

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |   locktype    |  pid  |       mode       | granted
----------+---------------+-------+------------------+---------
 tb1      | relation      | 10456 | RowExclusiveLock | t
          | virtualxid    | 10456 | ExclusiveLock    | t
          | transactionid | 10456 | ExclusiveLock    | t
 tb1      | relation      | 10456 | SIReadLock       | t
 tb1      | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 24300 | SIReadLock       | t
 tb1      | relation      | 24300 | AccessShareLock  | t
 tb1      | relation      | 24300 | RowExclusiveLock | t
          | virtualxid    | 24300 | ExclusiveLock    | t
          | transactionid | 24300 | ExclusiveLock    | t
(10 行记录)

-- session A
postgres=# commit;
COMMIT

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |   locktype    |  pid  |       mode       | granted
----------+---------------+-------+------------------+---------
 tb1      | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 10456 | RowExclusiveLock | t
          | virtualxid    | 10456 | ExclusiveLock    | t
          | transactionid | 10456 | ExclusiveLock    | t
 tb1      | relation      | 10456 | SIReadLock       | t
 tb1      | relation      | 24300 | SIReadLock       | t
(6 行记录)

-- session B
postgres=# commit;
ERROR:  could not serialize access due to read/write dependencies among transactions
描述:  Reason code: Canceled on identification as a pivot, during commit attempt.
提示:  The transaction might succeed if retried.

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation | locktype | pid | mode | granted
----------+----------+-----+------+---------
(0 行记录)


```


## 加索引测试上面的场景
```
-- session A
postgres=# create index idx_tb1 on tb1(id);
CREATE INDEX
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN

-- session C
-- 这里变成了行锁和页锁
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |  locktype  |  pid  |      mode       | granted
----------+------------+-------+-----------------+---------
 idx_tb1  | relation   | 24300 | AccessShareLock | t
 tb1      | relation   | 24300 | AccessShareLock | t
          | virtualxid | 24300 | ExclusiveLock   | t
 idx_tb1  | page       | 24300 | SIReadLock      | t
 tb1      | tuple      | 24300 | SIReadLock      | t
(5 行记录)

-- session B
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=#  select sum(id) from tb1 where id=10;
 sum
-----
  10
(1 行记录)

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |  locktype  |  pid  |      mode       | granted
----------+------------+-------+-----------------+---------
 idx_tb1  | relation   | 10456 | AccessShareLock | t
 tb1      | relation   | 10456 | AccessShareLock | t
          | virtualxid | 10456 | ExclusiveLock   | t
 idx_tb1  | page       | 10456 | SIReadLock      | t
 tb1      | tuple      | 10456 | SIReadLock      | t
          | virtualxid | 24300 | ExclusiveLock   | t
 tb1      | tuple      | 24300 | SIReadLock      | t
 idx_tb1  | page       | 24300 | SIReadLock      | t
 idx_tb1  | relation   | 24300 | AccessShareLock | t
 tb1      | relation   | 24300 | AccessShareLock | t
(10 行记录)


-- session A
postgres=# insert into tb1 values(1, 'a');
INSERT 0 1

-- session C
-- 事务 24300 获得 行互斥锁
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |   locktype    |  pid  |       mode       | granted
----------+---------------+-------+------------------+---------
 idx_tb1  | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 10456 | AccessShareLock  | t
          | virtualxid    | 10456 | ExclusiveLock    | t
 idx_tb1  | page          | 10456 | SIReadLock       | t
 tb1      | tuple         | 10456 | SIReadLock       | t
 tb1      | relation      | 24300 | RowExclusiveLock | t
          | virtualxid    | 24300 | ExclusiveLock    | t
          | transactionid | 24300 | ExclusiveLock    | t
 tb1      | tuple         | 24300 | SIReadLock       | t
 idx_tb1  | page          | 24300 | SIReadLock       | t
 idx_tb1  | relation      | 24300 | AccessShareLock  | t
 tb1      | relation      | 24300 | AccessShareLock  | t
(12 行记录)

-- session B
postgres=# insert into tb1 values(2, 'b');
INSERT 0 1


-- session C
-- -- 事务 10456 获得 行互斥锁
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation |   locktype    |  pid  |       mode       | granted
----------+---------------+-------+------------------+---------
 idx_tb1  | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 10456 | AccessShareLock  | t
 tb1      | relation      | 10456 | RowExclusiveLock | t
          | virtualxid    | 10456 | ExclusiveLock    | t
          | transactionid | 10456 | ExclusiveLock    | t
 idx_tb1  | page          | 10456 | SIReadLock       | t
 tb1      | tuple         | 10456 | SIReadLock       | t
          | virtualxid    | 24300 | ExclusiveLock    | t
 idx_tb1  | page          | 24300 | SIReadLock       | t
          | transactionid | 24300 | ExclusiveLock    | t
 tb1      | tuple         | 24300 | SIReadLock       | t
 idx_tb1  | relation      | 24300 | AccessShareLock  | t
 tb1      | relation      | 24300 | AccessShareLock  | t
 tb1      | relation      | 24300 | RowExclusiveLock | t
(14 行记录)

-- session A
postgres=# commit;
COMMIT

-- session B
-- 索引页用了同一个, 并且被插入语句更新了. 所以发生了冲突
postgres=# commit;
ERROR:  could not serialize access due to read/write dependencies among transactions
描述:  Reason code: Canceled on identification as a pivot, during commit attempt.
提示:  The transaction might succeed if retried.

-- session C
postgres=# select relation::regclass, locktype, pid, mode, granted from pg_locks where pid in (24300, 10456) order by pid;
 relation | locktype | pid | mode | granted
----------+----------+-----+------+---------
(0 行记录)


-- 如果其中一个插入的值不在1号索引页则没有问题, 例如
-- session A
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=# select sum(id) from tb1 where id=100;
 sum
-----
 100
(1 行记录)


-- session B
postgres=# begin ISOLATION LEVEL SERIALIZABLE;
BEGIN
postgres=# select sum(id) from tb1 where id=10;
 sum
-----
  10
(1 行记录)


-- session A
postgres=# insert into tb1 values(1, 'a');
INSERT 0 1
postgres=# commit;
COMMIT

-- session B
postgres=# insert into tb1 values(200000, 'c');
INSERT 0 1
postgres=# commit;
COMMIT

```
> 注意事项:
>
> PostgreSQL 的 hot_standby节点不支持串行事务隔离级别, 只能支持read committed和repeatable read隔离级别



## PostgreSQL多版本并发控制

```
? RR1 tuple-v1 IDLE IN TRANSACTION;
? RC1 tuple-v1 IDLE IN TRANSACTION;
? RC2 tuple-v1 UPDATE -> tuple-v2 COMMIT;
? RR1 tuple-v1 IDLE IN TRANSACTION;
? RC1 tuple-v2 IDLE IN TRANSACTION;
? RR2 tuple-v2 IDLE IN TRANSACTION;
? RC3 tuple-v2 UPDATE -> tuple-v3 COMMIT;
? RR1 tuple-v1 IDLE IN TRANSACTION;
? RR2 tuple-v2 IDLE IN TRANSACTION;
? RC1 tuple-v3 IDLE IN TRANSACTION;
```

PostgreSQL多版本并发控制不需要UNDO表空间。