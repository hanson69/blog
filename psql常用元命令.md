# psql常用元命令

---
- author：Hanson Huang
- date：2019.08.18
- label：PostgreSQL, greenplum, 元命令
---

PostgreSQL提供了很多元命令方便查找相关信息，可以通过‘\\?’列出所有的元命令帮助信息。这里列出一些常用的元命令。
```
postgres=# \?
General
  \h [NAME]              help on syntax of SQL commands, * for all commands
  \q                     quit psql

Informational
  (options: S = show system objects, + = additional detail)
  \d[S+]                 list tables, views, and sequences
  \d[S+]  NAME           describe table, view, sequence, or index
  \db[+]  [PATTERN]      list tablespaces
  \df[antw][S+] [PATRN]  list [only agg/normal/trigger/window] functions
  \dg[+]  [PATTERN]      list roles (groups)
  \dx[+]  [PATTERN]      list extensions
  \di[S+] [PATTERN]      list indexes
  \dn[+]  [PATTERN]      list schemas
  \do[S]  [PATTERN]      list operators
  \dp     [PATTERN]      list table, view, and sequence access privileges
  \dr[S+] [PATTERN]      list foreign tables
  \ds[S+] [PATTERN]      list sequences
  \dt[S+] [PATTERN]      list tables
  \dT[S+] [PATTERN]      list data types
  \du[+]  [PATTERN]      list roles (users)
  \dv[S+] [PATTERN]      list views
  \dE     [PATTERN]      list external tables
  \l[+]                  list all databases

Formatting
  \x [on|off]            toggle expanded output (currently off)

Connection
  \c[onnect] [DBNAME|- USER|- HOST|- PORT|-]
                         connect to new database (currently "postgres")
  \encoding [ENCODING]   show or set client encoding
  \conninfo              display information about current connection

Operating System
  \cd [DIR]              change the current working directory
  \timing [on|off]       toggle timing of commands (currently off)
  \! [COMMAND]           execute command in shell or start interactive shell
```

###### 1. 获取SQL的帮助信息
比如获取创建表的SQL：

```SQL
postgres=# \h create TABLE
Command:     CREATE TABLE
Description: define a new table
Syntax:
CREATE [[GLOBAL | LOCAL] {TEMPORARY | TEMP}] TABLE table_name (
[ { column_name data_type [ DEFAULT default_expr ]     [column_constraint [ ... ]
[ ENCODING ( storage_directive [,...] ) ]
]
```

###### 2. 查询当前库中所有的表

```
postgres=# \d
            List of relations
 Schema | Name | Type  | Owner  | Storage
--------+------+-------+--------+---------
 public | test | table | hanson | heap
(1 row)
```


###### 3. 查询表、视图、索引的描述信息
```
postgres=# \d test
     Table "public.test"
 Column |  Type   | Modifiers
--------+---------+-----------
 id     | integer |
Distributed by: (id)
```

###### 4. 查询所有表空间

```
postgres=# \db
         List of tablespaces
    Name    | Owner  | Filespace Name
------------+--------+----------------
 pg_default | hanson | pg_system
 pg_global  | hanson | pg_system
(2 rows)
```


###### 5. 查询函数
```
postgres=# \df pg_reload_conf
                               List of functions
   Schema   |      Name      | Result data type | Argument data types |  Type
------------+----------------+------------------+---------------------+--------
 pg_catalog | pg_reload_conf | boolean          |                     | normal
(1 row)
```
技巧：
如果想查询以pg_re开头的函数，可以通过''\df pg_re'，然后按两次Tab键查询。

```
postgres=# \df pg_re
pg_read_file               pg_relation_size           pg_reload_conf             pg_renice_session
pg_resgroup_get_status     pg_resqueue_status         pg_resqueue_status_kv      pg_resgroup_get_status_kv
```

###### 6. 查询所有角色
```
postgres=# \du
                                                                      List of roles
 Role name |                                                           Attributes
                   | Member of
-----------+--------------------------------------------------------------------------------------------------------------
-------------------+-----------
 hanson    | Superuser, Create role, Create DB, Ext gpfdist Table, Wri Ext gpfdist Table, Ext http Table, Ext hdfs Table,
Wri Ext hdfs Table | {}
```

