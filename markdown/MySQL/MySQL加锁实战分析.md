# MySQL 加锁实战分析

### <font color=#1FA774>前言</font>

在文章 **[锁](./锁.html)** 中介绍了三大类锁：全局锁、表锁、行锁

- **全局锁：**锁整个数据库，主要用于全库备份
- **表锁：**存储引擎提供的 S 锁和 X 锁 (一般不用)；server 层提供的 MDL 锁；意向 S 锁，意向 X 锁；用于主键自增的 AUTO-INC 锁
- **行锁：**Record Lock (记录锁)；Gap Lock (间隙锁)；Next-Key Lock (记录锁 + 间隙锁)

InnoDB 中支持行级锁，而 MyISAM 中只支持表级锁，锁粒度更细会提高系统整体的并发量，本篇文章主要分析在 InnoDB 中执行语句时是如何加行锁滴！！

在执行下面四类语句时都会加行级锁：

```mysql
# 下面两条 select 和普通的 select 的区别在于：
#  - 普通 select 是快照读，不会加锁
#  - 下面的两条 select 是当前读，永远都读最新的数据，会加锁
select ... lock in share mode;  # 对读取的记录加共享锁 (S 锁)
select ... for update           # 对读取的记录加独占锁 (X 锁)

# update 和 delete 每次也都是读取最新的数据，然后执行修改或删除操作，且都会加锁
update ...                      # 对更新的记录加独占锁 (X 锁)
delete ...                      # 对删除的记录加独占锁 (X 锁)
```

**<font color='red'>注意：</font>**上面四种类型语句的加锁只有在事务提交后才会被释放

下面先来分析一下「S 锁和 X 锁」与「Record Lock、Gap Lock、Next-Key Lock」的关系！为什么说行锁有 Record Lock、Gap Lock、Next-Key Lock，又说行锁分为 S 锁和 X 锁？？

举个很简单的例子，世界上的人分为两类：男人和女人，而男人和女人中又可以细分为：老人、小孩、年轻人等。行锁的分类也类似：

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/0155051683395705zoCcpk3.svg)

**<font color='red'>注意：</font>**S 型 Gap Lock 和 X 型 Gap Lock 其实没有区别，都是不允许在指定间隙插入新记录，所以 Gap Lock 并没有刻意区分 S 锁和 X 锁

S 锁和 X 锁的兼容性规则如下图所示：(**<font color='red'>注意：</font>**无论是 Gap Lock 还是 Next-Key Lock 的 S 锁和 X 锁都遵循该兼容规则)

![13](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230505/2327411683300461VXaaF813.svg)

**<font color='red'>重点：</font>**一般情况下，对于锁定读的语句，在隔离级别为 READ UNCOMMITTED 和 READ COMMITTED 时，会为当前记录加 Record Lock；在隔离级别为 REPEATABLE READ 和 SERIALIZABLE 时，会为当前记录加 Next-Key Lock。但存在锁退化的特殊情况，后文出现时会分析原因！！

**<font color='red'>注意：</font>**虽然 **[REPEATABLE READ 隔离级别中并不能彻底解决幻读](./RR隔离级别下彻底解决幻读了吗.html)**，但在实现时尽最大能力避免幻读的出现，所以加的是 Next-Key Lock，其中包含的间隙锁含义就是专门用于解决幻读

### <font color=#1FA774>几个概念</font>

在正式介绍之前再介绍几个小小的概念 (不要急～)

**唯一索引：**主键索引、声明为`UNIQUE KEY`的索引都是唯一索引，值不允许重复。需要注意`UNIQUE`索引中可以包含多个 NULL 值，而主键索引规定不允许为 NULL

**非唯一索引：**除了上面提到的两种索引外都是非唯一索引，值允许重复

**等值查询：**查询语句中判断条件是一个等值判断，如：`where id = 1`。如果是唯一索引中的等值查询，那么最多查询出一条记录；如果是非唯一索引中的等值查询，那么可能会查询出多条记录

**范围查询：**查询语句中判断条件是一个范围判断，如：`where id > 1`。无论是否为唯一索引，都可能会查询出多条记录

最后顺便在这一部分给出后文都会使用到的表结构：

