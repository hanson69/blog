---
title: 【翻译】PostgreSQL中的索引-1
date: 2019-11-13 23:17:47
categories: PG基础
tags:
- 索引
- 索引扫描
- 位图扫描
- 顺序扫描
- 覆盖索引
- 表达式索引
- 部分索引
- 排序
- INDEX CONCURRENTLY
---



在本文中，我们将讨论与DBMS核心相关的常规索引引擎和各个索引访问方法之间的职责分配。



# 索引

在PostgreSQL中，索引是特殊的数据库对象，主要用于加速数据访问。它们是辅助结构：可以删除每个索引，然后从表中的信息重新创建它们。有人可能会说，DBMS可以在没有索引的情况下运行，只是会访问比较慢。但是，事实并非如此，因为索引还用于强制执行某些完整性约束。



尽管索引类型（也称为访问方法）之间存在所有差异，但它们最终都会将一个键（例如，索引列的值）与包含该键的表行相关联。每行由TID（元组ID）标识，TID由文件中的块数和该行在块内的位置组成。也就是说，使用已知键或有关它的某些信息，我们可以快速读取那些可能包含我们感兴趣的信息的行，而无需扫描整个表。

索引虽然加快了数据访问速度，但需要一定的维护成本。对于索引数据的每个操作，无论是插入、删除还是更新表行，都需要在同一事务中更新该表的索引。请注意，未建立索引的表字段的更新不会导致索引更新，这项技术称为HOT（Heap-Only Tuples）。

可扩展性：为了能够容易地添加新的访问方法到系统中，实现了通用索引引擎的接口。它的主要任务是从访问方法中获取tid并使用它们：

- 从表行的相应版本中读取数据。
- 使用预先构建的位图逐行或成批获取行版本TID。
- 根据其隔离级别，检查当前事务的行版本的可见性。

索引引擎参与执行查询，调用优化阶段创建的计划。优化器了解所有可能适用的访问方法的功能，对执行查询的不同方法进行排序和评估。该方法是否能够按所需的顺序返回数据，或者我们是否应该预期排序？我们能用这个方法搜索空值吗？这些都是优化器经常解决的问题。

除了优化器需要有关访问方法的信息，在构建索引时，系统必须决定是否可以在多个列上构建索引，以及此索引是否确保唯一性。
因此，每个访问方法都需要提供关于自身的所有必要信息。

剩下的就是访问方法的任务：

- 实现建立索引并将数据映射到页面的算法（以便缓冲区缓存管理器统一处理每个索引）。
- 通过“indexed-field operator expression”形式的谓词搜索索引中的信息。
- 评估索引使用成本。
- 正确并行处理时所需的锁。
- 生成预写日志（WAL）记录。

我们将首先考虑通用索引引擎的功能，然后继续考虑不同的访问方法。

# 索引引擎

索引引擎使PostgreSQL可以统一使用各种访问方法，但要考虑到它们的功能。

## 主要扫描技术

**译者注：**

> **访问方法（access method）**
> 
> PostgreSQL中实际用于从磁盘对象中检索数据的访问方法是堆文件的顺序扫描、索引扫描和位图索引扫描。
> 
> **顺序扫描（sequential scan**）。关系中的元组是从文件的第一块到最后一块顺序扫描。事务隔离性规则是“可见的”元组才会返回给调用者。
> 
> **索引扫描（index scan）**。给定一个搜索条件，如一个范围或等式谓词，这种访问方法会从相关的堆文件中返回匹配的元组集。该算子一次处理一个元组，开始是从索引中读取一个项，然后从堆文件中取相应的元组。在最坏情况下，这可能导致对于每个元组都要随机地取一次页面。
> 
> **位图索引扫描（bitmap index scan）**。位图索引扫描减少了在索引扫描中过多的随机取页面的风险。这通过在两个阶段中处理元组来实现。第一个阶段读取所有的索引项并在一个位图中存储堆元组的TID，第二个阶段按位图顺序去表中取匹配上的堆元组。这保证了每个堆页面只访问一次，并增加了顺序取页面的机会。另外，由多个索引建立的位图可以进行合并或求交，以便在访问堆之前计算复杂的布尔谓词。

 **注意**：
