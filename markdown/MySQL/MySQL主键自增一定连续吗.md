# MySQL 主键自增一定连续吗？

InnoDB 存储引擎会为表自动创建聚簇索引，聚簇索引是按照主键值的大小排序，可以在插入和按主键查找时提高速度

下面先介绍一下自增主键的基本内容。我们在创建表的时候可以设置主键自增，建表语句如下：

```mysql
create table `test_increment` (
	`id` int(11) not null auto_increment,
	`a` int(11) default null,
	`b` int(11) default null,
	primary key (`id`),
	unique key (`a`)
) engine=innodb;
```

那么自增的初始值是多少？自增的步长又是多少？这两个信息存储在系统变量中，可以通过下面命令查看：

```mysql
mysql> SHOW VARIABLES LIKE 'auto_inc%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| auto_increment_increment | 1     |
| auto_increment_offset    | 1     |
+--------------------------+-------+
2 rows in set (0.01 sec)
```

在返回结果中，auto_increment_offset 表示自增初始值；auto_increment_increment 表示自增步长

如果现在插入一条没有设置主键的记录，执行下面语句：

```mysql
insert into `test_increment` values (null, 1, 1);
```

然后查看表结构：

```mysql
mysql> show create table `test_increment`;
+----------------+--------------------------------------------------------------------------+
| Table          | Create Table                                                             |
+----------------+--------------------------------------------------------------------------+
| test_increment | CREATE TABLE `test_increment` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `a` (`a`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 |
+----------------+--------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

可以看到`AUTO_INCREMENT=2`，表示下一个插入元素的自增主键值为 2，因为值 1 已经给上面插入的那条记录

我们可以通过下面命令修改自增初始值和自增步长：

```mysql
SET @@auto_increment_offset = 4;
SET @@auto_increment_increment = 3;
```

### <font color=#1FA774>自增主键信息存储位置</font>

在 MyISAM 存储引擎中，自增值保存在数据文件中

在 InnoDB 存储引擎中，自增值保存在内存中，并没有持久化，当服务器重启后，会去表中找最大的主键值`max(id)`，然后将自增值设置为`max(id) + 1`

下面做个实验，先向表中插入两条数据：

```mysql
insert into `test_increment` values (null, 1, 1);
insert into `test_increment` values (null, 2, 2);
```

然后删除第二条数据：

```mysql
delete from `test_increment` where `id` = 2;
```

然后新插入一条数据：

```mysql
insert into `test_increment` values (null, 3, 3);
```

最后查看表中所有记录：

```mysql
mysql> select * from `test_increment`;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 |    1 |    1 |
|  3 |    3 |    3 |
+----+------+------+
2 rows in set (0.00 sec)
```

从返回结果中可以看出，第三条数据主键的值为 3，也就是自增过程不会回头。当我们把第二条数据删除，虽然主键的值 2 没有记录使用，但是自增过程是一路向前

如果在插入第三条数据之前将 MySQL 服务重启，那么结果就会发生变化：

```mysql
mysql> select * from `test_increment`;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 |    1 |    1 |
|  2 |    3 |    3 |
+----+------+------+
2 rows in set (0.00 sec)
```

当我们在插入数据时已经设置了主键的值：

```java
insert into `test_increment` values (4, 4, 4);
```

那么有下面两种情况 (insert_num 表示设置主键值，auto_increment 表示自增值)：

- 如果`insert_num < auto_increment`，那么自增值不变
- 如果`insert_num >= auto_increment`，那么把当前自增值修改为新的自增值

**<font color='red'>小技巧：</font>**通过查看表结构可以知道下一个自增主键的值

```mysql
mysql> show create table `test_increment`;
+----------------+--------------------------------------------------------------------------+
| Table          | Create Table                                                             |
+----------------+--------------------------------------------------------------------------+
| test_increment | CREATE TABLE `test_increment` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `a` (`a`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8 | # AUTO_INCREMENT=3 表示下一个自增主键的值为 3
+----------------+--------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

### <font color=#1FA774>自增值不连续的场景</font>

#### <font color=#9933FF>自增步长不为 1</font>

文章开头介绍可以修改自增初始值以及自增步长，在分布式系统中，往往为了避免两个库生成的主键发生冲突，使一个库自增主键为奇数，另一个库自增主键为偶数

在这种场景中，主键的值肯定不是连续的！

#### <font color=#9933FF>唯一键冲突插入失败</font>

根据文章开头的建表语句可以看出在`a`上添加了一个唯一索引，所以表中记录的`a`不允许重复

如果我们向表中插入一条新的记录：

```mysql
insert into `test_increment` values (null, 1, 4);
```

由于`a`重复会插入失败，但其实执行时将语句替换成下面的语句，其中主键的值 4 是根据自增主键自动算出来的

```mysql
insert into `test_increment` values (4, 1, 4);
```

虽然插入失败时，但是主键的值 4 已经被消耗，无法撤回，所以会导致表中没有`id = 4`的记录

#### <font color=#9933FF>批量插入</font>

当我们批量插入时：

```mysql
# 向表 test_increment 插入五条数据
insert into `test_increment`(a,b) values (1, 1), (2, 2), (3, 3), (4, 4), (5, 5);
# copy 一份表 test_increment
create table `test_increment_copy` like `test_increment`;
# 批量插入
insert into `test_increment_copy`(a, b) select a, b from `test_increment`;
# 新插入一条数据
insert into `test_increment_copy` values (null, 6, 6);
```

然后来看一下表`test_increment_copy`中的数据：

```mysql
mysql> select * from test_increment_copy;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 |    1 |    1 |
|  2 |    2 |    2 |
|  3 |    3 |    3 |
|  4 |    4 |    4 |
|  5 |    5 |    5 |
|  8 |    6 |    6 |
+----+------+------+
6 rows in set (0.01 sec)
```

看出看到最后一条记录的主键不是 6，而是 8。这是因为批量插入的时候，并不是每次申请一个自增值，而是递增的申请

- 第一次申请 1 个值：id = 1
- 第二次申请 2 个值：id = 2，id = 3
- 第三次申请 4 个值：id = 4，id = 5，id = 6，id = 7

所以上面五条记录需要申请三次，最后`AUTO_INCREMENT=8`

#### <font color=#9933FF>事务回滚</font>

假设事务 A 插入了一条记录，主键为 1；事务 B 插入了一条记录，主键为 2，此时`AUTO_INCREMENT=3`。最后我们将事务 A 回滚，但是自增值不会回滚，否则就会出现主键冲突

如果自增值也回滚，那么``AUTO_INCREMENT=1`，如果后续后执行其它事务就会申请到自增值 2，导致主键冲突。所以一般事务回滚时，自增值不回滚