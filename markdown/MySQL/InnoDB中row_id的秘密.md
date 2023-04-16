# InnoDB 中 row_id 的秘密

本篇文章来讲讲 InnoDB 中主键自增的策略！！

### <font color=#1FA774>行格式</font>

在 InnoDB 存储引擎下，对于每一条记录来说，除了存储用户定义的一些属性字段外，还会默认添加一些「隐藏列」

InnoDB 的行格式如下图所示：

![11](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220717/1729521658050192kslQcB11.svg)

关于「隐藏列」的详细属性如下表所示：

|     列名     | 是否必需 | 占用空间 |          描述           |
| :----------: | :------: | :------: | :---------------------: |
|    row_id    |    否    |  6 字节  | 行 ID，唯一标识一条记录 |
|    trx_id    |    是    |  6 字节  |         事务 ID         |
| roll_pointer |    是    |  7 字节  |        回滚指针         |

本篇文章主要关注`row_id`字段，至于其它字段后续可能会总结！！

### <font color=#1FA774>row_id</font>

有人可能会有疑问，表中都有用户自定义的「主键」了，为啥 InnoDB 还要搞一个`row_id`出来

但是存在一种情况，用户没有自定义「主键」，这时如何唯一确定一条记录呢？是不是很不方便，所以才有了隐藏列`row_id`

我们可以看到`row_id`是非必需的，这表示如果表中存在可以唯一标识一条记录的列，那么就不需要`row_id`了

其实这样的表述也不是很严谨，我们来看看官方文档是怎么说的：

> If a table has a **<font color='red'>PRIMARY KEY</font>** or **<font color='red'>UNIQUE NOT NULL</font>** index that consists of a single column that has an integer type, you can use **<font color='red'>_rowid</font>** to refer to the indexed column in **<font color='red'>SELECT</font>** statements

**解释一下：**如果在表中存在主键或非空唯一索引，并且仅由一个整数类型的列构成，那么就可以使用`SELECT`语句直接查询`_rowid`，并且这个`_rowid`的值会引用该索引列的值

着重看一下文档中提到的几个关键字：**<font color='red'>主键</font>**、**<font color='red'>唯一索引</font>**、**<font color='red'>非空</font>**、**<font color='red'>单独一列</font>**、**<font color='red'>数值类型</font>**。接下来我们就要从这些角度入手，探究一下神秘的隐藏字段`_rowid`

#### <font color=#9933FF>存在主键且为数值类型</font>

使用下面的语句建表：

```mysql
CREATE TABLE `table1` (
  `id` int(11) NOT NULL PRIMARY KEY,
  `name` varchar(32) DEFAULT NULL
) ENGINE=InnoDB;
```

插入三条测试数据后，执行下面的查询语句，在`select`查询语句中直接查询`_rowid`：

```mysql
select *, _rowid from table1;
```

查看执行结果，`_rowid`可以被正常查询：

```mysql
+----+------+--------+
| id | name | _rowid |
+----+------+--------+
|  1 | zs   |      1 |
|  2 | ls   |      2 |
|  3 | ww   |      3 |
+----+------+--------+
3 rows in set (0.00 sec)
```

可以看到在设置了主键，并且主键字段是数值类型的情况下，`_rowid`直接引用了主键字段的值。对于这种可以被`select`语句查询到的的情况，可以将其称为**<font color='red'>显式</font>**的`rowid`

#### <font color=#9933FF>存在主键不为数值类型</font>

由于主键必定是非空字段，所以直接看存在主键不为数值类型，使用下面的语句建表：

```mysql
CREATE TABLE `table2` (
  `id` varchar(20) NOT NULL PRIMARY KEY,
  `name` varchar(32) DEFAULT NULL
) ENGINE=InnoDB;
```

查看执行结果，`_rowid`**<font color='red'>不</font>**可以被正常查询：

```mysql
ERROR 1054 (42S22): Unknown column '_rowid' in 'field list'
```
#### <font color=#9933FF>无主键但存在非空唯一索引</font>

上面对两种类型的主键进行了测试后，接下来我们看一下当表中没有主键、但存在唯一索引的情况。首先测试非空唯一索引加在数值类型字段的情况，建表语句如下：

```mysql
CREATE TABLE `table3` (
  `id` int(11) NOT NULL UNIQUE KEY,
  `name` varchar(32)
) ENGINE=InnoDB;
```

查看执行结果，`_rowid`可以被正常查询：

