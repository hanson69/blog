---
title: 【翻译】PostgreSQL中的索引4-btree
date: 2020-05-10 19:51:24
categories: PG基础
tags:
- btree
---



我们已经讨论了PostgreSQL 索引引擎和访问方法的接口，以及访问方法之一的哈希索引。现在，我们将讨论B树，这是最传统且使用最广泛的索引。这篇文章很长，请耐心阅读。

# 一、结构


B树索引类型，实现为«btree»访问方法，适用于可以排序的数据。换句话说，必须为数据类型定义«greater»、«greater or equal»、«less»、«less or equal»和«equal»运算符。请注意，相同的数据有时可以以不同的方式排序，这使我们回到了运算符族的概念。

与往常一样，B树的索引行被压缩到页面中。在叶子的页面中，这些行包含要建立索引的数据（键）和对表行的引用（TID）。在内部页面中，每一行都引用索引的子页面，并包含该页中的最小值。

B树具有一些重要的特征：

· B树是平衡的，也就是说，每个叶页面与根都由相同数量的内部页面分隔开。因此，搜索任何值都需要花费相同的时间。

· B树是多分支的，即每个页面（通常为8 KB）包含许多（数百个）TID。因此，B树的深度很小，对于非常大的表，实际上可以达到4–5的深度。

· 索引中的数据按非递减顺序排序（在页面之间和每个页面内部），并且同一级别的页面通过双向列表相互连接。因此，我们可以仅通过列表的一个方向或另一个方向获得有序数据集，而不必每次都返回到根。


下面是带整数键的一个字段上的索引的简化示例：
![PostgreSQL中的索引4-btree](2020-05-10-20-56-53.png)


索引的第一页是一个元页，它引用索引根。内部节点位于根下方，叶子页面位于最底行。向下箭头表示从叶节点到表行（TID）的引用。

## 1.相等搜索


让我们考虑通过条件“ indexed-field = expression索引字段=表达式 ” 在树中搜索值。假设我们对49的键感兴趣。
![PostgreSQL中的索引4-btree](2020-05-10-20-59-11.png)


搜索从根节点开始，我们需要确定要降到哪个子节点。由于知道根节点（4、32、64）中的键，因此我们可以计算出子节点中的值范围。由于32≤49 <64，所以我们需要下降到第二个子节点。接下来，递归重复相同的过程，直到我们到达一个叶节点，可以从该叶节点获得所需的TID。

实际上，许多细节使这个看似简单的过程变得复杂。例如，一个索引可以包含非唯一键，并且可能有太多相等的值以至于它们不适合一页。回到我们的示例，似乎我们应该从内部节点下降到参考值49。但是，从图中可以清楚地看出，通过这种方式，我们将跳过上一页中的«49»键中的一个。因此，一旦我们在内部页面中找到一个完全相等的键，就必须向左移一个位置，然后从左到右查看底层的索引行以寻找所需的键。

（另一个复杂之处在于，在搜索过程中，其他进程可以更改数据：可以重建树，可以将页面拆分为两个，依此类推。所有算法都针对这些并发操作而设计，不会互相干扰，也不会在任何可能的地方引起额外的锁定，但我们会避免在此基础上进行扩展。）

## 2.不等搜索


当通过条件“indexed-field ≤ expression索引字段 ≤ 表达式 ”（或"indexed-field ≥ expression索引字段 ≥ 表达式 “）进行搜索时，我们首先通过等式条件"indexed-field = expression" 在索引中找到一个值（如果有的话），然后按照适当的方向遍历叶页面直到结束。

该图说明了n≤35的过程：
![PostgreSQL中的索引4-btree](2020-05-10-21-00-22.png)

以类似的方式支持«greater»和«less»运算符，只是必须删除最初找到的值。

3.按范围搜索


通过范围 "expression1 ≤ indexed-field ≤ expression2 表达式1≤索引字段≤表达式2"进行搜索时，通过条件"indexed-field = expression1 索引字段=表达式1"找到一个值，然后遍历叶页面直到"indexed-field ≤ expression2 索引字段≤表达式2"被满足; 反之亦然：从第二个表达式开始，然后朝相反的方向走，直到到达第一个表达式。

该图显示了条件23≤n≤64的过程：
![PostgreSQL中的索引4-btree](2020-05-10-21-01-46.png)