> 
> 位图扫描与位图索引不同，这两个完全不同，没有共同点。
> 
> 位图索引是一种索引类型，而位图扫描是一种扫描方法。

### 索引扫描

我们可以使用索引提供的TID进行不同的处理。让我们考虑一个例子：


```SQL
postgres=# create table t(a integer, b text, c boolean);

postgres=# insert into t(a,b,c)
  select s.id, chr((32+random()*94)::integer), random() < 0.01
  from generate_series(1,100000) as s(id)
  order by 2;

postgres=# create index on t(a);

postgres=# analyze t;
```


我们创建了一个三字段表。第一个字段包含从1到100000的数字，并且在此字段上创建索引（无论什么类型）。第二个字段包含各种ASCII字符，但不可打印的字符除外。最后，第三个字段包含一个逻辑值，该逻辑值对于大约1％的行为true，对于其余行为false。行以随机顺序插入表中。

让我们尝试根据条件“ a = 1”选择一个值。请注意，条件看起来像“ indexed-field operator expression ”，其中operator为“等于”，expression（搜索关键字）为“ 1”。在大多数情况下，条件必须看起来像这样才能使用索引。


```SQL
postgres=# explain (costs off) select * from t where a = 1;
          QUERY PLAN          
-------------------------------
 Index Scan using t_a_idx on t
   Index Cond: (a = 1)
(2 rows)
```


在这种情况下，优化器决定使用index scan。使用索引扫描时，访问方法将一个接一个地返回TID值，直到到达最后一个匹配行。索引引擎依次访问由TID指示的表行，获取行版本，针对多版本并发规则检查其可见性，并返回获得的数据。

### 位图扫描

当我们只处理几个值时，索引扫描工作正常。但是，随着检索到的行数的增加，它更有可能多次返回同一表页。此时，优化器将切换到位图扫描。


```SQL
postgres=# explain (costs off) select * from t where a <= 100;
             QUERY PLAN            
------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: (a <= 100)
   ->  Bitmap Index Scan on t_a_idx
         Index Cond: (a <= 100)
(4 rows)
```


访问方法首先返回所有符合条件的TID（Bitmap Index Scan node），然后根据这些TID构建行版本的位图。然后从表中读取行版本（Bitmap Heap Scan），每个页面只能读取一次。

注意，在第二步中，可以重新检查状况（Recheck Cond）。检索到的行数可能太大，以致行版本的位图无法完全适合RAM（受“work_mem”参数限制）。在这种情况下，位图只为包含至少一个匹配行版本的页生成。这个“lossy”位图需要较少的空间，但是在读取页面时，我们需要重新检查其中包含的每一行的条件。请注意，即使对于少量检索到的行，也因此是“精确”位图（例如在我们的示例中），“Recheck Cond”步骤仍然在计划中表示，尽管实际上没有执行。

如果对多个表字段施加条件，并且对这些字段进行了索引，则位图扫描可以同时使用多个索引（如果优化器认为这是有效的）。对于每个索引，将生成行版本的位图，然后对其执行逐位布尔乘法（如果表达式由AND连接）或布尔加法（如果表达式由or连接）。例如：


```
postgres=# create index on t(b);

postgres=# analyze t;

postgres=# explain (costs off) select * from t where a <= 100 and b = 'a';
                    QUERY PLAN                    
--------------------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: ((a <= 100) AND (b = 'a'::text))
   ->  BitmapAnd
         ->  Bitmap Index Scan on t_a_idx
               Index Cond: (a <= 100)
         ->  Bitmap Index Scan on t_b_idx
               Index Cond: (b = 'a'::text)
(7 rows)
```


在这里，BitmapAnd节点通过按位«and»操作连接两个位图。

位图扫描使我们能够避免重复访问同一数据页。但是，如果表页中的数据的物理顺序与索引记录完全相同呢？毫无疑问，我们不能完全依靠页面中数据的物理顺序。如果需要排序的数据，我们必须在查询中明确指定ORDER BY子句。但是实际上可能会对 «almost all» 数据进行排序：例如，如果按所需的顺序添加了行，并且在此之后或执行CLUSTER命令后没有更改，则可能会出现这种情况。在这种情况下，构建位图是一个过度的步骤，并且定期进行索引扫描也将同样有效（除非我们考虑了连接多个索引的可能性）。因此，在选择访问方法时，规划器会查看一个特殊的统计数据，该统计数据显示列值的物理行顺序和逻辑顺序之间的相关性：