```mysql
+----+------+--------+
| id | name | _rowid |
+----+------+--------+
|  1 | zs   |      1 |
|  2 | ls   |      2 |
|  3 | ww   |      3 |
+----+------+--------+
3 rows in set (0.00 sec)
```

#### <font color=#9933FF>无主键但存在可空唯一索引</font>

唯一索引与主键不同的是，唯一索引所在的字段可以为`NULL`，在上面的`table3`中，在唯一索引所在的列上添加了`NOT NULL`非空约束，如果我们把这个非空约束删除掉，还能显式地查询到`_rowid`吗？下面再创建一个表，不同是在唯一索引所在的列上，不添加非空约束：

```mysql
CREATE TABLE `table4` (
  `id` int(11) UNIQUE KEY,
  `name` varchar(32)
) ENGINE=InnoDB;
```

查看执行结果，`_rowid`**<font color='red'>不</font>**可以被正常查询：

```mysql
ERROR 1054 (42S22): Unknown column '_rowid' in 'field list'
```

#### <font color=#9933FF>无主键但存在非空且非数值唯一索引</font>

和主键类似的，我们再对唯一索引被加在非数值类型的字段的情况进行测试。下面在建表时将唯一索引添加在字符类型的字段上，并添加非空约束：

```mysql
CREATE TABLE `table5` (
  `id` varchar(20),
  `name` varchar(32) NOT NULL PRIMARY KEY
) ENGINE=InnoDB;
```

查看执行结果，`_rowid`**<font color='red'>不</font>**可以被正常查询：

```mysql
ERROR 1054 (42S22): Unknown column '_rowid' in 'field list'
```

#### <font color=#9933FF>存在联合主键或联合唯一索引</font>

在上面的测试中，我们都是将主键或唯一索引作用在单独的一列上，那么如果使用了联合主键或联合唯一索引时，结果会如何呢？还是先看一下官方文档中的说明：

> **<font color='red'>_rowid</font>** refers to the **<font color='red'>PRIMARY KEY</font>** column if there is a **<font color='red'>PRIMARY KEY</font>** consisting of a single integer column. If there is a **<font color='red'>PRIMARY KEY</font>** but it does not consist of a single integer column, **<font color='red'>_rowid</font>** cannot be used.

简单来说就是，如果主键存在、且仅由数值类型的一列构成，那么`_rowid`的值会引用主键。如果主键是由多列构成，那么`_rowid`将不可用

根据这一描述，我们测试一下联合主键的情况，下面将两列数值类型字段作为联合主键建表：

```mysql
CREATE TABLE `table6` (
  `id` int(11) NOT NULL,
  `no` int(11) NOT NULL,
  `name` varchar(32),
  PRIMARY KEY(`id`,`no`)
) ENGINE=InnoDB;
```
查看执行结果，`_rowid`**<font color='red'>不</font>**可以被正常查询：

```mysql
ERROR 1054 (42S22): Unknown column '_rowid' in 'field list'
```

同样，这一理论也可以作用于唯一索引，如果非空唯一索引不是由单独一列构成，那么也无法直接查询得到`_rowid`

#### <font color=#9933FF>存在多个唯一索引</font>

在 MySQL 中，每张表只能存在一个主键，但是可以存在多个唯一索引。那么如果同时存在多个符合规则的唯一索引，会引用哪个作为`_rowid`的值呢？老规矩，还是看官方文档的解答：

> Otherwise, **<font color='red'>_rowid</font>** refers to the column in the first **<font color='red'>UNIQUE NOT NULL</font>** index if that index consists of a single integer column. If the first **<font color='red'>UNIQUE NOT NULL</font>** index does not consist of a single integer column, **<font color='red'>_rowid</font>** cannot be used.

简单翻译一下，如果表中的第一个非空唯一索引仅由一个整数类型字段构成，那么`_rowid`会引用这个字段的值。否则，如果第一个非空唯一索引不满足这种情况，那么`_rowid`将不可用

在下面的表中，创建两个都符合规则的唯一索引：

```mysql
CREATE TABLE `table7` (
  `id` int(11) NOT NULL,
  `no` int(11) NOT NULL,
  `name` varchar(32),
  UNIQUE KEY(no),
  UNIQUE KEY(id)
) ENGINE=InnoDB;
```
查看执行结果，`_rowid`可以被正常查询：