# 二、示例


让我们看一个查询计划的示例。像往常一样，我们使用演示数据库，这次我们将考虑aircraft表。它仅包含9行，规划器会选择不使用索引，因为整个表都被存放在一个页面。但是，为了说明目的，该表对我们来说很有趣。


```
demo=# select * from aircrafts;

 aircraft_code |        model        | range

---------------+---------------------+-------

 773           | Boeing 777-300      | 11100

 763           | Boeing 767-300      |  7900

 SU9           | Sukhoi SuperJet-100 |  3000

 320           | Airbus A320-200     |  5700

 321           | Airbus A321-200     |  5600

 319           | Airbus A319-100     |  6700

 733           | Boeing 737-300      |  4200

 CN1           | Cessna 208 Caravan  |  1200

 CR2           | Bombardier CRJ-200  |  2700

(9 rows)

demo=# create index on aircrafts(range);

 

demo=# set enable_seqscan = off;
```



（或者明确地说，«在aircrafts表上使用btree（range）创建索引»，但是它是一个默认构建的B树。）

通过等式搜索：


```
demo=# explain(costs off) select * from aircrafts where range = 3000;

                    QUERY PLAN                     

---------------------------------------------------

 Index Scan using aircrafts_range_idx on aircrafts

   Index Cond: (range = 3000)

(2 rows)
```



按不等式搜索：


```
demo=# explain(costs off) select * from aircrafts where range < 3000;

                    QUERY PLAN                    

---------------------------------------------------

 Index Scan using aircrafts_range_idx on aircrafts

   Index Cond: (range < 3000)

(2 rows)
```


按范围搜索：


```
demo=# explain(costs off) select * from aircrafts

where range between 3000 and 5000;

                     QUERY PLAN                      

-----------------------------------------------------

 Index Scan using aircrafts_range_idx on aircrafts

   Index Cond: ((range >= 3000) AND (range <= 5000))

(2 rows)
```



# 三、排序


让我们再次强调一点，对于任何类型的扫描（索引，仅索引或位图），«btree»访问方法都会返回有序数据，我们可以在上图中清楚地看到。

因此，如果表在排序条件上具有索引，则优化器将考虑这两个选项：表的索引扫描（该索引扫描很容易返回排序后的数据）以及表的顺序扫描（随后对结果进行排序）。

## 1.排序顺序


创建索引时，我们可以显式地指定排序顺序。例如，我们可以通过这种方式按照飞行距离创建索引：

demo=# create index on aircrafts(range desc);


在这种情况下，较大的值将出现在左侧的树中，而较小的值将出现在右侧。如果我们可以在任何方向上遍历索引值，为什么还需要这样做？

其目的是多列索引。让我们创建一个视图来显示aircraft（飞机）模型，传统的划分为短程，中程和远程飞机：


```
demo=# create view aircrafts_v as

select model,

       case

           when range < 4000 then 1

           when range < 10000 then 2

           else 3

       end as class

from aircrafts;

 

demo=# select * from aircrafts_v;

        model        | class

---------------------+-------

 Boeing 777-300      |     3

 Boeing 767-300      |     2

 Sukhoi SuperJet-100 |     1

 Airbus A320-200     |     2

 Airbus A321-200     |     2

 Airbus A319-100     |     2

 Boeing 737-300      |     2

 Cessna 208 Caravan  |     1

 Bombardier CRJ-200  |     1

(9 rows)
```


让我们创建一个索引（使用表达式）：


```
demo=# create index on aircrafts(

  (case when range < 4000 then 1 when range < 10000 then 2 else 3 end),

  model);
```


现在，我们可以使用该索引来获取对两列按升序排序的数据：


```
demo=# select class, model from aircrafts_v order by class, model;

 class |        model        

-------+---------------------

     1 | Bombardier CRJ-200

     1 | Cessna 208 Caravan

     1 | Sukhoi SuperJet-100

     2 | Airbus A319-100

     2 | Airbus A320-200

     2 | Airbus A321-200

     2 | Boeing 737-300

     2 | Boeing 767-300

     3 | Boeing 777-300

(9 rows)

demo=# explain(costs off)

select class, model from aircrafts_v order by class, model;

                       QUERY PLAN                       

--------------------------------------------------------

 Index Scan using aircrafts_case_model_idx on aircrafts

(1 row)
```