```SQL
postgres=# select attname, correlation from pg_stats where tablename = 't';
 attname | correlation
---------+-------------
 b       |    0.533512
 c       |    0.942365
 a       | -0.00768816
(3 rows)
```


接近1的绝对值表示相关性高（如“ c”列一样），相反，接近0的绝对值表示混杂分布（“ a”列）。

**了解更多**：
[PostgreSQL bitmapAnd, bitmapOr, bitmap index scan, bitmap heap scan](https://github.com/digoal/blog/blob/master/201702/20170221_02.md)

### 顺序扫描

在非选择条件下，优化器将更倾向于使用对整个表的顺序扫描而不是使用索引：


```SQL
postgres=# explain (costs off) select * from t where a <= 40000;
       QUERY PLAN      
------------------------
 Seq Scan on t
   Filter: (a <= 40000)
(2 rows)
```


问题是条件选择性越高（与之匹配的行越少），索引工作越好。检索到的行数的增长会增加读取索引页的开销。

顺序扫描的速度快于随机扫描的速度。这尤其适用于硬盘，在硬盘中，将磁头带到磁道上的机械操作比读取数据本身要花费更多的时间。对于SSD，这种影响不太明显。可以使用两个参数来考虑访问成本的差异：“ seq_page_cost”和“ random_page_cost”，我们不仅可以全局设置，而且可以在表空间级别设置这种方式，以适应不同磁盘子系统的特性。

## 覆盖索引

通常，访问方法的主要任务是返回匹配表行的标识符，以供索引引擎从这些行中读取必需的数据。但是，如果索引已经包含查询所需的所有数据呢？这样的索引称为Covering，在这种情况下，优化器可以应用 index-only scan：


```SQL
postgres=# vacuum t;

postgres=# explain (costs off) select a from t where a < 100;
             QUERY PLAN            
------------------------------------
 Index Only Scan using t_a_idx on t
   Index Cond: (a < 100)
(2 rows)
```


此名称可能使人想到索引引擎根本不会访问表，而仅从访问方法中获取所有必要的信息。但这并非完全如此，因为PostgreSQL中的索引不存储使我们能够判断行可见性的信息。因此，访问方法将返回与搜索条件匹配的行的版本，而不管它们在当前事务中的可见性如何。

但是，如果索引引擎每次都需要查看表中的可见性，则此扫描方法与常规索引扫描没有什么不同。

为了解决该问题，PostgreSQL为表维护了一个所谓的可见性映射，在这个映射中，vacuum标记那些数据更改时间不够长的页面，使得所有事务都可以看到这些数据，而不考虑启动时间和隔离级别。如果索引返回的行的标识符与此页面相关，则可以避免可见性检查。

因此，定期vacuum可提高覆盖指数的效率。此外，优化器考虑了死元组的数量，并且如果它预测可见性检查的开销成本很高，则可以决定不使用index-only scan。

我们可以使用EXPLAIN ANALYZE命令了解对表的强制访问次数：


```SQL
postgres=# explain (analyze, costs off) select a from t where a < 100;
                                  QUERY PLAN                                  
-------------------------------------------------------------------------------
 Index Only Scan using t_a_idx on t (actual time=0.025..0.036 rows=99 loops=1)
   Index Cond: (a < 100)
   Heap Fetches: 0
 Planning time: 0.092 ms
 Execution time: 0.059 ms
(5 rows)
```


在这种情况下，不需要访问表（Heap Fetches: 0），因为刚刚完成VACUUM。通常，此数字越接近零越好。

并非所有索引都将索引值与行标识符一起存储。如果访问方法无法返回数据，则不能将其用于 index-only scans。

> PostgreSQL 11引入了一个新功能：INCLUDE-indexes。如果有一个唯一索引缺少一些列作为某个查询的覆盖索引该怎么办？不能简单地将列添加到索引，因为它将破坏其唯一性。不能简单地将列添加到索引中，因为它将破坏其唯一性。该特性允许包含不影响唯一性且不能用于搜索谓词的非键列，但仍然可以提供仅索引扫描。

## NULL

NULL表示不存在或未知值，在关系数据库中起着重要的作用。

但是要处理一个特殊的值。正则布尔代数变为三元代数；不清楚NULL是否应该小于或大于正则值（这需要特殊的排序结构，首先为NULL，最后为NULL）；不清楚聚合函数是否应该考虑NULL；需要为计划器提供特殊的统计信息…

从索引支持的角度来看，我们是否需要对这些值进行索引还不清楚。如果没有索引NULL，则索引可能更紧凑。但是，如果索引了NULL，我们可以将索引用于“ indexed-field IS [NOT] NULL”之类的条件，并且当在没有为表指定任何条件时也可作为覆盖索引（因为在这种情况下，索引必须返回所有表行的数据，包括NULL的表行）。

对于每个访问方法，开发人员都会单独决定是否索引NULL。但一般来说，它们会被索引。

## 多个字段的索引

为了支持多个字段的条件，可以使用多列索引。例如，我们可以在表的两个字段上建立索引：

postgres=# create index on t(a,b);

postgres=# analyze t;

优化器很可能更喜欢此索引而不是连接位图，因为在这里，我们无需任何辅助操作即可轻松获得所需的TID：


```SQL
postgres=# explain (costs off) select * from t where a <= 100 and b = 'a';
                   QUERY PLAN                  
------------------------------------------------
 Index Scan using t_a_b_idx on t
   Index Cond: ((a <= 100) AND (b = 'a'::text))
(2 rows)
```


从第一个字段开始，通过某些字段的条件，也可以使用多列索引来加快数据检索：


```SQL
postgres=# explain (costs off) select * from t where a <= 100;
              QUERY PLAN              
--------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: (a <= 100)
   ->  Bitmap Index Scan on t_a_b_idx
         Index Cond: (a <= 100)
(4 rows)
```


通常，如果不对第一个字段施加条件，则不会使用索引。但是有时优化器可能认为索引的使用比顺序扫描更有效。当考虑“ btree”索引时，我们将扩展这个主题。

并非所有访问方法都支持在几列上建立索引。

## 表达式索引

我们已经提到搜索条件必须看起来像“indexed-field operator expression ”。在下面的示例中，将不使用索引，因为使用的是包含字段名称的表达式而不是字段名称本身：


```SQL
postgres=# explain (costs off) select * from t where lower(b) = 'a';
                QUERY PLAN                
------------------------------------------
 Seq Scan on t
   Filter: (lower((b)::text) = 'a'::text)
(2 rows)
```


重写此特定查询不需要太多时间，只需将字段名写入运算符的左侧。但是，如果这不可能，则表达式的索引（函数索引）将有所帮助：


```SQL
postgres=# create index on t(lower(b));

postgres=# analyze t;

postgres=# explain (costs off) select * from t where lower(b) = 'a';
                     QUERY PLAN                    
----------------------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: (lower((b)::text) = 'a'::text)
   ->  Bitmap Index Scan on t_lower_idx
         Index Cond: (lower((b)::text) = 'a'::text)
(4 rows)
```



函数索引不是建立在表字段上，而是建立在任意表达式上。优化器将为“indexed-expression operator expression”等条件考虑此索引。如果要索引的表达式的计算是一个代价高昂的操作，那么索引的更新也将需要相当多的计算资源。
请记住，索引表达式会收集一个单独的统计数据。我们可以通过索引名在«pg_stats»视图中了解此统计信息：


```SQL
postgres=# \d t
       Table "public.t"
 Column |  Type   | Modifiers
--------+---------+-----------
 a      | integer |
 b      | text    |
 c      | boolean |
Indexes:
    "t_a_b_idx" btree (a, b)
    "t_a_idx" btree (a)
    "t_b_idx" btree (b)
    "t_lower_idx" btree (lower(b))
postgres=# select * from pg_stats where tablename = 't_lower_idx';
```


如有必要，可以用与常规数据字段相同的方式控制直方图的数量（请注意，列名可能因索引表达式而异）：


```SQL
postgres=# \d t_lower_idx
 Index "public.t_lower_idx"
 Column | Type | Definition
--------+------+------------
 lower  | text | lower(b)
btree, for table "public.t"
postgres=# alter index t_lower_idx alter column "lower" set statistics 69;
```


> PostgreSQL 11引入了一种更简洁的方法，通过在ALTER INDEX…SET statistics命令中指定列号来控制索引的统计目标。

## 部分索引

有时需要只需索引部分表行，这通常与极度不均匀的分布有关：通过索引搜索不频繁的值很有意义，但是通过对表进行全面扫描可以更容易地找到频繁的值。

我们当然可以在«c»列上建立一个常规索引，该索引将按照我们期望的方式工作：


```SQL
postgres=# create index on t(c);

postgres=# analyze t;

postgres=# explain (costs off) select * from t where c;
          QUERY PLAN          
-------------------------------
 Index Scan using t_c_idx on t
   Index Cond: (c = true)
   Filter: c
(3 rows)

postgres=# explain (costs off) select * from t where not c;
    QUERY PLAN    
-------------------
 Seq Scan on t
   Filter: (NOT c)
(2 rows)
```


索引大小为276页：


```SQL
postgres=# select relpages from pg_class where relname='t_c_idx';
 relpages
----------
      276
(1 row)
```


但是由于«c»列仅对1％的行具有true值，因此实际上从未使用过99％的索引。在这种情况下，我们可以建立部分索引：


```SQL
postgres=# create index on t(c) where c;

postgres=# analyze t;

索引的大小减少到5页：

postgres=# select relpages from pg_class where relname='t_c_idx1';
 relpages
----------
        5
(1 row)
```


有时，大小和性能上的差异可能非常明显。

## 排序

如果访问方法以某些特定顺序返回行标识符，这将为优化器提供执行查询的其他选项。

我们可以扫描表，然后对数据进行排序：


```SQL
postgres=# set enable_indexscan=off;

postgres=# explain (costs off) select * from t order by a;
     QUERY PLAN      
---------------------
 Sort
   Sort Key: a
   ->  Seq Scan on t
(3 rows)
```


但是我们可以按照期望的顺序使用索引轻松读取数据：


```SQL
postgres=# set enable_indexscan=on;

postgres=# explain (costs off) select * from t order by a;
          QUERY PLAN          
-------------------------------
 Index Scan using t_a_idx on t
(1 row)
```


所有访问方法中只有«btree»可以返回排序的数据，因此让我们开始更详细的讨论，直到考虑这种类型的索引。

## CREATE INDEX CONCURRENTLY

通常，建立索引需要为表获取SHARE锁。该锁允许从表中读取数据，但禁止在建立索引时进行任何更改。

我们可以确定这一点，例如，如果在«t»表上建立索引期间，我们在另一个会话中执行以下查询：


```SQL
postgres=# select mode, granted from pg_locks where relation = 't'::regclass;
   mode    | granted
-----------+---------
 ShareLock | t
(1 row)
```


如果表足够大并且广泛用于插入，更新或删除，则这似乎是不可接受的，因为修改过程将长时间等待锁定释放。

在这种情况下，我们可以使用并发建立索引。


```SQL
postgres=# create index concurrently on t(a);
```


此命令将表锁定为SHARE UPDATE EXCLUSIVE模式，该模式允许读取和更新（禁止更改表结构，禁止concurrent vacuuming, analysis, 或在该表上构建另一个索引）。

但是，也有另一面。首先，索引的建立将比平时慢，因为完成了两次遍历表而不是一次遍历，并且还必须等待修改数据的并行事务完成。

其次，在同时建立索引的情况下，可能会发生死锁或违反唯一约束。但是这样还会建立索引，尽管该索引无法运行。我们必须要删除并重建这样的索引。无效的索引可通过 \d 元命令查看到，输出中用INVALID单词进行标记，下面的查询可以返回所有的无效索引：


```SQL
postgres=# select indexrelid::regclass index_name, indrelid::regclass table_name
from pg_index where not indisvalid;
 index_name | table_name
------------+------------
 t_a_idx    | t
(1 row)
```


# 英文原文：
https://habr.com/en/company/postgrespro/blog/441962/