```mysql
CREATE TABLE `user` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(30) NOT NULL,
  `age` int NOT NULL,
  PRIMARY KEY (`id`),
  KEY `index_age` (`age`)
) ENGINE=InnoDB;
```

表中记录如下图所示：

![4](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/2056121683464172IwPn9M4.svg)

**<font color='red'>注意：</font>**后文所有实验内容都基于 **MySQL 8.0.33**，且在 **Repeatable Read** 隔离级别下！！

### <font color=#1FA774>唯一索引等值查询</font>

#### <font color=#9933FF>记录存在</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where id = 15 for update;
commit;
```

那么事务 A 会为`id = 15`的记录加 **<font color='red'>X 型记录锁</font>**，如下图所示：

![5](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/2056391683464199rIwjft5.svg)

由于 X 型记录锁和其它任何锁都不兼容，如果在事务 A 没有提交期间其它事务执行需要获取`id = 15`记录锁的语句，都将会被阻塞，下面列出三种情况：

<img src="https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/0321501683400910E7xY3Y6.svg" alt="6" style="zoom:90%;" />

上面的分析都是口说无凭，下面直接来点证据！！在 MySQL 8 中新增表`performance_schema.data_locks`，它记录了正在执行事务中的锁信息，从该表中可以得知运行事务的加锁情况

```mysql
# 在另一个会话中执行下面语句即可得到运行事务的加锁情况
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 5259466976:1066:5769321912
ENGINE_TRANSACTION_ID: 3436
            THREAD_ID: 56
             EVENT_ID: 47
        OBJECT_SCHEMA: study
          OBJECT_NAME: user
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 5769321912
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 5259466976:5:4:4:5771827736
ENGINE_TRANSACTION_ID: 3436
            THREAD_ID: 56
             EVENT_ID: 47
        OBJECT_SCHEMA: study
          OBJECT_NAME: user
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 5771827736
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,REC_NOT_GAP          # X 型记录锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 15
2 rows in set (0.00 sec)
```

从输出可以看出：

- **表锁：**X 型意向锁，**由于本篇文章侧重于行锁，所以关于表级意向锁详细介绍可见 [意向共享锁 & 意向独占锁](./锁.html#意向共享锁--意向独占锁)**
- **行锁：**X 型记录锁

通过 LOCK_MODE 可以判断表级加锁的类型：(这是第一次出现就详细介绍了一下，下同！！)

- 如果 LOCK_MODE 为`X`，说明是 Next-Key Lock
- 如果 LOCK_MODE 为`X,REC_NOT_GAP`，说明是 X 型 Record Lock
- 如果 LOCK_MODE 为`X,GAP`，说明是 X 型 GAP Lock

**<font color='red'>问题：</font>**在 **[前言](./MySQL加锁实战分析.html#前言)** 部分的最后还特意强调 RR 隔离级别下会为记录加 Next-Key Lock，但在唯一索引等值查询且记录存在的情况下，给记录加的锁退化成 Record Lock，为什么？？

Next-Key Lock 和 Record Lock 唯一的区别就是多了间隙锁，而间隙锁是专门为了避免出现幻读现象。在唯一索引等值查询且记录存在的条件下，那么查询结果集数量为 1

不允许其它事务对查询出来的记录删除，也就避免了再次查询后结果集为 0 的情况。因为唯一索引的约束使得不能插入 id 相同的记录，那么就算在查询记录前后插入新记录也不会使结果集数量增加

**综上所述：**在一个事务中前后查询的结果集数量即不会增加也不会减少，避免了幻读的出现，所以只加记录锁就够了，不需要加间隙锁！！

#### <font color=#9933FF>记录不存在</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where id = 1 for update;
commit;
```

那么事务 A 会在`id > 1`且差值最小的记录出加**<font color='red'>间隙锁</font>**，如下图所示：

![7](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/205650168346421076SlK07.svg)

由于间隙锁不允许插入新记录，如果在事务 A 没有提交期间其它事务执行插入`id < 5`记录的操作，都将会被阻塞，具体见下图：

![8](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/0429581683404998KR3UNJ8.svg)

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,GAP                  # X 型间隙锁
            # ......