同样，我们可以执行查询以降序对数据进行排序：


```
demo=# select class, model from aircrafts_v order by class desc, model desc;

 class |        model        

-------+---------------------

     3 | Boeing 777-300

     2 | Boeing 767-300

     2 | Boeing 737-300

     2 | Airbus A321-200

     2 | Airbus A320-200

     2 | Airbus A319-100

     1 | Sukhoi SuperJet-100

     1 | Cessna 208 Caravan

     1 | Bombardier CRJ-200

(9 rows)

demo=# explain(costs off)

select class, model from aircrafts_v order by class desc, model desc;

                           QUERY PLAN                            

-----------------------------------------------------------------

 Index Scan BACKWARD using aircrafts_case_model_idx on aircrafts

(1 row)
```


但是，我们不能使用此索引对一列降序，而对另一列升序来获取数据。这需要单独排序：


```
demo=# explain(costs off)

select class, model from aircrafts_v order by class ASC, model DESC;

                   QUERY PLAN                    

-------------------------------------------------

 Sort

   Sort Key: (CASE ... END), aircrafts.model DESC

   ->  Seq Scan on aircrafts

(3 rows)
```


（请注意，作为最后的选择，计划器选择了顺序扫描，而不考虑之前的«enable_seqscan = off»设置。这是因为实际上此设置并不禁止表扫描，而只是设置其天文成本（个人理解是成本比较大的时候，上边的开关不起作用），请仔细阅读该计划 «costs on»。）

要使该查询使用索引，必须使用所需的排序方向来构建索引：


```
demo=# create index aircrafts_case_asc_model_desc_idx on aircrafts(

 (case

    when range < 4000 then 1

    when range < 10000 then 2

    else 3

  end) ASC,

  model DESC);

 

demo=# explain(costs off)

select class, model from aircrafts_v order by class ASC, model DESC;

                           QUERY PLAN                            

-----------------------------------------------------------------

 Index Scan using aircrafts_case_asc_model_desc_idx on aircrafts

(1 row)
```



## 2.列顺序


使用多列索引时出现的另一个问题是索引中列出列的顺序。对于B树，此顺序非常重要：页面内的数据将按第一个字段排序，然后按第二个字段排序，以此类推。

我们可以用以下符号方式表示我们在范围区间和模型上构建的索引：
![PostgreSQL中的索引4-btree](2020-05-10-21-03-27.png)

实际上，这么小的索引肯定只适合一个根页面。在图中，为了清楚起见，它故意分布在几页中。

从此图表可以清楚地看出，通过像«class = 3»（仅在第一个字段中进行搜索）或«class = 3 and model = 'Boeing 777-300'»（在两个字段中进行搜索）这样的谓词进行搜索将有效地工作。

但是，根据谓词 «model = 'Boeing 777-300'» 进行搜索的效率会低得多：从根节点开始，我们无法确定要下降到哪个子节点，因此，我们必须下降到所有子节点。这并不意味着永远不能使用这样的索引——它的效率是有问题的。例如，如果我们有三类飞机，而每一类中都有很多模型，则我们将不得不查看大约三分之一的索引，这可能比全表扫描更有效……或者相反。

但是，如果我们创建这样的索引：


```
demo=# create index on aircrafts(

  model,

  (case when range < 4000 then 1 when range < 10000 then 2 else 3 end));
```


字段的顺序将发生变化：

![PostgreSQL中的索引4-btree](2020-05-10-21-04-22.png)

使用此索引，根据谓词 «model = 'Boeing 777-300'» 进行的搜索将有效地进行，但是根据谓词«class = 3»进行的搜索将无效。

## 3.空值


«btree»访问方法可以索引NULL，并支持按条件IS NULL和IS NOT NULL进行搜索。

让我们考虑flights航班表，其中出现NULL：


```
demo=# create index on flights(actual_arrival);

 

demo=# explain(costs off) select * from flights where actual_arrival is null;

                      QUERY PLAN                       

-------------------------------------------------------

 Bitmap Heap Scan on flights

   Recheck Cond: (actual_arrival IS NULL)

   ->  Bitmap Index Scan on flights_actual_arrival_idx

         Index Cond: (actual_arrival IS NULL)

(4 rows)
```