```mysql
+----+----+------+--------+
| id | no | name | _rowid |
+----+----+------+--------+
|  1 |  1 | zs   |      1 |
|  2 |  2 | ls   |      2 |
|  3 |  3 | ww   |      3 |
+----+----+------+--------+
3 rows in set (0.00 sec)
```

可以看到`_rowid`的值与`no`这一列的值相同，证明了`_rowid`会严格地选取第一个创建的唯一索引作为它的引用。

那么，如果表中创建的第一个唯一索引不符合`_rowid`的引用规则，第二个唯一索引满足规则，这种情况下，`_rowid`可以被显示地查询吗？针对这种情况我们建表如下，表中的第一个索引是联合唯一索引，第二个索引才是单列的唯一索引情况，再来进行一下测试：

```mysql
CREATE TABLE `table8` (
  `id` int(11) NOT NULL,
  `no` int(11) NOT NULL,
  `name` varchar(32),
  UNIQUE KEY `index1`(`id`,`no`),
  UNIQUE KEY `index2`(`id`)
) ENGINE=InnoDB;
```
查看执行结果，`_rowid`**<font color='red'>不</font>**可以被正常查询：

```mysql
ERROR 1054 (42S22): Unknown column '_rowid' in 'field list'
```

如果将上面创建唯一索引的语句顺序调换，那么将可以正常显式的查询到`_rowid`

#### <font color=#9933FF>同时存在主键与唯一索引</font>

从上面的例子中，可以看到唯一索引的**定义顺序**会决定将哪一个索引应用`_rowid`，那么当同时存在主键和唯一索引时，定义顺序会对其引用造成影响吗？

按照下面的语句创建两个表，只有创建主键和唯一索引的顺序不同：

```mysql
CREATE TABLE `table9` (
  `id` int(11) NOT NULL,
  `no` int(11) NOT NULL,
  PRIMARY KEY(id),
  UNIQUE KEY(no)
) ENGINE=InnoDB;

CREATE TABLE `table10` (
  `id` int(11) NOT NULL,
  `no` int(11) NOT NULL,
  UNIQUE KEY(id),
  PRIMARY KEY(no)
) ENGINE=InnoDB;
```

查看执行结果，`_rowid`可以被正常查询：

```mysql
+----+----+--------+
| id | no | _rowid |
+----+----+--------+
|  1 |  1 |      1 |
|  2 |  2 |      2 |
|  3 |  3 |      3 |
+----+----+--------+
3 rows in set (0.00 sec)
```

可以得出结论，当同时存在符合条件的主键和唯一索引时，无论创建顺序如何，`_rowid`都会优先引用主键字段的值

#### <font color=#9933FF>无符合条件的主键与唯一索引</font>

上面，我们把能够直接通过`select`语句查询到的称为**<font color='red'>显式</font>**的`_rowid`，在其他情况下虽然`_rowid`不能被**<font color='red'>显式</font>**查询，但是它也是一直存在的，这种情况我们可以将其称为**<font color='red'>隐式</font>**的`_rowid`

实际上，`innoDB`在没有默认主键的情况下会生成一个 6 字节长度的无符号数作为自动增长的`_rowid`，如本文最上方的图所示

因此最大为 $2^{48} - 1$，到达最大值后会从 0 开始计算。下面，我们创建一个没有主键与唯一索引的表，在这张表的基础上，探究一下隐式的`_rowid`

```mysql
CREATE TABLE `table11` (
  `id` int(11),
  `name` varchar(32)
) ENGINE=InnoDB;
```

在开始动手前，还需要做一点铺垫， 在`innoDB`中其实维护了一个全局变量`dictsys.row_id`，没有定义主键的表都会共享使用这个`row_id`，在插入数据时会把这个全局`row_id`当作自己的主键，然后再将这个全局变量加 1。

接下来我们需要用到`gdb`调试的相关技术，`gdb`是一个在Linux下的调试工具，可以用来调试可执行文件。在服务器上，先通过`yum install gdb`安装，安装完成后，通过下面的`gdb`命令 把 `row_id` 修改为 1：

```bash
gdb -p 14754 -ex 'p dict_sys->row_id=1' -batch
```

在空表中插入3行数据：

```mysql
INSERT INTO table11 VALUES (100000001, 'Hydra');
INSERT INTO table11 VALUES (100000002, 'Trunks');
INSERT INTO table11 VALUES (100000003, 'Susan');
```

