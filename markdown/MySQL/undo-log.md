# undo log

### <font color='#1FA774'>undo log 是什么？</font>

在 **[MySQL 事务](./MySQL事务.html#事务的状态)** 提到一个 MySQL 事务在未提交前其实已经执行，但仅仅只是对内存中缓冲页页修改，并未刷新回磁盘，事务提价后也不会立刻刷回磁盘，**脏页刷盘时机可见 [脏页刷新回磁盘](./InnoDB的Buffer-Pool.html#脏页刷新回磁盘)**

一个事务提交后不立刻刷盘，不怕数据丢失吗？这个由 **[redo log](./redo-log.html)** 保证持久性！！本篇文章讨论另外一个问题：**事务如何回滚**

如果一个事务由于客观原因 (如：断电，故障) 导致没有完全执行完，也就是事务没有执行到提交那一步，那么就应该回滚到执行事务之前的状态，保证事务的 **[原子性](./MySQL事务.html#事务的特性)**

如果一个事务由于主观原因 (如：手动回滚) 导致没有完全执行完，也需要回滚到执行事务之前的状态

回滚的实现思路也很简单：将事务中执行增删改操作的有用信息记录下来，回滚时执行相反操作即可

- 如果事务中插入了一条记录，就保存插入记录的主键，回滚时只需要把该主键对应的记录删掉即可
- 如果事务中删除了一条记录，就保存删除记录的主键及有索引的列，事务提交前这些记录并未真正删除，回滚时只需根据主键和索引列恢复即可
- 如果事务中修改了一个记录，就保存更新列的旧值，回滚时只需要把这些列改回旧值即可

这些保存的内容就被称为 undo log (撤销日志)，根据 undo log 可以恢复到事务执行前的状态

**<font color='red'>注意：</font>**insert undo log 只用于事务未提交前的回滚操作，所以如果事务已经提交，那么 undo log 就没用了，占用的空间会被系统回收，也可能会被重用

**<font color='red'>注意：</font>**update undo log 不仅用于回滚操作，而且在 **[MVCC](./MVCC.html)** 中也会使用，所以就算事务提交了也不能立马删掉 update undo log (包括 delete undo log)

### <font color='#1FA774'>trx_id & roll_pointer</font>

在 **[InnoDB 行格式](./InnoDB记录的存储结构.html#innodb-行格式)** 中介绍一条记录中包含了三个隐藏列，其中 row_id 非必需，后面两个隐藏列就是本部分重点介绍的内容

![11](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220717/1729521658050192kslQcB11.svg)

trx_id 表示事务 ID，占 6 个字节，保存着记录最新被哪个事务修改 (包括插入、删除、更新操作)

roll_pointer 表示回滚指针，指向记录最新的 undo log，而删除更新的 undo log 中也有 roll_pointer 指针，会形成一个链表，表示该记录的版本链

### <font color='#1FA774'>undo log 格式</font>

undo log 被存储到类型为 FIL_PAGE_UNDO_LOG 的页面中，每一个 undo log 都有一个编号 undo no，这个编号是以事务为范围，也就是每个事务所产生的 undo log 编号从 0 开始递增

存储 undo log 的页面在表空间中，而存储 redo log 的 block 不属于表空间，这是因为 redo log 为了保证持久化刷盘更频繁，独立于表空间外可以是顺序 IO，效率更高

由于存储 unod log 的页面上属于表空间，所以也存在数据丢失的风险，所以 undo log 的持久化和数据页一样都依靠 redo log 来保证！！

我们可以将 SQL 语句分为四类：增删改查

- 对于查询操作，不存在回滚，所以没有对应的 undo log
- 对于插入操作，创建了 TRX_UNDO_INSERT_REC 类型的 undo log 结构
- 对于删除操作，创建了 TRX_UNDO_DEL_MARK_REC 类型的 undo log 结构
- 对于更新操作，创建了 TRX_UNDO_UPD_EXIST_REC 类型的 undo log 结构

在正式介绍三种类型的 undo log 结构之前，建一个表：

```mysql
create table undo_demo {
	id int not null,
	key1 varchar(100),
	col varchar(100),
	PRIMARY KEY (id),
	KEY idx_key1 (key1)
}
```

#### <font color=#9933FF>插入操作对应的 undo log 结构</font>

插入操作的回滚相比于另外两种来说简单很多，因为它只需要保存插入记录的主键，回滚时根据主键删除记录即可。TRX_UNDO_INSERT_REC 类型的 undo log 完整结构如下图所示：

![18](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230502/0238421682966322pVN3RT18.svg)

现在向表中插入两条记录：

```mysql
begin;  # 显式开启一个事务，假设该事务的 id 为 100
# 插入两条记录
insert into undo_demo(id, key1, col) values (1, 'AWM', '狙击枪'), (2, 'M416', '步枪');
```

插入的两条记录对应的 undo log 如下：

![19](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230502/0243171682966597NVcZol19.svg)

下面来看看每条记录中 roll_pointer 指针：

![20](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230502/0400261682971226ZDyO8q20.svg)

#### <font color=#9933FF>删除操作对应的 undo log 结构</font>

删除一个记录并非真正的删除了它，而只是将 **[deleted_flag](./InnoDB数据页的结构.html#deletedflag)** 置为 1，并将记录添加到垃圾链表的头部

![21](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230502/03161916829685791gizCR21.svg)

删除操作可以被分为两个阶段：

- **阶段 1：**仅仅将记录头中 deleted_flag 属性设置为 1，也会修改 trx_id，roll_pointer 这些隐藏列的值，其余的不做修改，该阶段称为 delete mark
- **阶段 2：**事务提交后有专门的线程负责将记录真正删掉，如：从正常链表中移除，添加到垃圾链表头部等一些其它设置，该阶段称为 purge

可以看到阶段 2 是在事务提交后才会执行，而事务提交后就不需要回滚，所以对于删除的回滚只关注阶段 1 即可，阶段 1 又被称为中间状态记录，如下图所示：

![22](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230502/0326071682969167htXfwR22.svg)

下面给出 TRX_UNDO_DEL_MARK_REC 类型的 undo log 完整结构，如下图所示：

![23](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230502/03573416829710542h5a8f23.svg)

将一条记录删掉，但事务未提交之前版本链如下：

![24](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230502/0403351682971415A3WwRE24.svg)

#### <font color=#9933FF>更新操作对应的 undo log 结构</font>

更新操作需要分情况讨论

- **不更新主键：**
    - **就地更新：**如果所有更新列的新值占用空间和对应旧值一样 (大了或者小了都不行)，就原地更新
    - **先删除旧记录，再插入新纪录：**此处的删除会添加到垃圾链表，也就是会执行上部分介绍的阶段 2。插入新记录时会检查垃圾链表的表头空间是否可以满足新记录 (只检查表头，不会遍历整个链表)，如果可以就使用垃圾链表表头空间，可能会产生碎片；否则就新申请一块空间，如果当前页满了，就分裂页面
- **更新主键：**将旧记录 delete mark 操作 (不会 purge 操作)，然后插入一条新记录

下面给出 TRX_UNDO_UPD_EXIST_REC 类型的 undo log 完整结构，如下图所示：

![25](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230502/0417171682972237xCjhHi25.svg)