NULL位于叶节点的一端或另一端，具体取决于创建索引的方式（NULLS FIRST或NULLS LAST）。如果查询包括排序，则这一点很重要：如果SELECT命令在其ORDER BY子句中指定的NULL顺序与构建索引指定的顺序相同（NULLS FIRST或NULLS LAST），则可以使用索引。

在下面的示例中，这些顺序相同，因此，我们可以使用索引：


```
demo=# explain(costs off)

select * from flights order by actual_arrival NULLS LAST;

                       QUERY PLAN                      

--------------------------------------------------------

 Index Scan using flights_actual_arrival_idx on flights

(1 row)
```


这些顺序是不同的，优化器选择顺序扫描和后续排序：


```
demo=# explain(costs off)

select * from flights order by actual_arrival NULLS FIRST;

               QUERY PLAN              

----------------------------------------

 Sort

   Sort Key: actual_arrival NULLS FIRST

   ->  Seq Scan on flights

(3 rows)
```


要使用索引，必须使用位于开头的NULL创建索引：


```
demo=# create index flights_nulls_first_idx on flights(actual_arrival NULLS FIRST);

 

demo=# explain(costs off)

select * from flights order by actual_arrival NULLS FIRST;

                     QUERY PLAN                      

-----------------------------------------------------

 Index Scan using flights_nulls_first_idx on flights

(1 row)
```



像这样的问题肯定是由NULL无法排序引起的，即NULL与其他任何值的比较结果均未定义：


```
demo=# \pset null NULL

 

demo=# select null < 42;

 ?column?

----------

 NULL

(1 row)
```


这与B树的概念背道而驰，不符合一般模式。但是，NULL在数据库中起着非常重要的作用，因此我们总是不得不为它们设置例外。

由于可以为NULL建立索引，因此即使对表没有施加任何条件，也可以使用索引（因为索引肯定包含所有表行上的信息）。如果查询需要数据排序并且索引确保了所需的顺序，则这样做很有意义。在这种情况下，计划者可以选择索引访问来保存单独的排序。

# 四、属性


让我们看一下«btree»访问方法的属性（根据之前第二节提供的查询）。


```
amname |     name      | pg_indexam_has_property

--------+---------------+-------------------------

 btree  | can_order     | t

 btree  | can_unique    | t

 btree  | can_multi_col | t

 btree  | can_exclude   | t
```



如我们所见，B树可以排序数据并支持唯一性——这是为我们提供此类属性的唯一访问方法。也允许使用多列索引，但是其他访问方法（尽管不是全部）都可以支持此类索引。下次，我们将讨论EXCLUDE约束的支持，这不是没有原因的。


```
name      | pg_index_has_property

---------------+-----------------------

 clusterable   | t

 index_scan    | t

 bitmap_scan   | t

 backward_scan | t
```



«btree»访问方法支持两种获取值的技术：索引扫描和位图扫描。正如我们所看到的，访问方法既可以“向前”«forward»又可以“向后”«backward»遍历树。


```
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
```



该层的前四个属性说明特定列的值如何精确排序。在此示例中，值以升序（«asc»）排序，最后提供NULL（«nulls_last»）。但是正如我们已经看到的，其他组合也是可能的。


```
«search_array»属性通过索引支持这样的表达式：

demo=# explain(costs off)

select * from aircrafts where aircraft_code in ('733','763','773');

                           QUERY PLAN                            

-----------------------------------------------------------------

 Index Scan using aircrafts_pkey on aircrafts

   Index Cond: (aircraft_code = ANY ('{733,763,773}'::bpchar[]))

(2 rows)
```


«returnable»属性表示支持仅索引扫描，这是合理的，因为索引行本身存储索引值（与哈希索引不同）。在这里，有必要谈谈关于覆盖基于B树的索引。

## 1.具有其他行的唯一索引


正如我们前面所讨论的，覆盖索引是存储查询所需的所有值的索引，（几乎）不需要访问表本身。唯一索引可以专门覆盖。

但是，假设我们要向唯一索引添加查询所需的额外列。新的组合键的值现在可能不是唯一的，因此将需要在同一列上使用两个索引：一个唯一的（用于支持完整性约束）和另一个非唯一的（用于覆盖）。这肯定是低效的。