###### 7. 查询所有的库

```SQL
postgres=# \l
                 List of databases
   Name    | Owner  | Encoding | Access privileges
-----------+--------+----------+-------------------
 postgres  | hanson | UTF8     |
 template0 | hanson | UTF8     | =c/hanson
                               : hanson=CTc/hanson
 template1 | hanson | UTF8     | =c/hanson
                               : hanson=CTc/hanson
(3 rows)
```

###### 8. 切换扩展输出

```SQL
postgres=# select * from gp_segment_configuration;
 dbid | content | role | preferred_role | mode | status | port  | hostname | address | replication_port
------+---------+------+----------------+------+--------+-------+----------+---------+------------------
    1 |      -1 | p    | p              | s    | u      |  5432 | mdw      | mdw     |
    2 |       0 | p    | p              | s    | u      | 40000 | sdw1     | sdw1    |            41000
    4 |       2 | p    | p              | s    | u      | 40000 | sdw2     | sdw2    |            41000
    3 |       1 | p    | p              | s    | u      | 40001 | sdw1     | sdw1    |            41001
    5 |       3 | p    | p              | s    | u      | 40001 | sdw2     | sdw2    |            41001
    6 |       0 | m    | m              | s    | u      | 50000 | sdw2     | sdw2    |            51000
    7 |       1 | m    | m              | s    | u      | 50001 | sdw2     | sdw2    |            51001
    8 |       2 | m    | m              | s    | u      | 50000 | sdw1     | sdw1    |            51000
    9 |       3 | m    | m              | s    | u      | 50001 | sdw1     | sdw1    |            51001
(9 rows)

postgres=# \x
Expanded display is on.
postgres=# select * from gp_segment_configuration;                                                                        -[ RECORD 1 ]----+------
dbid             | 1
content          | -1
role             | p
preferred_role   | p
mode             | s
status           | u
port             | 5432
hostname         | mdw
address          | mdw
replication_port |
-[ RECORD 2 ]----+------
dbid             | 2
content          | 0
role             | p
preferred_role   | p
mode             | s
status           | u
port             | 40000
hostname         | sdw1
address          | sdw1
replication_port | 41000
--More--
```

###### 9. 连接到其它库
```
template1=# \c postgres hanson
You are now connected to database "postgres" as user "hanson".
postgres=#
```

###### 10. 查询当前库的编码

```
postgres=# \encoding
UTF8
```

###### 11. 查询当前连接信息

```
postgres=# \conninfo
You are connected to database "postgres" as user "hanson" via socket in "/tmp" at port "5432".
```
######  12. 查询命令执行时间

```SQL
postgres=# \timing
Timing is on.
postgres=# select * from test where id=10;
 id
----
 10
(1 row)

Time: 145.279 ms
```
######  13. 执行Linux命令

```
postgres=# \! date
Wed Aug 14 21:14:39 CST 2019
postgres=# \! pwd
/home/hanson
```

###### 14. 查询当前元命令的SQL实现

```
[hanson@mdw ~]$ psql -E
psql (8.3.23)
Type "help" for help.

postgres=# \d
********* QUERY **********
select version()
**************************

********* QUERY **********
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'i' THEN 'index' WHEN 'S' THEN 'sequence' WHEN 's' THEN 'special' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner", CASE c.relstorage WHEN 'h' THEN 'heap' WHEN 'x' THEN 'external' WHEN 'a' THEN 'append only' WHEN 'v' THEN 'none' WHEN 'c' THEN 'append only columnar' END as "Storage"

FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','v','S','')
AND c.relstorage IN ('h', 'a', 'c','x','v','')
      AND n.nspname <> 'pg_catalog'
      AND n.nspname <> 'information_schema'
      AND n.nspname !~ '^pg_toast'
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
**************************

            List of relations
 Schema | Name | Type  | Owner  | Storage
--------+------+-------+--------+---------
 public | test | table | hanson | heap
(1 row)
```

