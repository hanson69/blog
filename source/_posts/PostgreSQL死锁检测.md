---
title: PostgreSQL死锁检测
date: 2019-11-01 19:46:57
categories: PG基础
tags:
- 死锁
- 检测
- 回滚
---

死锁是指两个或两个以上的事务在执行过程中相互持有对方期待的锁，若没有其他机制，它们都将无法进行下去。
例如，事务1在表A上持有一个排斥锁，同时试图请求一个在表B上的排斥锁，而事务2已经持有表B的排斥锁，同时却在请求表A上的一个排斥锁，那么两个事务就都不能执行了。PostgreSQL能够自动侦测到死锁，然后会退出其中一个事务，从而允许其他事务完成。

**例子**
```
-- 会话 1

postgres=# create table d_lock(id int, info text);  
CREATE TABLE  
postgres=# insert into d_lock values (1,'test');  
INSERT 0 1  
postgres=# insert into d_lock values (2,'test');  
INSERT 0 1  
  
postgres=# begin;  
BEGIN  
postgres=# update d_lock set info='a' where id=1;  
UPDATE 1  

-- 会话 2
postgres=# begin;  
BEGIN  
postgres=# update d_lock set info='b' where id=2;  
UPDATE 1  
postgres=# update d_lock set info='b' where id=1;  
等待  

-- 会话 1
postgres=# update d_lock set info='a' where id=2;  -- 等待，检测到死锁，自动回滚  
ERROR:  deadlock detected  
DETAIL:  Process 13602 waits for ShareLock on transaction 96548629; blocked by process 18060.  
Process 18060 waits for ShareLock on transaction 96548628; blocked by process 13602.  
HINT:  See server log for query details.  
CONTEXT:  while updating tuple (0,2) in relation "d_lock"  

-- 会话 2
会话 1 自动释放锁后，会话2更新成功  
UPDATE 1  

```


死锁的发生必须具备以下四个必要条件。

> 1.、互斥条件：指事务对所分配到的资源加了排他锁，即在一段时间内只能由一个事务加锁占用。如果此时还有其他进程请求排他锁，则请求者只能等待，直至持有排他锁的事务释放为止。
> 
> 2、请求和保持条件：指事务已经至少持有了一把排他锁，但又提出了新的排他锁请求，而该资源上的排他锁已被其他事务占有，此时请求被阻塞，但同时它对自己已获得的排他锁又保持不放。
> 
> 3、不剥夺条件：指事务已获得的锁，在未使用完之前，不能被其他进程剥夺，只能在使用完时由自己释放。
> 
> 4、环路等待条件：指在发生死锁时，必然存在一个事务-资源的环形链，即事务集合
> {T0，Tl，T2，…，Tn}中的T0正在等待一个T1持有的排他锁；P1正在等待P2持有的排他锁，……，Pn正在等待P0持有的排他锁。


理解了死锁的原因，尤其是产生死锁的四个必要条件，就可以最大可能地避免、预防和解除死锁。

防止死锁的最好方法通常是保证所有使用一个数据库的应用都以相同的顺序在多个对象上请求排他锁。比如，在应用编程中，人为规定在一个事务中只能以一个固定顺序来更新表。假设在数据库中，有A、B、C、D四张表，现规定只能按如下顺序修改这几张表：“B→C→A→D”，若某个进程先更新了A表，然后又想更新C表，则必须先回滚先前对A表的更新，然后再按规定的顺序，先更新C表，再更新A表。
由于数据库可以自动检测出死锁，所以应用也可以通过捕获死锁异常来处理死锁。但这不是一个很好的方法，因为数据库检测死锁需要一定的代价，可能会导致应用程序过久地持有排他锁，从而致使系统的并发处理能力下降。
排他锁持有的时间越长，就越容易导致死锁，所以在程序设计时要尽量短地持有排他锁。

在PostgreSQL中，事务自己的锁是从不冲突的，因此一个事务可以在持有“SHARE”
模式的锁时再请求“ROWEXCLUSIVE”锁，因为这样并不会阻塞自己。
当事务要更新表中的数据时，应该申请“ROWEXCLUSIVE”锁，而不应申请“SHARE”
锁，因为事务在更新数据时，还是会对表加“ROWEXCLUSIVE”锁。想象一下，当两个并发的事务都请求“SHARE”锁后，开始更新数据前，对表要加“ROWEXCLUSIVE”锁，但由于各自先前已加了“SHARE”锁，所以都要等待对方释放“SHARE”锁，从而出现死锁。