在我们公司的Anastasiya Lubennikova改进的«btree»方法，可以在唯一索引中包含其他非唯一的列。我们希望该补丁能被社区所采用，成为PostgreSQL的一部分，但是这不会在版本10中实现。目前，该补丁在Pro Standard 9.5+中可用，这就是它的样子。

实际上，此补丁已提交给PostgreSQL 11。


让我们考虑bookings预订表：
```
demo=# \d bookings

              Table "bookings.bookings"

    Column    |           Type           | Modifiers

--------------+--------------------------+-----------

 book_ref     | character(6)             | not null

 book_date    | timestamp with time zone | not null

 total_amount | numeric(10,2)            | not null

Indexes:

    "bookings_pkey" PRIMARY KEY, btree (book_ref)

Referenced by:

    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
```

在此表中，主键（book_ref，booking code预订代码）由常规的«btree»索引提供。让我们创建一个带有附加列的新唯一索引：
```
demo=# create unique index bookings_pkey2 on bookings(book_ref) INCLUDE (book_date);
```

现在，我们用一个新的索引替换现有的索引（在事务中，同时应用所有更改）：

```
demo=# begin;

 

demo=# alter table bookings drop constraint bookings_pkey cascade;

 

demo=# alter table bookings add primary key using index bookings_pkey2;

 

demo=# alter table tickets add foreign key (book_ref) references bookings (book_ref);

 

demo=# commit;
```


这是我们得到的：


```
demo=# \d bookings

              Table "bookings.bookings"

    Column    |           Type           | Modifiers

--------------+--------------------------+-----------

 book_ref     | character(6)             | not null

 book_date    | timestamp with time zone | not null

 total_amount | numeric(10,2)            | not null

Indexes:

    "bookings_pkey2" PRIMARY KEY, btree (book_ref) INCLUDE (book_date)

Referenced by:

    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
```


现在，一个相同的索引可以作为唯一索引，并充当此查询的覆盖索引，例如：

```
demo=# explain(costs off)

select book_ref, book_date from bookings where book_ref = '059FC4';

                    QUERY PLAN                    

--------------------------------------------------

 Index Only Scan using bookings_pkey2 on bookings

   Index Cond: (book_ref = '059FC4'::bpchar)

(2 rows)
```



# 五、创建索引


众所周知，同样重要的是，对于大型表，最好在没有索引的情况下加载数据，然后再创建所需的索引。这样不仅速度更快，索引的大小也更小。

事实是，«btree»索引的创建比将值按行插入树中使用的过程更有效。大致上，对表中所有可用的数据进行排序，并创建这些数据的叶页面。然后将内部页面构建在此基础之上，直到整个金字塔收敛到根为止。

此过程的速度取决于可用RAM的大小，该大小受«maintenance_work_mem»参数的限制。因此，增加参数值可以加快此过程。对于唯一索引，除了«maintenance_work_mem»之外，还分配了大小为 «work_mem»的内存。

## 1.比较语义


上次我们提到PostgreSQL需要知道调用哪些哈希函数来获取不同类型的值，并且该关联存储在«hash»访问方法中。同样，系统必须弄清楚如何对值进行排序。这对于排序，分组（有时），合并联接等都是必需的。PostgreSQL不会绑定运算符名称（例如>，<，=），因为用户可以定义自己的数据类型并为相应的运算符赋予不同的名称。«btree»访问方法使用的运算符族定义了运算符名称。

例如，这些比较运算符用于«bool_ops»运算符族：

```
postgres=# select   amop.amopopr::regoperator as opfamily_operator,

         amop.amopstrategy

from     pg_am am,

         pg_opfamily opf,

         pg_amop amop

where    opf.opfmethod = am.oid

and      amop.amopfamily = opf.oid

and      am.amname = 'btree'

and      opf.opfname = 'bool_ops'

order by amopstrategy;

  opfamily_operator  | amopstrategy

---------------------+--------------

 <(boolean,boolean)  |            1

 <=(boolean,boolean) |            2

 =(boolean,boolean)  |            3

 >=(boolean,boolean) |            4

 >(boolean,boolean)  |            5

(5 rows)
```



在这里我们可以看到五个比较运算符，但是正如已经提到的，我们不应该依赖它们的名称。为了弄清楚每个运算符做了哪些比较，引入了策略概念。定义了五种策略来描述运算符语义：

