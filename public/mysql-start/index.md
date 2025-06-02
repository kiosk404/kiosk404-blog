# MySQL 基本入门






最近一段时间，需要做一个 信息管理系统，所以涉及到很多 数据库相关的知识，这里就专门恶补一下。之前有用到过 MySQL，但是用法太过基础，没有系统的去学习。这里参照着 《MySQL必知必会》做一些总结。

另外 MySQL 的一些基本操作，可以参考 : [MySQL教程](https://www.yiibai.com/mysql)

# 主键

在mysql中，主键全称“主键约束”，是一个列或多列的组合，**其值能唯一地标识表中的每一行**。主键的作用是确定该数据的唯一性。主要用于和其他表的外键进行关联，以及本记录的修改与删除。

```mysql
> CREATE TABLE tb_emp ( 
    id INT(11) PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(25),
    salary FLOAT
);
Query OK, 0 rows affected
Time: 0.047s

> DESC tb_emp;
+--------+-------------+------+-----+---------+----------------+
| Field  | Type        | Null | Key | Default | Extra          |
+--------+-------------+------+-----+---------+----------------+
| id     | int         | NO   | PRI | <null>  | auto_increment |
| name   | varchar(25) | YES  |     | <null>  |                |
| salary | float       | YES  |     | <null>  |                |
+--------+-------------+------+-----+---------+----------------+
3 rows in set
Time: 0.006s

```

主键一般使用自增字段做主键，但是如果有多台服务器，可能会有主键重复的问题，需要考虑手动赋值的方式。

<br/>

# 外键和连接

**外键就是从表中用来引用主表中数据的那个公共字段**。在关联关系 R 中，公众字段（字段 A）是表 A 的主键，所以表 A 是主表，表 B 是从表。表 B 中的公共字段（字段 A）是外键。

<img src="https://img1.kiosk007.top/static/images/blog/mysql1.webp" alt="mysql1" style="zoom:13%;display:block;margin:auto;clear:both" />

在 MySQL 中，外键是通过外键约束来定义的。外键约束就是约束的一种，它必须在从表中定义，包括指明哪个是外键字段，以及外键字段所引用的主表中的主键字段是什么。

```mysql
CREATE TABLE 从表名
(
  字段名 类型,
  ...
-- 定义外键约束，指出外键字段和参照的主表字段
CONSTRAINT 外键约束名
FOREIGN KEY (字段名) REFERENCES 主表名 (字段名)
)
```

如下所示，创建外键约束，先创建主表

```mysql
CREATE TABLE demo.importhead (
    listnumber INT PRIMARY KEY,
    supplierid INT,
    stocknumber INT,
    importtype INT,
    importquantity DECIMAL(10 , 3 ),
    importvalue DECIMAL(10 , 2 ),
    recorder INT,
    recordingdate DATETIME
);
```

再创建从表

```mysql

CREATE TABLE demo.importdetails
(
  listnumber INT,
  itemnumber INT,
  quantity DECIMAL(10,3),
  importprice DECIMAL(10,2),
  importvalue DECIMAL(10,2),
  -- 定义外键约束，指出外键字段和参照的主表字段
  CONSTRAINT fk_importdetails_importhead
  FOREIGN KEY (listnumber) REFERENCES importhead (listnumber)
);
```

创建好了外键，关联关系建立后，需要通过连接查询，这时需要用到连接，MySQL中有 **内连接**（**INNER JOIN**）和外连接（**OUTER JOIN**）。

- 内连接表示查询结果只返回符合连接条件的记录，这种连接方式比较常用；
- 外连接则表示查询结果返回一个表中的所有记录，以及另一个表中满足连接条件的记录。

如下是一条内连接查询。

```mysql

mysql> SELECT
    ->     a.transactionno,
    ->     a.itemnumber,
    ->     a.quantity,
    ->     a.price,
    ->     a.transdate,
    ->     b.membername
    -> FROM
    ->     demo.trans AS a
    ->         JOIN
    ->     demo.membermaster AS b ON (a.cardno = b.cardno);
+---------------+------------+----------+-------+---------------------+------------+
| transactionno | itemnumber | quantity | price | transdate           | membername |
+---------------+------------+----------+-------+---------------------+------------+
|             1 |          1 |    1.000 | 89.00 | 2020-12-01 00:00:00 | 张三       |
+---------------+------------+----------+-------+---------------------+------------+
1 row in set (0.00 sec)
```

<br/>

# 时间函数

在MySQL中，时间函数是用来处理时间的，比如

1. 统计一天之中不同时段的销售情况，就要获取时间值中的小时值，这里会用到函数 HOUR()，类似的还有 YEAR() 、MONTH()、Day()、HOUR()、MINUTE()、SECOND()
2. 要计算与去年同期相比的增长率，就要用到函数 DATE_ADD()

```mysql
-- 用 DATE_ADD 函数，获取到 2020 年 12 月 10 日上一年的日期：2019 年 12 月 10 日。
mysql> SELECT DATE_ADD('2020-12-10', INTERVAL - 1 YEAR);
+-------------------------------------------+
| DATE_ADD('2020-12-10', INTERVAL - 1 YEAR) |
+-------------------------------------------+
| 2019-12-10                                |
+-------------------------------------------+
1 row in set (0.00 sec)

-- 获取去年的当月的月初
mysql> SELECT DATE_ADD(LAST_DAY(DATE_ADD(DATE_ADD('2020-12-10', INTERVAL - 1 YEAR),INTERVAL - 1 MONTH)),INTERVAL 1 DAY);
+-----------------------------------------------------------------------------------------------------------+
| DATE_ADD(LAST_DAY(DATE_ADD(DATE_ADD('2020-12-10', INTERVAL - 1 YEAR),INTERVAL - 1 MONTH)),INTERVAL 1 DAY) |
+-----------------------------------------------------------------------------------------------------------+
| 2019-12-01                                                                                                |
+-----------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

3. 要计算今天是周几，需要用到函数 DAYOFWEEK() 

4. 获取时间中的小时、分钟等值，用到函数 EXTRACT()

```mysql

mysql> SELECT
    -> EXTRACT(HOUR FROM b.transdate) AS 时段,
    -> SUM(a.quantity) AS 数量,
    -> SUM(a.salesvalue) AS 金额
    -> FROM
    -> demo.transactiondetails a
    -> JOIN
    -> demo.transactionhead b ON (a.transactionid = b.transactionid)
    -> GROUP BY EXTRACT(HOUR FROM b.transdate)
    -> ORDER BY EXTRACT(HOUR FROM b.transdate);
+------+--------+--------+
| 时段 | 数量   | 金额   |
+------+--------+--------+
|    9 | 16.000 | 500.00 |
|   10 | 11.000 | 139.00 |
|   11 | 10.000 |  30.00 |
|   12 | 40.000 | 200.00 |
|   13 |  5.000 | 445.00 |
|   15 |  6.000 |  30.00 |
|   17 |  1.000 |   3.00 |
|   18 |  2.000 | 178.00 |
|   19 |  2.000 |   6.00 |
+------+--------+--------+
9 rows in set (0.00 sec)
```



详细的时间函数可以参考官方连接：[https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html#function_date-format](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html#function_date-format)

<br/>

# 索引

索引是为了提升查询速度，就和一本书的目录一样。MySQL 支持单字段索引和组合索引，而单字段索引比较常用。

创建索引可以在建表时添加索引。也可以给已经存在的表添加索引。

```mysql
-- 建表时添加索引
CREATE TABLE `t_user` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `id_no` varchar(18) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin DEFAULT NULL COMMENT '身份编号',
  `username` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin DEFAULT NULL COMMENT '用户名',
  `age` int(11) DEFAULT NULL COMMENT '年龄',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  KEY `union_idx` (`id_no`,`username`,`age`),
  KEY `create_time_idx` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;

-- 已存在表创建索引
CREATE INDEX 索引名 ON TABLE 表名 (字段);
```

上述表结构中有三个索引：

- *id*：数据库主键索引
- *union_idx*：id_no、username、age 构成的联合索引
- *create_time_idx*：由create_time构成的普通索引

<br/>

要知道索引是怎么起作用的，我们需要借助 MySQL 中的 EXPLAIN 这个关键字。EXPLAIN 关键字能够查看

```mysql
explain select * from t_user where id_no = '1002';
+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+--------+
| id | select_type | table  | partitions | type | possible_keys | key       | key_len | ref   | rows | filtered | Extra  |
+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+--------+
| 1  | SIMPLE      | t_user | <null>     | ref  | union_idx     | union_idx | 75      | const | 1    | 100.0    | <null> |
+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+--------+

```

- table=t_user：显示这一行的数据是关于哪张表的。
- type=ref：这列最重要，显示了连接使用了哪种类别，有无使用索引，是使用 expalin 命令分析瓶颈的关键项之一。

> 结果值从好到坏依次是：
> system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL
> 也就是说 type 记录了是否使用了索引还是全表扫描，const, eg_reg, ref, range, index, ALL
> 一般来说，得保证查询至少达到range级别，最好能达到ref，否则就可能会出现性能问题。

- possible_key=union_idx：表示可以选择的索引是 union_idx 这个联合索引。
- key=union_idx：表示实际选择的索引是 union_idx 
- key_len=75：MySQL 决定使用的键长度，在不损失精确性的情况下，长度越短越好。
- ref=const：显示使用哪个列或常数与key一起从表中选择行。

- rows=1：表示需要读取的记录数
- filtered：
- extra=NULL：包含MySQL解决查询的详细信息，也是关键参考项之一，显示了查询中MySQL的附加信息，关注 Using filesort 和 Using temporary。

<br/>

**组合索引**

组合索引的多个字段是有序的，遵循左对齐的原则。比如我们创建的组合索引，排序的方式是 id_no、username、age。因此、筛选的条件也需要遵守从左到右，如果中断，那么断点后的条件就没有办法利用索引了。比如 `id_no=1001 AND username=TOM1 AND age=11`

如果只用组合索引的一部分，那效果就没有单字段索引的效果那么好了。

<br/>

> 本节参考：[15 个必知的MySQL索引失效的场景，别再踩坑了](https://cloud.tencent.com/developer/article/1992920)

<br/>

# 触发器

实际的场景中，经常会遇到这样的情况，有2个或者多个相互关联的表，如商品信息和库存信息是分别存放在2个不同的数据表中，当新加入一条商品记录的时候，为了保证数据的完整性，必须同时在库存表中添加一条库存记录。

这样就需要维护两个一起触发的操作。很容易漏掉其中一步导致数据的缺失。

这个时候，可以创建一个触发器，让商品信息数据的插入操作自动触发库存数据的插入操作。这样一来，就不需要担心忘记添加库存数据而导致数据丢失。

```mysql
-- 创建触发器的语法结构是
CREATE TRIGGER 触发器名称 {BEFORE|AFTER} {INSERT|UPDATE|DELETE}
ON 表名 FOR EACH ROW 表达式；

-- 查看触发器的语句是：
SHOW TRIGGERS\G;

-- 删除触发器的语法结构是：
DROP TRIGGER 触发器名称;
```

- 表名：表示触发器监控的对象
- INSERT|UPDATE|DELETE：表示触发的事件。INSERT 表示插入记录时触发；UPDATE 表示更新记录时触发；DELETE 表示删除记录时触发。
- BEFORE|AFTER：表示触发的时间。BEFORE 表示在事件之前触发；AFTER 表示在事件之后触发。

实际小案例：超市会员的会员卡余额变动的明细，我们改变会员卡余额时总是忘去记录这笔交易，这个操作其实可以写到触发器中。

1. 查询出编号为 2 的会员卡的储蓄金额为 200.

```mysql

mysql> SELECT memberdeposit
-> FROM demo.membermaster
-> WHERE memberid = 2;
+---------------+
| memberdeposit |
+---------------+
| 200.00 |
+---------------+
1 row in set (0.00 sec)
```

2. 该会员的储蓄值减去150 

```mysql
mysql> UPDATE demo.membermaster
-> SET memberdeposit = memberdeposit - 150
-> WHERE memberid = 2;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1 Changed: 1 Warnings: 0
```

3. 读出会员编号2当前的生于储蓄金额

```mysql

mysql> SELECT memberdeposit
-> FROM demo.membermaster
-> WHERE memberid = 2;
+---------------+
| memberdeposit |
+---------------+
| 50.00 |
+---------------+
1 row in set (0.00 sec)
```

4. 把会员编号和前面查询中获得的储蓄值的起始金额、储值余额、金额变化，写入该会员储蓄值历史表中

```mysql
mysql> INSERT INTO demo.deposithist
-> (
-> memberid,
-> transdate,
-> oldvalue,
-> newvalue,
-> changedvalue
-> )
-> SELECT 2,NOW(),200,50,-150;
Query OK, 1 row affected (0.02 sec)
Records: 1 Duplicates: 0 Warnings: 0
```

5. 这样，我们就完成了记录会员储蓄值金额的变动操作。

```mysql
mysql> SELECT *
-> FROM demo.deposithist;
+----+----------+---------------------+----------+----------+--------------+
| id | memberid | transdate | oldvalue | newvalue | changedvalue |
+----+----------+---------------------+----------+----------+--------------+
| 1 | 2 | 2020-12-20 10:37:51 | 200.00 | 50.00 | -150.00 |
+----+----------+---------------------+----------+----------+--------------+
1 row in set (0.00 sec)
```

其实这4个独立的操作语句就可以改成触发器的模式

```mysql

DELIMITER //
CREATE TRIGGER demo.upd_membermaster BEFORE UPDATE  -- 在更新前触发
ON demo.membermaster
FOR EACH ROW                              -- 表示每更新一条记录，触发一次
BEGIN                                     -- 开始程序体
IF (new.memberdeposit <> old.memberdeposit)  -- 如果储值金额有变化
THEN
INSERT INTO demo.deposithist
(
memberid,
transdate,
oldvalue,
newvalue,
changedvalue
)
SELECT
NEW.memberid,
NOW(),
OLD.memberdeposit,                  -- 更新前的储值金额
NEW.memberdeposit,                  -- 更新后的储值金额
NEW.memberdeposit-OLD.memberdeposit; -- 储值金额变化值
END IF;
END
//
DELIMITER ;
```