```SQL
postgres=# create table tb1 (id int primary key, name text);
postgres=# insert into tb1 values (1, 'a');
postgres=# insert into tb1 values (1, 'b');

-- session A
postgres=# begin;
BEGIN

postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
          53922
(1 row)



-- session B
postgres=# begin ;
BEGIN
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
          54126
(1 row)

-- session A
postgres=# select * from tb1 for share;
 id | name
----+------
  1 | a
  2 | b
(2 rows)

--session C
postgres=# select relation::regclass, locktype, mode, granted, pid from pg_locks where pid in (53922, 54126);
 relation |   locktype    |      mode       | granted |  pid
----------+---------------+-----------------+---------+-------
 tb1_pkey | relation      | AccessShareLock | t       | 53922
 tb1      | relation      | RowShareLock    | t       | 53922
          | virtualxid    | ExclusiveLock   | t       | 53922
          | transactionid | ExclusiveLock   | t       | 53922
(4 rows)


--session B
postgres=# select * from tb1 for share;
 id | name
----+------
  1 | a
  2 | b
(2 rows)

--session C
postgres=# select relation::regclass, locktype, mode, granted, pid from pg_locks where pid in (53922, 54126) order by 5;
 relation |   locktype    |      mode       | granted |  pid
----------+---------------+-----------------+---------+-------
          | virtualxid    | ExclusiveLock   | t       | 53922
 tb1      | relation      | RowShareLock    | t       | 53922
          | transactionid | ExclusiveLock   | t       | 53922
 tb1_pkey | relation      | AccessShareLock | t       | 53922
          | transactionid | ExclusiveLock   | t       | 54126
 tb1_pkey | relation      | AccessShareLock | t       | 54126
 tb1      | relation      | RowShareLock    | t       | 54126
          | virtualxid    | ExclusiveLock   | t       | 54126
(8 rows)


-- session A
postgres=# update tb1 set name = 'aa' where id = 1;
-- 等待锁

--session C
postgres=# select relation::regclass, locktype, mode, granted, pid from pg_locks where pid in (53922, 54126) order by 5;
 relation |   locktype    |       mode       | granted |  pid
----------+---------------+------------------+---------+-------
          | transactionid | ShareLock        | f       | 53922
 tb1      | tuple         | ExclusiveLock    | t       | 53922
          | transactionid | ExclusiveLock    | t       | 53922
 tb1_pkey | relation      | AccessShareLock  | t       | 53922
          | virtualxid    | ExclusiveLock    | t       | 53922
 tb1_pkey | relation      | RowExclusiveLock | t       | 53922
 tb1      | relation      | RowShareLock     | t       | 53922
 tb1      | relation      | RowExclusiveLock | t       | 53922
          | transactionid | ExclusiveLock    | t       | 54126
 tb1_pkey | relation      | AccessShareLock  | t       | 54126
 tb1      | relation      | RowShareLock     | t       | 54126
          | virtualxid    | ExclusiveLock    | t       | 54126
(12 rows)


-- session B 
-- 注意到在PostgreSQL中，整个session B 回滚了。
postgres=# update tb1 set name = 'aa' where id = 1;
ERROR:  deadlock detected
DETAIL:  Process 54126 waits for ExclusiveLock on tuple (0,1) of relation 16417 of database 13164; blocked by process 53922.
Process 53922 waits for ShareLock on transaction 593; blocked by process 54126.
HINT:  See server log for query details.

-- session A
postgres=# update tb1 set name = 'aa' where id = 1;
-- 更新完成
```



从这个例子可以看出，如果涉及多种锁模式，那么事务应该总是最先请求最严格的锁模式，否则就容易出现死锁。

1. PostgreSQL死锁被检测到的属于哪个SESSION？从实验来看应该是死锁发生时的最后一个事务。
> 
>  死锁检测算法介绍：
> src/backend/storage/lmgr/README

2. 死锁被检测到之后，PostgreSQL不允许事务中的部分SQL语句执行成功，要么全部成功，要么全部失败。这个和psql的默认配置有关 ：

```
[hanson@postgres ~]$ man psql
……
ON_ERROR_ROLLBACK
  When  on,  if a statement in a transaction block generates an error, the error is ignored and the transaction contin-
  ues. When interactive, such errors are only ignored in interactive sessions, and not when reading script files.  When
  off  (the  default),  a  statement  in a transaction block that generates an error aborts the entire transaction. The
  on_error_rollback-on mode works by issuing an implicit SAVEPOINT for you, just before  each  command  that  is  in  a
  transaction block, and rolls back to the savepoint on error.
……

```

如果开启 ON_ERROR_ROLLBACK, 会在每一句SQL前设置隐形的savepoint, 可以继续下面的SQL, 而不用全部回滚, 如下 :
```
postgres=# \set ON_ERROR_ROLLBACK on  
postgres=# begin;  
BEGIN  
postgres=# insert into t values (1);  
ERROR:  relation "t" does not exist  
LINE 1: insert into t values (1);  
                    ^  
-- 还可以继续
postgres=# \dt  
No relations found.  
postgres=# create table t (id int);  
CREATE TABLE  
postgres=# insert into t values (1);  
INSERT 0 1  
postgres=# insert into t values ('a');  
ERROR:  invalid input syntax for integer: "a"  
LINE 1: insert into t values ('a');  

-- 提交成功                              ^  
postgres=# commit;  
COMMIT  

-- 插入的 1 没有被回滚
postgres=# select * from t;  
 id   
----  
  1  
(1 row)  
  
postgres=# \set ON_ERROR_ROLLBACK off  
postgres=# begin;  
BEGIN  
postgres=# insert into t values (1);  
INSERT 0 1  
postgres=# insert into t values ('a');  
ERROR:  invalid input syntax for integer: "a"  
LINE 1: insert into t values ('a');  
                              ^  
-- 回滚                               
postgres=# commit;  
ROLLBACK  

-- 新插入的 1 被回滚 
postgres=# select * from t;  
 id   
----  
  1  
(1 row)  
```



### 死锁检测间隔配置

```
-- 默认为 1 秒
postgres=# show deadlock_timeout ;  
 deadlock_timeout   
------------------  
 1s  
(1 row)
```


## 参考：
《PostgreSQL修炼之道从小工到专家》

[https://github.com/digoal/blog/blob/0ef02248fe7419c55a98a425feefd2421ad25537/201104/20110408_01.md](https://github.com/digoal/blog/blob/0ef02248fe7419c55a98a425feefd2421ad25537/201104/20110408_01.md)



