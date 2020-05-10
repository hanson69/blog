---
title: PG的行级安全性（RLS）
date: 2020-05-10 22:51:24
categories: PG基础
tags:
- 行级安全性
- RLS
---

生活中有这样一个场景需求：部门的领导只能看到自己部门的数据，不可以查看公司的所有数据。

针对这个场景，PG的行级安全性（Row-level security ，RLS）能够以快速，简单的方式构建多租户系统，可以为这些权限配置相关的策略。

创建策略语法：


```SQL
postgres=# \h create policy
Command:     CREATE POLICY
Description: define a new row level security policy for a table
Syntax:
CREATE POLICY name ON table_name
    [ AS { PERMISSIVE | RESTRICTIVE } ]
    [ FOR { ALL | SELECT | INSERT | UPDATE | DELETE } ]
    [ TO { role_name | PUBLIC | CURRENT_USER | SESSION_USER } [, ...] ]
    [ USING ( using_expression ) ]
    [ WITH CHECK ( check_expression ) ]
```

首先以超级用户身份登录并创建一个包含几个记录的表：

```SQL
postgres=# CREATE TABLE t_person (gender text, name text);
CREATE TABLE
postgres=# INSERT INTO t_person  VALUES     
            ('male', 'joe'),             
            ('male', 'paul'),             
            ('female', 'sarah'),             
            (NULL, 'R2- D2');
INSERT 0 4
```

授予访问权限给joe
```SQL
postgres=# create role joe;
CREATE ROLE
postgres=# GRANT ALL ON t_person TO joe;
GRANT
```
为表启用“行级别安全”：

```SQL
ALTER TABLE t_person ENABLE ROW LEVEL SECURITY;
```

如果没有配置任何策略，会有一个拒绝所有的默认策略，因此joe角色实际上将获得一个空表。

```SQL
postgres=# SELECT * FROM t_person ;
 gender |  name
--------+--------
 male   | joe
 male   | paul
 female | sarah
        | R2- D2
(4 rows)

postgres=# set role joe;
SET
postgres=> SELECT * FROM t_person ;
 gender | name
--------+------
(0 rows)
```

实际上，默认策略在很大程度上是有意义的，因为用户被迫显式设置权限。现在，该表处于行级安全性控制之下，可以以超级用户身份编写策略：

```SQL
postgres=> CREATE POLICY joe_pol_1  ON t_person  FOR SELECT TO joe  USING  (gender = 'male');
ERROR:  must be owner of table t_person
postgres=> set role hanson;
SET
postgres=# CREATE POLICY joe_pol_1  ON t_person  FOR SELECT TO joe  USING  (gender = 'male');
CREATE POLICY
```

以joe角色身份登录并选择所有数据将仅返回两行

```SQL
postgres=> SELECT * FROM  t_person;
 gender | name
--------+------
 male   | joe
 male   | paul
(2 rows)
```

策略允许哪些操作执行（本例为SELECT子句）。然后是USING子句，定义了允许joe角色看到的内容。因此，USING子句是附加到每个查询的强制性过滤器，仅用于选择用户应该看到的行。

如果不止一个策略，PostgreSQL将使用OR条件。即更多策略将看到更多数据。用户可以选择是否将条件连接为OR和AND（PERMISSIVE 和 RESTRICTIVE）

默认是PERMISSIVE，即OR状态。如果使用RESTRICTIVE，那么这些子句将与AND关联。

如果允许joe角色可以查看机器人。有两个选择可以实现我们的目标。第一种选择是简单地使用ALTER POLICY子句来更改现有策略。

第二个选择是创建第二个策略，如下例所示

```SQL
postgres=> set role hanson;
SET
postgres=# CREATE POLICY joe_pol_2  ON t_person  FOR SELECT TO joe  USING  (gender IS NULL);
CREATE POLICY
```

除非使用RESTRICTIVE，否则这些策略仅使用前面所述的OR条件进行连接。因此，PostgreSQL现在将返回三行，而不是两行：