· 1——小于

· 2——小于或等于

· 3——等于

· 4——大于或等于

· 5——大于


一些运算符族可以包含多个运算符实现一项策略。例如，«integer_ops»运算符族包含策略1的以下运算符：

```
postgres=# select   amop.amopopr::regoperator as opfamily_operator

from     pg_am am,

         pg_opfamily opf,

         pg_amop amop

where    opf.opfmethod = am.oid

and      amop.amopfamily = opf.oid

and      am.amname = 'btree'

and      opf.opfname = 'integer_ops'

and      amop.amopstrategy = 1

order by opfamily_operator;

  opfamily_operator  

----------------------

 <(integer,bigint)

 <(smallint,smallint)

 <(integer,integer)

 <(bigint,bigint)

 <(bigint,integer)

 <(smallint,integer)

 <(integer,smallint)

 <(smallint,bigint)

 <(bigint,smallint)

(9 rows)
```

因此，当比较一个运算符系列中包含的不同类型的值时，优化器可以避免强制类型转换。

## 2.对新数据类型的索引支持


该文档提供了创建用于复数的新数据类型以及用于对该类型的值进行排序的运算符类的示例。本示例使用C语言，当速度至关重要时，这是完全合理的。但是，为了更好地理解比较语义，我们可以在同一实验中使用纯SQL。

让我们用两个字段创建一个新的复合类型：实部和虚部。

```
postgres=# create type complex as (re float, im float);
```


我们可以创建一个具有新类型字段的表，并向该表中添加一些值：

```
postgres=# create table numbers(x complex);

postgres=# insert into numbers values ((0.0, 10.0)), ((1.0, 3.0)), ((1.0, 1.0));
```

现在出现了一个问题：如果在数学意义上没有为复数定义任何顺序关系，该如何对它们进行排序？

事实证明，已经为我们定义了比较运算符：

```
postgres=# select * from numbers order by x;

   x    

--------

 (0,10)

 (1,1)

 (1,3)

(3 rows)
```


默认情况下，对于复合类型，排序是按组件方式进行的：首先比较第一个字段，然后比较第二个字段，依此类推，与逐个字符比较文本字符串的方式大致相同。但是我们可以定义不同的顺序。例如，复数可被视为向量，并按模（长度）排序，模的计算是坐标平方和的平方根（毕达哥拉斯定理）。为了定义这样的顺序，让我们创建一个辅助函数来计算模量：

```
postgres=# create function modulus(a complex) returns float as $$

    select sqrt(a.re*a.re + a.im*a.im);

$$ immutable language sql;
```


现在，我们将使用此辅助函数系统地定义所有五个比较运算符的函数：

```
postgres=# create function complex_lt(a complex, b complex) returns boolean as $$

    select modulus(a) < modulus(b);

$$ immutable language sql;

 

postgres=# create function complex_le(a complex, b complex) returns boolean as $$

    select modulus(a) <= modulus(b);

$$ immutable language sql;

 

postgres=# create function complex_eq(a complex, b complex) returns boolean as $$

    select modulus(a) = modulus(b);

$$ immutable language sql;

 

postgres=# create function complex_ge(a complex, b complex) returns boolean as $$

    select modulus(a) >= modulus(b);

$$ immutable language sql;

 

postgres=# create function complex_gt(a complex, b complex) returns boolean as $$

    select modulus(a) > modulus(b);

$$ immutable language sql;
```


我们将创建相应的运算符。为了说明它们不需要被称为“>”，“ <”等，让我们给它们加上«weird»奇怪的名称。

```
postgres=# create operator #<#(leftarg=complex, rightarg=complex, procedure=complex_lt);

 

postgres=# create operator #<=#(leftarg=complex, rightarg=complex, procedure=complex_le);

 

postgres=# create operator #=#(leftarg=complex, rightarg=complex, procedure=complex_eq);

 

postgres=# create operator #>=#(leftarg=complex, rightarg=complex, procedure=complex_ge);

 

postgres=# create operator #>#(leftarg=complex, rightarg=complex, procedure=complex_gt);


在这一点上，我们可以比较数字：

postgres=# select (1.0,1.0)::complex #<# (1.0,3.0)::complex;

 ?column?

----------

 t

(1 row)
```