```

**<font color='red'>问题：</font>**在唯一索引等值查询且记录不存在的情况下，给记录加的锁从 Next-Key Lock 退化成 Gap Lock，为什么？？

在唯一索引等值查询且记录不存在的条件下，那么查询结果集数量为 0，结果集不可能变为负数，所以完全不用担心其它事务删除操作。加间隙锁可以避免插入`id < 5`的记录，那么查询结果集不可能增加

**综上所述：**在一个事务中前后查询的结果集数量即不会增加也不会减少，避免了幻读的出现，所以只加间隙锁就够了，不需要加记录锁！！

#### <font color=#9933FF>总结</font>

对于记录存在的情况，可以找到符合条件的记录，按理来说应该为符合条件的记录加 Next-Key Lock，但由于唯一索引等值查询的约束，直接可以退化成记录锁

对于记录不存在的情况，可以找到大于要求条件的第一条记录，按理来说应该为这条记录加 Next-Key Lock，但由于唯一索引等值查询的约束，直接可以退化成间隙锁

### <font color=#1FA774>唯一索引范围查询</font>

#### <font color=#9933FF>对于 > 的范围查询</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where id > 15 for update;
commit;
```

那么事务 A 会加三个 **<font color='red'>X 型 Next-Key Lock</font>**，如下图所示：

![9](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/2056591683464219uq4ZfC9.svg)

在事务 A 没有提交期间，其它事务不允许对`id = 20, 25`的记录更新、删除、锁定读，也不允许插入`15 < id < 20 || 20 < id < 25 || 25 < id `的记录

**<font color='red'>🤔️ 疑惑：</font>**给 Supremum 记录加 Next-Key Lock，相当于加了记录锁和间隙锁，可是 Supremum 记录本身就无法被访问，那为什么不直接退化成间隙锁呢？？？！！！

**锁退化的前提是：退化前的锁和退化后的锁可以实现同样的效果**。对于普通记录而言，锁退化可以使锁住的范围更小，从而提高系统整体的并发度；对于 Supremum 记录而言，Next-Key Lock 只会锁间隙，因为 Supremum 记录也无法访问，所以加 Next-Key Lock 或者只加间隙锁效果都一样，锁住的范围也一样，但为了追求简单，加锁时不用判断，所以并没有锁退化

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record # 锁定区间 (25, +∞]
*************************** 3. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 20                     # 锁定区间 (15, 20]
*************************** 4. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 25                     # 锁定区间 (20, 25]
```

#### <font color=#9933FF>对于 >= 的范围查询 (临界值记录存在)</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where id >= 15 for update;
commit;
```

那么事务 A 会加一个 **<font color='red'>X 型记录锁</font>** 和三个 **<font color='red'>X 型 Next-Key Lock</font>**，如下图所示：

![10](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/2057101683464230FTqy1B10.svg)

在事务 A 没有提交期间，其它事务不允许对`id = 15, 20, 25`的记录更新、删除、锁定读，也不允许插入`15 < id < 20 || 20 < id < 25 || 25 < id `的记录

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,REC_NOT_GAP          # X 型记录锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 15                     # 锁定区间 [15, 15]
*************************** 3. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record # 锁定区间 (25, +∞]
*************************** 4. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 20                     # 锁定区间 (15, 20]
*************************** 5. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 25                     # 锁定区间 (20, 25]
```

**<font color='red'>问题：</font>**按理来说应该会加四个 X 型 Next-Key Lock，但其中有一个锁退化成 X 型记录锁，为什么？？

只要将事务 A 执行的查询语句拆分成下面两部分就可以一眼看出原因：

```mysql
select * from user where id = 15 for update;
select * from user where id > 15 for update;
```

由于`id = 15`的记录存在，那么第一条语句就和 **[唯一索引等值查询 - 记录存在](./MySQL加锁实战分析.html#记录存在)** 的情况一样了，锁退化原因也一样！！

#### <font color=#9933FF>对于 >= 的范围查询 (临界值记录不存在)</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where id >= 16 for update;
commit;
```