```SQL
postgres=> SELECT * FROM  t_person;
 gender |  name
--------+--------
 male   | joe
 male   | paul
        | R2- D2
(3 rows)
```

查看查询的执行计划

```SQL
postgres=> explain SELECT * FROM  t_person;
                            QUERY PLAN
------------------------------------------------------------------
 Seq Scan on t_person  (cost=0.00..23.20 rows=9 width=64)
   Filter: ((gender = 'female'::text) OR (gender = 'male'::text))
(2 rows)
```
如上所见，两个USING子句均已添加为查询的强制性过滤器。

在语法定义中有两种类型的子句：

- USING：此子句过滤已存在的行。这与SELECT和UPDATE子句相关。
- CHECK：此子句将过滤将要创建的新行，因此它们与INSERT和UPDATE子句相关。

如果尝试插入，则会发生以下情况：

```SQL
postgres=> INSERT INTO t_person VALUES ('male', 'kaarel');
ERROR:  new row violates row-level security policy for table "t_person"
```

由于没有针对INSERT子句的策略，因此该语句会出错。
下面创建插入策略，允许joe角色将男性和女性添加到表中：

```SQL
postgres=# CREATE POLICY joe_pol_3  ON t_person           
            FOR INSERT TO joe           
            WITH CHECK (gender IN ('male', 'female'));
CREATE POLICY
```

```SQL
postgres=# set role joe;
SET

INSERT INTO t_person VALUES ('female', 'maria');
INSERT 0 1
```

但是，也有一个陷阱。考虑下面的例子

```SQL
postgres=> INSERT INTO  t_person VALUES ('female', 'maria') RETURNING *;
ERROR:  new row violates row-level security policy for table "t_person"
```
因为joe角色仅可以查看男性记录，只有男人记录才能使用RETURNING *子句。这里的麻烦是该语句将返回一个女人，这是不允许的。

查看表的相关权限信息
```SQL
postgres=> \z t_person
Access privileges
-[ RECORD 1 ]-----+------------------------------------------------------------
Schema            | public
Name              | t_person
Type              | table
Access privileges | hanson=arwdDxt/hanson                                      +
                  | joe=arwdDxt/hanson
Column privileges |
Policies          | joe_pol_1 (r):                                             +
                  |   (u): (gender = 'male'::text)                             +
                  |   to: joe                                                  +
                  | joe_pol_2 (r):                                             +
                  |   (u): (gender IS NULL)                                    +
                  |   to: joe                                                  +
                  | joe_pol_3 (a):                                             +
                  |   (c): (gender = ANY (ARRAY['male'::text, 'female'::text]))+
                  |   to: joe
```

- a：为INSERT子句追加：
- r: 为SELECT子句读取;
- w：为UPDATE子句写：
- d: 为DELETE子句删除;
- D：用于TRUNCATE子句（引入时，t已经被使用）;
- x：用于引用;
- t: 用于触发器


删除joe角色

```SQL
postgres=# DROP ROLE joe;
ERROR:  role "joe" cannot be dropped because some objects depend on it
DETAIL:  privileges for table t_person
target of policy joe_pol_1 on table t_person
target of policy joe_pol_2 on table t_person
target of policy joe_pol_3 on table t_person
owner of table tb_joe
```

PostgreSQL会发出错误消息，因为有其它对象属于joe角色。出于这个原因，PostgreSQL可以将表从一个用户重新分配给另一个用户，使用REASSIGN子句：

```SQL
postgres=# REASSIGN OWNED  BY joe TO hanson;
REASSIGN OWNED
postgres=# DROP ROLE joe;
ERROR:  role "joe" cannot be dropped because some objects depend on it
DETAIL:  privileges for table t_person
target of policy joe_pol_1 on table t_person
target of policy joe_pol_2 on table t_person
target of policy joe_pol_3 on table t_person
```

现在“owner of table tb_joe”错误没了。我们现在所能做的就是一个接一个地解决所有这些问题，然后删除角色，没有捷径可走。


# 原文：
《Mastering PostgreSQL11 (2ndEdition)——Digging into row-level security – RLS》