除了五个运算符之外，«btree»访问方法还需要定义一个函：如果第一个值小于，等于或大于第二个，那么它必须返回-1、0或1。此辅助功能称为support支持。其他访问方法可能需要定义其他支持函数。

```
postgres=# create function complex_cmp(a complex, b complex) returns integer as $$

    select case when modulus(a) < modulus(b) then -1

                when modulus(a) > modulus(b) then 1 

                else 0

           end;

$$ language sql;
```

现在，我们准备创建一个运算符类（自动创建同名运算符族）：

```
postgres=# create operator class complex_ops

default for type complex

using btree as

    operator 1 #<#,

    operator 2 #<=#,

    operator 3 #=#,

    operator 4 #>=#,

    operator 5 #>#,

    function 1 complex_cmp(complex,complex);
```


现在可以按需要进行排序：

```
postgres=# select * from numbers order by x;

   x    

--------

 (1,1)

 (1,3)

 (0,10)

(3 rows)
```


肯定会得到«btree»索引的支持。

要完成图片，您可以使用以下查询获取支持函数：

```
postgres=# select amp.amprocnum,

       amp.amproc,

       amp.amproclefttype::regtype,

       amp.amprocrighttype::regtype

from   pg_opfamily opf,

       pg_am am,

       pg_amproc amp

where  opf.opfname = 'complex_ops'

and    opf.opfmethod = am.oid

and    am.amname = 'btree'

and    amp.amprocfamily = opf.oid;

 amprocnum |   amproc    | amproclefttype | amprocrighttype

-----------+-------------+----------------+-----------------

         1 | complex_cmp | complex        | complex

(1 row)
```

# 六、内部结构


我们可以使用«pageinspect»扩展来探索B树的内部结构。

```
demo=# create extension pageinspect;


索引元页：

demo=# select * from bt_metap('ticket_flights_pkey');

 magic  | version | root | level | fastroot | fastlevel

--------+---------+------+-------+----------+-----------

 340322 |       2 |  164 |     2 |      164 |         2

(1 row)
```


这里最有趣的是索引级别：一百万行的表在两列上的索引最少需要2个级别（不包括根）。

有关块164的统计信息（根）：

```
demo=# select type, live_items, dead_items, avg_item_size, page_size, free_size

from bt_page_stats('ticket_flights_pkey',164);

 type | live_items | dead_items | avg_item_size | page_size | free_size

------+------------+------------+---------------+-----------+-----------

 r    |         33 |          0 |            31 |      8192 |      6984

(1 row)


块中的数据（这里牺牲了屏幕宽度的«data»字段，包含二进制表示的索引键的值）：

demo=# select itemoffset, ctid, itemlen, left(data,56) as data

from bt_page_items('ticket_flights_pkey',164) limit 5;

 itemoffset |  ctid   | itemlen |                           data                           

------------+---------+---------+----------------------------------------------------------

          1 | (3,1)   |       8 |

          2 | (163,1) |      32 | 1d 30 30 30 35 34 33 32 33 30 35 37 37 31 00 00 ff 5f 00

          3 | (323,1) |      32 | 1d 30 30 30 35 34 33 32 34 32 33 36 36 32 00 00 4f 78 00

          4 | (482,1) |      32 | 1d 30 30 30 35 34 33 32 35 33 30 38 39 33 00 00 4d 1e 00

          5 | (641,1) |      32 | 1d 30 30 30 35 34 33 32 36 35 35 37 38 35 00 00 2b 09 00

(5 rows)
```


第一个元素与技术有关，并指定块中所有元素的上限（我们未讨论的实现细节），而数据本身以第二个元素开头。显然，最左边的子节点是块163，然后是块323，依此类推。反过来，可以使用相同的函数进行探索。

现在，遵循一个良好的传统，阅读文档，README（ https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/nbtree/README;hb=HEAD ） 和源代码。

还有一个更有用的扩展是“ amcheck ”，它将整合到PostgreSQL 10中，对于较低版本，可以从github（ https://github.com/petergeoghegan/amcheck ）获得。此扩展检查B树中数据的逻辑一致性，并使我们能够提前检测到故障。

确实如此，«amcheck»是PostgreSQL从版本10开始的一部分。

# 英文原文：
https://habr.com/en/company/postgrespro/blog/443284/