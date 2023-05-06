# MVCC

### <font color='#1FA774'>事务 id</font>

在 **[InnoDB 行格式](./InnoDB记录的存储结构.html#innodb-行格式)** 中介绍每条记录中都有三个隐藏列，其中一个是 trx_id，记录着修改该记录的最新事务 id

在 **[undo log 格式](./undo-log.html#undo-log-格式)** 中介绍每条 update undo log 和 delete undo log 中也有一个 trx_id 属性，记录着修改 undo log 对应原记录的事务 id

MySQL 服务器会在内存中维护一个**全局变量**，每当需要为某个事务分配事务 id 时，就会把该变量作为事务 id 分配给该事务，并且把该变量自增 1

**<font color='red'>注意：</font>**只有在事务中第一次执行增删改操作时才会触发事务 id 的分配，所以如果只有读操作不会分配事务 id，而是生成一个很大的值，可以看作是伪事务 id，**详情可见 [mysql中事务id，有啥用？](https://blog.csdn.net/weixin_45701550/article/details/120836349)**

下面总结一些常用的命令：

```mysql
# 查看 innodb 的状态，其中包含了事务的全局变量，如：Trx id counter 1036332
show engine innodb status\G;

# 查看当前事务的事务 id
# 如果事务中未执行任何操作，暂时不会分配事务 id
# 如果事务中只执行了读操作，会分配一个很大的伪事务 id，但这并不是从事务的全局变量中分配
# 如果事务中执行了增删改操作，会分配全局变量作为该事务的事务 id，然后全局变量 +1
select TRX_ID from INFORMATION_SCHEMA.INNODB_TRX where TRX_MYSQL_THREAD_ID = CONNECTION_ID();
```

### <font color='#1FA774'>版本链</font>

在 **[undo log](./undo-log.html)** 中浅浅的提了一下版本链的概念，本部分详细介绍！

记录的隐藏列中有一个 roll_pointer 属性，表示回滚指针，指向记录最新的 undo log，而删除更新的 undo log 中也有 roll_pointer 指针，会形成一个链表，表示该记录的版本链

![7](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230504/1651261683190286tOq3Qz7.svg)

**<font color='red'>注意：</font>**为了统一格式，图中的 undo log 中有些属性是没有的，**关于 undo log 格式的详细介绍可见 [undo log 格式](./undo-log.html#undo-log-格式)**

通过版本链，可以找到该记录任何版本的数据，从而可以在并发事务中控制哪个版本对哪个事务可见，这种机制称为**多版本并发控制 (Multi Version Concurrency Control，MVCC)**

**<font color='red'>注意：</font>**只有普通 select 语句才可以读到历史版本，被称为**快照读**；对于`select ... for update`只能读到最新数据，也就是 B+ 树页面的记录，被称为**当前读**

**<font color='red'>延伸：</font>**对于 update 和 delete 操作，也只能修改最新数据，也就是 B+ 树页面的记录，如果该记录正在被其它事务修改且没有提交，该记录会被加锁，当前事务的操作会被阻塞

### <font color='#1FA774'>ReadView</font>

通过 MVCC 可以控制事务可见的版本，而 MVCC 是通过 ReadView 实现，每个事务会生成一个 ReadView，不同生成时机可以实现不同的可见性效果

MySQL 中有四种隔离级别：READ UNCOMMITTED (读未提交)、READ COMMITTED (读已提交)、REPEATABLE READ (可重复读)、SERIALIZABLE (串行化)，**详情可见 [事务隔离级别](./MySQL事务.html#事务隔离级别)**

- 对于读未提交的隔离级别，每次直接读取记录最新版本即可
- 对于串行化的隔离级别，需要使用加锁的方式来访问
- 对于读已提交和可重复读的隔离级别，必须保证读取的记录版本已经提交，这两种都是通过 ReadView 实现

每个 ReadView 中都包含四个字段：

- **m_idx：**在生成 ReadView 时，当前系统中活跃的读写事务的事务 id 列表，活跃的事务指未提交的事务
- **min_trx_id：**在生成 ReadView 时，当前系统中活跃的读写事务中最小的事务 id，也就是 m_idx 中最小的事务 id
- **max_trx_id：**在生成 ReadView 时，系统应该分配给下一个事务的事务 id 值，也就是事务全局变量 + 1
- **creator_trx_id：**生成 ReadView 事务的事务 id，也就是事务全局变量的值

每个 ReadView 都将记录中的 trx_id 划分为三种情况

![8](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230504/1930071683199807iG0lqf8.svg)

一个事务在访问记录某个版本时，通过以下步骤来判断该版本是否对该事务可见：

- 如果记录版本中 trx_id = creator_trx_id，表示该版本为本事务修改，可以访问
- 如果记录版本中 trx_id < min_trx_id，表示该版本在本事务创建 ReadView 时已经被提交，可以访问
- 如果记录版本中 trx_id >= max_trx_id，表示该版本在本事务创建 ReadView 时还未启动事务，不可以访问
- 如果版本记录中 min_trx_id <= trx_id < max_trx_id，分两种情况：
    - trx_id 在 m_idx 中，表示该版本在本事务创建 ReadView 时正在执行事务，不可以访问
    - trx_id 不在 m_idx 中，表示该版本在本事务创建 ReadView 时已经被提交，可以访问

![9](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230504/1949491683200989fT3SwO9.svg)

### <font color='#1FA774'>READ COMMITTED 实现细节</font>

**<font color='red'>在事务中每次读取数据前都生成一个 ReadView，所以一个事务中可能会生成多个 ReadView</font>**

![10](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230504/20351616832037163NnOEe10.svg)

对于当前事务，操作 1 和操作 2 都无法看见事务 A 的修改，因为此时事务 A 还没有提交；操作 3 和操作 4 可以看见事务 A 的修改，因为此时事务 A 已经提交

在生成的四个 ReadView 中，ReadView1 和 ReadView2 中的 m_idx 列表中有事务 A 的 id，ReadView3 和 ReadView4 中的 m_idx 列表中没有事务 A 的 id，因此此时事务 A 已提交

在 READ COMMITTED 的隔离级别中，只需要保证读取的数据是已提交的即可，也就是避免出现脏读，但依旧会出现不可重复读的现象，因为当前事务中可以读到在执行事务期间其它事务提交后修改的数据

### <font color='#1FA774'>REPEATABLE READ 实现细节</font>

**<font color='red'>在事务中第一次读取数据前生成一个 ReadView，所以一个事务中只会生成一个 ReadView</font>**

![11](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230504/2045541683204354KKpfDy11.svg)

对于当前事务，四个操作都无法看见事务 A 的修改，整个事务期间只有一个 ReadView，而 ReadView 生成时事务 A 并没有提交，所以事务 A 的 id 在 ReadView 中的 m_idx 列表中

在 REPEATABLE READ 的隔离级别中，由于当前事务中不可以看见在执行事务期间其它事务提交后修改的数据，所以避免出现不可重复读

上面是更新的操作，下面再来一种删除的操作，先看图：

![24](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230506/1324211683350661obn80B24.svg)

在时刻 1 和时刻 4，事务 A 两次查询的结果是一致的，因为这是在 REPEATABLE READ 的隔离级别中，利用 ReadView 保证可重复读

**<font color='red'>疑问：</font>**`id = 2`的记录在事务 B 提交后在 **[purge 阶段](./undo-log.html#删除操作对应的-undo-log-结构)** 被加入到垃圾链表中。在时刻 4 事务 A 执行`select`时会全表扫描，可是被删记录已经不在聚簇索引中，那怎么找到删除记录的版本链呢？

其实事务 B 提交后不会立刻执行 purge 阶段，只有当系统中最早产生的 ReadView 都不再访问 unod log 后，才会开始执行 purge 阶段