查看表中的数据，此时对应的`_rowid`理论上是 1~3：

```mysql
+-----------+--------+
| id        | name   |
+-----------+--------+
| 100000001 | Hydra  |
| 100000002 | Trunks |
| 100000003 | Susan  |
+-----------+--------+
7 rows in set (0.01 sec)
```

然后通过`gdb`命令把`row_id`改为最大值 $2^{48}$，此时已超过`dictsys.row_id`最大值：

```bash
gdb -p 14754 -ex 'p dict_sys->row_id=281474976710656' -batch
```

再向表中插入三条数据：

```mysql
INSERT INTO table11 VALUES (100000004, 'King');
INSERT INTO table11 VALUES (100000005, 'Queen');
INSERT INTO table11 VALUES (100000006, 'Jack');
```

查看表中的全部数据，可以看到第一次插入的三条数据中，有两条数据被覆盖了：

```mysql
+-----------+--------+
| id        | name   |
+-----------+--------+
| 100000004 | King   |
| 100000005 | Queen  |
| 100000006 | Jack   |
| 100000003 | Susan  |
+-----------+--------+
7 rows in set (0.01 sec)
```

为什么会出现数据覆盖的情况呢，我们对这一结果进行分析。首先，在第一次插入数据前`_rowid`为1，插入的三条数据对应的`_rowid`为 1、2、3。如下图所示：

![14](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220717/21311316580646732v9d9k14.svg)

当手动设置`_rowid`为最大值后，下一次插入数据时，插入的`_rowid`重新从0开始，因此第二次插入的三条数据的`_rowid`应该为 0、1、2。这时准备被插入的数据如下所示：

![16](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220717/2130321658064632oNLNMU16.svg)

当出现相同`_rowid`的情况下，新插入的数据会根据`_rowid`覆盖掉原有的数据，过程如图所示：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220717/21354716580649470pGDhF1.svg" alt="1" style="zoom:80%;" />

所以当表中的主键或唯一索引不满足我们前面提到的要求时，`innoDB`使用的隐式的`_rowid`是存在一定风险的，虽然说`2^48`这个值很大，但还是有可能被用尽的，当`_rowid`用尽后，之前的记录就会被覆盖。从这一角度也可以提醒大家，在建表时一定要创建主键，否则就有可能发生数据的覆盖

### <font color=#1FA774>总结</font>

**<font color='red'>主键</font>**、**<font color='red'>唯一索引</font>**、**<font color='red'>非空</font>**、**<font color='red'>单独一列</font>**、**<font color='red'>数值类型</font>**

当没有主键、但存在唯一索引的情况下，只有该唯一索引被添加在数值类型的字段上，且该字段添加了非空约束时，才能够显式地查询到`_rowid`，并且`_rowid`引用了这个唯一索引字段的值

当同时存在符合条件的主键和唯一索引时，无论创建顺序如何，`_rowid`都会优先引用主键字段的值

既没有主键也没有非空唯一索引时，`_rowid`也存在，只不过是隐式的使用`InnoDB`维护的全局变量`dictsys.row_id`

说白了，上面的总结全都是围绕官方文档的三句话展开的，这里再次把三句话放到下面：

>If a table has a **<font color='red'>PRIMARY KEY</font>** or **<font color='red'>UNIQUE NOT NULL</font>** index that consists of a single column that has an integer type, you can use **<font color='red'>_rowid</font>** to refer to the indexed column in **<font color='red'>SELECT</font>** statements
>
>**<font color='red'>_rowid</font>** refers to the **<font color='red'>PRIMARY KEY</font>** column if there is a **<font color='red'>PRIMARY KEY</font>** consisting of a single integer column. If there is a **<font color='red'>PRIMARY KEY</font>** but it does not consist of a single integer column, **<font color='red'>_rowid</font>** cannot be used.
>
>Otherwise, **<font color='red'>_rowid</font>** refers to the column in the first **<font color='red'>UNIQUE NOT NULL</font>** index if that index consists of a single integer column. If the first **<font color='red'>UNIQUE NOT NULL</font>** index does not consist of a single integer column, **<font color='red'>_rowid</font>** cannot be used.

### <font color=#1FA774>缺少主键或非空唯一索引存在问题</font>

- 使用不了主键索引，查询会进行全表扫描
- 影响数据插入性能，插入数据需要生成`_rowid`，而生成的`_rowid`是全局共享的，并发会导致锁竞争，影响性能
