---
title: pg_stat_statements分析1——使用介绍
date: 2020-04-25 23:47:27
categories: PG基础
tags:
- 慢查询
- 统计
- 性能跟踪
---

# 简介
pg_stat_statements用于跟踪性能问题，比慢查询日志更加实用，因为慢查询日志只会指向单个慢查询，不会向我们显示由大量中等查询引起的问题。因此，建议始终打开此模块。pg_stat_statements开销很小，绝不会破坏系统的整体性能。

pg_stat_statements插件必须通过在postgresql.conf的shared_preload_libraries中增加pg_stat_statements来载入，然后重启PG。

pg_stat_statements插件还提供了一个视图 pg_stat_statements插件 用于查看
插件收集的统计信息，该视图不是全局可用的，需要通过CREATE EXTENSION
pg_stat_statements 为启用。

# pg_stat_statements视图
```
postgres=# \d pg_stat_statements
                    View "public.pg_stat_statements"
       Column        |       Type       | Collation |
---------------------+------------------+-----------+
 userid              | oid              |           |
 dbid                | oid              |           |
 queryid             | bigint           |           |
 query               | text             |           |
 calls               | bigint           |           |
 total_time          | double precision |           |
 min_time            | double precision |           |
 max_time            | double precision |           |
 mean_time           | double precision |           |
 stddev_time         | double precision |           |
 rows                | bigint           |           |
 shared_blks_hit     | bigint           |           |
 shared_blks_read    | bigint           |           |
 shared_blks_dirtied | bigint           |           |
 shared_blks_written | bigint           |           |
 local_blks_hit      | bigint           |           |
 local_blks_read     | bigint           |           |
 local_blks_dirtied  | bigint           |           |
 local_blks_written  | bigint           |           |
 temp_blks_read      | bigint           |           |
 temp_blks_written   | bigint           |           |
 blk_read_time       | double precision |           |
 blk_write_time      | double precision |           |
```
pg_stat_statements将查询和常数值分开，常数值会被忽略，在pg_stat_statements显示中它会被一个参数符号，比如$1所替换，这样就可以聚合只使用不同参数的相同查询。SELECT ... FROM x WHERE y = 10将变成SELECT ... FROM x WHERE y =？。

pg_stat_statements记录查询消耗的总时间以及调用次数。

stddev_time ：该字段特别值得注意，因为它将告诉我们查询运行时是稳定的还是波动的。不稳定的运行时可能由于各种原因而发生：

> 1.如果数据没有完全缓存在RAM中，则必须进入磁盘的查询将比其缓存的查询花费更长的时间；
> 
> 2.不同的参数可能导致不同的计划和完全不同的结果集；
> 
> 3.并发性和锁定可能会产生影响。

shared_blks_*** ：显示有多少块来自缓存（_hit）或来自操作系统（_read）。如果有许多块来自操作系统，则查询的运行时间可能会波动。

local_blks_***：本地缓冲区是由数据库连接直接分配的内存块。

temp_blks_***：提供了有关临时文件I/O的信息。注意，临时文件I/O会在构建大型索引或执行其他大型DDL时发生。但是，对于OLTP来说，临时文件通常是一件非常糟糕的事情，因为它可能会阻塞磁盘，从而降低整个系统的速度。大量的临时文件I/O可能指向某些不良情况。以下列出了前三项：

> 1.不理想的work_mem设置（OLTP）；
> 
> 2.次优的maintenance_work_mem设置（DDL）；
> 
> 3.不应首先运行的查询；

blk_***_time：包含有关I/O计时的信息。默认情况下，这两个字段为空。原因是在某些系统上，测量计时可能会涉及很多开销。因此，track_io_timing的默认值为false。

# 查询实例
```SQL
SELECT round((100 * total_time / sum(total_time) OVER ())::numeric, 2) percent,
	round(total_time::numeric, 2) AS total,
	calls,              
	round(mean_time::numeric, 2) AS mean,
	query
FROM  pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
```
大多数情况下，前几个查询已经对应了系统上的大部分负载，没必要显示1000个查询。