该情况和 **[对于 > 的范围查询](./MySQL加锁实战分析.html#对于--的范围查询)** 相同，加锁情况也相同，这里就不展开介绍

#### <font color=#9933FF>对于 < 的范围查询</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where id < 11 for update;
commit;
```

那么事务 A 会加一个 **<font color='red'>X 型间隙锁</font>** 和两个 **<font color='red'>X 型 Next-Key Lock</font>**，如下图所示：

![11](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/221005168346860565PuSA11.svg)

在事务 A 没有提交期间，其它事务不允许对`id = 5, 10`的记录更新、删除、锁定读，也不允许插入`id < 5 || 5 < id < 10 || 10 < id < 15 `的记录

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 5                      # 锁定区间 (-∞, 5]
*************************** 3. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10                     # 锁定区间 (5, 10]
*************************** 4. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,GAP                  # X 型间隙锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 15                     # 锁定区间 (10, 15)
```

#### <font color=#9933FF>对于 <= 的范围查询 (临界值记录存在)</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where id <= 10 for update;
commit;
```

那么事务 A 会加两个 **<font color='red'>X 型 Next-Key Lock</font>**，如下图所示：

![12](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/2240371683470437pvhX4o12.svg)

在事务 A 没有提交期间，其它事务不允许对`id = 5, 10`的记录更新、删除、锁定读，也不允许插入`id < 5 || 5 < id < 10 `的记录

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 5                      # 锁定区间 (-∞, 5]
*************************** 3. row ***************************
            # ......
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10                     # 锁定区间 (5, 10]
```

#### <font color=#9933FF>对于 <= 的范围查询 (临界值记录不存在)</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where id <= 11 for update;
commit;
```

该情况和 **[对于 < 的范围查询](./MySQL加锁实战分析.html#对于--的范围查询-2)** 相同，加锁情况也相同，这里就不展开介绍

#### <font color=#9933FF>总结</font>

对于 < 的范围查询，必须遍历到第一条不符合条件的记录才停止，按理来说应该为这条记录加 Next-Key Lock，但由于唯一索引的约束，直接可以退化成间隙锁，为符合条件的记录加 Next-key Lock 即可

对于 <= 的范围查询 (临界值记录存在)，由于唯一索引的约束，遍历到临界值记录就可以停止，此时遍历的记录均为符合要求的记录，为这些记录加 Next-key Lock 即可

对于 <= 的范围查询 (临界值记录不存在)，直接等价于对于 < 的范围查询

### <font color=#1FA774>非唯一索引等值查询</font>

非唯一索引属于二级索引，如果通过执行计划得知语句使用非唯一索引查询，那么就是在对应的二级索引树中，二级索引树中的记录是按照二级索引大小排序，如下图所示：

![13](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/2305101683471910cgUKR513.svg)

#### <font color=#9933FF>记录存在</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where age = 18 for update;
commit;
```

那么事务 A 会为`age = 18`的记录对应的主键索引加一个 **<font color='red'>X 型记录锁</font>**，以及对应的二级索引加一个 **<font color='red'>X 型 Next-Key Lock</font>** 和一个 **<font color='red'>X 型间隙锁</font>**，如下图所示：

![14](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230507/2326051683473165CnJn0x14.svg)

在事务 A 没有提交期间，其它事务不允许对`age = 18`的记录对应的二级索引更新、删除、锁定读，不允许对`id = 20`的记录对应的主键索引更新、删除、锁定读

关于其它事务是否可以插入新记录的情况，略微的麻烦，慢慢分析～。插入一条记录会在所有的索引树中也添加该记录，目前只有二级索引中有两个间隙锁，会影响记录的插入：

- 首先肯定不能插入`10 < age < 20`的记录，该范围内的记录刚好落在二级索引的两个间隙中，如下图 (a) 所示
- 对于插入`age = 10`的记录，也必须满足`id < 10`，否则会插入到第一个间隙中，导致插入失败，如下图 (b) 所示
- 同理，对于插入`age = 20`的记录，也必须满足`id > 15`，否则会插入到第二个间隙中，导致插入失败，如下图 (c) 所示

**<font color='red'>重点：</font>**判断能否插入的标准就是新记录是否会插入到间隙锁定的区间内，如果是，就不能插入；否则就可以插入

![15](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230508/0016541683476214XYl52F15.svg)

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
           INDEX_NAME: index_age              # 二级索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 18, 20                 # 锁定区间 (10, 18]
*************************** 3. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,REC_NOT_GAP          # X 型记录锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 20                     # 锁定区间 [20, 20]
*************************** 4. row ***************************
            # ......
           INDEX_NAME: index_age              # 二级索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,GAP                  # X 型间隙锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 20, 15                 # 锁定区间 (18, 20)
```

#### <font color=#9933FF>记录不存在</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where age = 19 for update;
commit;
```

那么事务 A 为二级索引加一个 **<font color='red'>X 型间隙锁</font>**，如下图所示：

![16](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230508/00281416834768943nmf9916.svg)

在事务 A 没有提交期间，其它事务不允许插入`18 < age < 20`的记录，对于插入`age = 18, 20`的记录还需要根据`id`的值来判断，**判断标准同 [非唯一索引等值查询 - 记录存在](./MySQL加锁实战分析.html#记录存在-2)**

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
           INDEX_NAME: index_age              # 二级索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,GAP                  # X 型间隙锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 20, 15                 # 锁定区间 (18, 20)
```

### <font color=#1FA774>非唯一索引范围查询</font>

**<font color='red'>注意：</font>**在非唯一索引范围查询时，二级索引上查询的记录全都加 Next-key Lock，不会出现锁退化的情况

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where age >= 20 for update;
commit;
```

那么事务 A 会为记录对应的主键索引加两个 **<font color='red'>X 型记录锁</font>**，以及对应的二级索引加三个 **<font color='red'>X 型 Next-Key Lock</font>**，如下图所示：

![17](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230508/01074716834792673tMf7217.svg)

在事务 A 没有提交期间，其它事务不允许对`age = 20 30`的记录对应的二级索引更新、删除、锁定读，不允许对`id = 15 25`的记录对应的主键索引更新、删除、锁定读

其它事务不允许插入`18 < age`的记录，对于插入`age = 18`的记录，也必须满足`id < 20`，否则会插入到第一个间隙中，导致插入失败

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
           INDEX_NAME: index_age              # 二级索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record # 锁定区间 (30, +∞)
*************************** 3. row ***************************
            # ......
           INDEX_NAME: index_age              # 二级索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 20, 15                 # 锁定区间 (18, 20]
*************************** 4. row ***************************
            # ......
           INDEX_NAME: index_age              # 二级索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 30, 25                 # 锁定区间 (20, 30]
*************************** 5. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,REC_NOT_GAP          # X 型记录锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 15                     # 锁定区间 [15, 15]
*************************** 6. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X,REC_NOT_GAP          # X 型记录锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 25                     # 锁定区间 [25, 25]
```

### <font color=#1FA774>查询不使用索引</font>

假设事务 A 执行的语句如下：

```mysql
# 事务 A
begin;
select * from user where name = 'ggg' for update;
commit;
```

通过执行计划可知该语句没有使用任何索引：

```mysql
mysql> explain select * from user where name = 'ggg';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    8 |    12.50 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

那么事务 A 会加六个 **<font color='red'>X 型记录锁</font>**，相当于为每个记录都加了一个 **<font color='red'>X 型记录锁</font>**，即：锁全表

上面的分析都是口说无凭，下面直接来点证据！！为了节约篇幅，只摘出了关键信息

```mysql
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
            # ......
            LOCK_TYPE: TABLE                  # 表级锁
            LOCK_MODE: IX                     # X 型意向锁
            # ......
*************************** 2. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record # 锁定区间 (30, +∞)
*************************** 3. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 5                      # 锁定区间 (-∞, 5]
*************************** 4. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10                     # 锁定区间 (5, 10]
*************************** 5. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 15                     # 锁定区间 (10, 15]
*************************** 6. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 20                     # 锁定区间 (15, 20]
*************************** 7. row ***************************
            # ......
           INDEX_NAME: PRIMARY                # 主键索引
            LOCK_TYPE: RECORD                 # 行级锁
            LOCK_MODE: X                      # Next-Key Lock
          LOCK_STATUS: GRANTED
            LOCK_DATA: 25                     # 锁定区间 (20, 25]
```

### <font color=#1FA774>总结</font>

**唯一索引加锁流程：**

![18](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230508/0255401683485740fMKFTz18.svg)

**非唯一索引加锁流程：**

![19](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230508/0255471683485747SGmHcq19.svg)
