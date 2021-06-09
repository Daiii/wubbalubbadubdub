[toc]

## explain

MySQL官网：https://dev.mysql.com/doc/refman/5.7/en/explain.html

>When EXPLAIN is used with an explainable statement, MySQL displays information from the optimizer about the statement execution plan. That is, MySQL explains how it would process the statement, including information about how tables are joined and in which order.

explain是能解释MySQL如果处理SQL语句，表的加载顺序，表是如何连接，以及索引使用情况。

**是SQL优化的重要工具。**

### explain详解

![explain字段](images/explain1.png)

**概要描述**：

* id:选择标识符
* select_type:表示查询的类型
* table:输出结果集的表
* partitions:匹配的分区
* type:表示表的连接类型
* possible_keys:表示查询时，可能使用的索引
* key:表示实际使用的索引
* key_len:索引字段的长度
* ref:列与索引的比较
* rows:扫描出的行数(估算的行数)
* filtered:按表条件过滤的行百分比
* Extra:执行情况的描述和说明

#### id

SELECT识别符，可以理解为SELECT的查询序列号。

**SQL执行顺序的标识，SQL从大到小顺序执行。**

1. id相同时，执行顺序由上至下。
2. 如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行。
3. id如果相同，可以理解为是一组，由上往下顺序执行；在所有组中，id值越大，优先级越高，越先被执行。

#### select_type

标识查询中每个select子局的类型

* SIMPLE(简单SELECT，不使用UNION或子查询等)
* PRIMARY(子查询中最外层查询，查询中若包含任何复杂的子部分，最外层的select被标记为PRIMARY)
* UNION(UNION中的第二个或后面的SELECT语句)
* DEPENDENT UNION(UNION中的第二个或后面的SELECT语句，取决于外面的查询)
* UNION RESULT(UNION的结果，union语句中第二个select开始后面所有select)
* SUBQUERY(子查询中的第一个SELECT，结果不依赖于外部查询)
* DEPENDENT SUBQUERY(子查询中的第一个SELECT，依赖于外部查询)
* DERIVED(派生表的SELECT, FROM子句的子查询)
* UNCACHEABLE SUBQUERY(一个子查询的结果不能被缓存，必须重新评估外链接的第一行)

#### table

显示数据来自于哪个表，有时不是真正的表(DERIVED)，虚拟(派生)表最后一位是数字，代表id为多少的查询。

#### type

可以理解为对表的访问方式，表示MySQL在表中如何找到所需行的方式，又称”访问类型“。

常用的类型有： **ALL、index、range、 ref、eq_ref、const、system、NULL（从左到右，性能从差到好）**

* **ALL**：Full Table Scan，遍历全表以找到匹配的行。
* **index**：Full Index Scan，遍历索引树。通常比ALL快，因为索引文件通常比数据文件小。all和index都是读全表，但index是从索引中检索的，而all是从硬盘中检索的。
* **range**：只检索给定范围内的行，一般条件查询中出现了>、<、in、between等查询。
* **ref**：非唯一行索引扫描，返回匹配某个单独值的所有行。
* **eq_ref**：类似于ref，区别就是使用的索引是唯一索引。一般是两表关联，关联条件中的字段是主键或唯一索引。
* **const**：表示通过索引一次就找到了对应的行。primary key或者unique索引。
* **system**：当表里只有一行数据的时候返回这个。
* **NULL**：MySQL在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成。

#### possible_keys

指出MySQL能使用哪个索引在表中找到数据，查询涉及到的字段上如果存在索引，则该索引将被列出，但不一定被查询使用(该查询可以利用到的索引，如果没有任何索引显示NULL)。

#### key

显示MySQL实际决定使用的索引，一定包含在possible_keys中。

#### key_len

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度(key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的)。

不损失精确性的情况下，长度越短越好 。

key_len计算规则：

* 字符串
  * char(n)：n字节长度
  * varchar(n)：如果是utf-8，则长度3n+2字节，加的2字节用来存储字符串长度
* 数值类型
  * tinyint：1字节
  * smallint：2字节
  * int：4字节
  * bigint：8字节
* 时间类型
  * date：3字节
  * timestamp：4字节
  * datetime：8字节
* 如果字段允许为NULL，需要一个字节记录是否为NULL

#### ref

列与索引的标记，表示上述表的连接匹配条件，即那些列或常量被用于查找索引上的值。

#### rows

估算出结果集行数，表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录要读取的行数。

#### filtered

表示选取的行和读取的行的百分比，百分比。

#### extra

一些重要的额外信息

1. Using filesort：使用外部的索引排序，而不是按照表内的索引顺序进行读取。
2. Using temporary：使用了临时表保存中间结果。常见于排序order by和group by。
3. Using index：表示select语句中使用了覆盖索引，直接从索引中取值，而不需要回行(从磁盘中取数据)。
4. Using where：使用了where过滤。
5. Using index condition：5.6之后新增的，表示查询的列有非索引的列，先判断索引的条件，以减少磁盘IO。
6. Using join buffer：使用了连接缓存。
7. impos where：where子句的值总是false。