---
title: 【翻译】PostgreSQL中的索引-2
date: 2019-11-27 13:11:03
categories: PG基础
tags:
- 索引
- 访问方法的属性
---


在第一篇文章中，我们提到了访问方法必须提供有关其自身的信息。让我们看一下访问方法接口的结构。

# 性质

访问方法的所有属性都存储在“pg_am”表中（“am”：access method）。我们还可以从同一张表中获得可用方法的列表：


```SQL
postgres=# select amname from pg_am;
 amname
--------
 btree
 hash
 gist
 gin
 spgist
 brin
(6 rows)
```


尽管可以正确地将顺序扫描称为访问方法，但是由于历史原因，它不在此列表中。

在PostgreSQL 9.5和更低的版本中，每个属性都由«pg_am»表的单独字段表示。从9.6版开始，使用特殊函数查询属性，并将其分为几层：

- 访问方法属性—«pg_indexam_has_property»
- 特定索引的属性—«pg_index_has_property»
- 索引各列的属性—«pg_index_column_has_property»

未来，访问方法层和索引层是分开的：到目前为止，所有基于一种访问方法的索引将始终具有相同的属性。

以下四个属性是访问方法的属性（以«btree»为例）：


```SQL
postgres=# select a.amname, p.name, pg_indexam_has_property(a.oid,p.name)
            from pg_am a,
            unnest(array['can_order','can_unique','can_multi_col','can_exclude']) p(name)
            where a.amname = 'btree'
            order by a.amname;
 amname |     name      | pg_indexam_has_property
--------+---------------+-------------------------
 btree  | can_order     | t
 btree  | can_unique    | t
 btree  | can_multi_col | t
 btree  | can_exclude   | t
(4 rows)
```


- can_order。
访问方法使我们能够在创建索引时指定值的排序顺序（到目前为止仅适用于“ btree”）。
- can_unique。
支持唯一约束和主键（仅适用于“ btree”）。
- can_multi_col。
索引可以建立在几列上。
- can_exclude。
支持排除约束EXCLUDE。

以下属性与索引有关（例如，让我们考虑现有索引）：


```SQL
postgres=# select p.name, pg_index_has_property('t_a_idx'::regclass,p.name)
            from unnest(array[
                        'clusterable','index_scan','bitmap_scan','backward_scan'
                            ]) p(name);
     name      | pg_index_has_property
---------------+-----------------------
 clusterable   | t
 index_scan    | t
 bitmap_scan   | t
 backward_scan | t
(4 rows)
```


- clusterable。
可以根据索引对行进行重新排序（使用同名命令CLUSTER进行聚集）。
- index_scan。
虽然这个属性看起来很奇怪，但并不是所有索引都可以逐个返回tid——有些索引一次返回所有结果，并且只支持位图扫描。
- bitmap_scan。
支持位图扫描。
- backward_scan。
可以按建立索引时指定的相反顺序返回结果。

最后，以下是列属性：


```SQL
postgres=# select p.name,
     pg_index_column_has_property('t_a_idx'::regclass,1,p.name)
from unnest(array[
       'asc','desc','nulls_first','nulls_last','orderable','distance_orderable',
       'returnable','search_array','search_nulls'
     ]) p(name);
        name        | pg_index_column_has_property
--------------------+------------------------------
 asc                | t
 desc               | f
 nulls_first        | f
 nulls_last         | t
 orderable          | t
 distance_orderable | f
 returnable         | t
 search_array       | t
 search_nulls       | t
(9 rows)
```


- asc，desc，nulls_first，nulls_last，orderable。
这些属性与值的排序有关（对“ btree”索引描述时，我们将讨论它们）。
- distance_orderable。
可以按照操作确定的排序顺序返回结果（到目前为止仅适用于GiST和RUM索引）。
- returnable。
在不访问表的情况下使用索引的可能性，即 index-only scans 的支持。
- search_array。
支持使用表达式 «indexed-field IN (list_of_constants)» 搜索多个值，这与 «indexed-field = ANY(array_of_constants)»相同。
- search_nulls。
通过 IS NULL 和 IS NOT NULL 条件进行搜索的可能性。