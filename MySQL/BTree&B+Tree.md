## explain

> 概要描述：
>
> id:选择标识符
> select_type:表示查询的类型。
> table:输出结果集的表
> partitions:匹配的分区
> type:表示表的连接类型
> possible_keys:表示查询时，可能使用的索引
> key:表示实际使用的索引
> key_len:索引字段的长度
> ref:列与索引的比较
> rows:扫描出的行数(估算的行数)
> filtered:按表条件过滤的行百分比
> Extra:执行情况的描述和说明

### id

SELECT识别符。这是SELECT的查询序列号。

**个人理解是SQL执行的顺序标识，SQL从大到小执行。**

1. id相同时，执行顺序由上至下。
2. 如果是子查询，id序号会递增，id值越大优先级越高，越先被执行
3. id如果相同，可以认为是同一组，从上往下顺序执行；在所有组中，id值越大，优先级越高

### select_type

表示查询的类型

1. SIMPLE(加单查询，不使用UNION或子查询)
2. PRIMARY(子查询中最外层的查询)
3. UNION(UNION中第二个或者后面的SELECT语句)
4. DEPENDENT UNION