# binlog

### <font color='#1FA774'>binlog 是什么？</font>

前文介绍了 **[undo log](./undo-log.html)**，它主要用于事务回滚，记录每一个增删改操作的信息，使可以恢复到执行操作之前的状态

前文介绍了 **[redo log](./redo-log.html)**，它主要用于保证持久性，记录每一个增删改操作在某表空间某页做了某修改，使系统崩溃后可以恢复未刷新到磁盘中的数据

本文要介绍的 binlog 和 redo log 有一点相似的地方，但不多！在每一个增删改操作后，server 层会生成一条 binlog，一个事务中可能会生成若干条 binlog，事务提交后统一写入 binglog 文件中

binlog 文件是记录了所有数据库表结构变更 (create、alter、drop table) 和表数据修改 (insert、update、delete) 的日志，不会记录查询类操作，如：`select`、`show`等

但并不是没有导致数据库发生变化就不需要记录到 binlog，假设表中没有`a = 2`的记录，但执行`update t set a = 1 where a = 2`操作，虽然没有导致数据库变化，但依然会记录到 binlog 中

### <font color='#1FA774'>binlog 的格式</font>

一共有三种二进制记录的方式：

- **Statement 模式：**记录每一条修改数据的 SQL，恢复时重新执行 SQL 即可，类似于 **[Redis 中的 AOF 持久化机制](../redis/Redis持久化机制.html#aof)**。缺点在于含有动态函数的 SQL 重新执行会发生变化，如：`now()`
- **Row 模式：**记录表中行更改情况，也就是每一行被修改成的内容。缺点在于批量修改会记录大量的数据，如一次性修改了 n 行数据，还那么就需要记录 n 行数据修改后的内容
- **Mixed 模式：**Statement 模式和 Row 模式的混合，默认使用 Statement 模式，少数特殊具体场景自动切换到 Row 模式

MySQL 5.1.5 之前 binlog 的格式只有 Statement 模式，MySQL 5.1.5 开始支持 Row 格式的 binlog，从 MySQL 5.1.8 开始支持 MIXED 模式

MySQL 5.7.7 之前默认使用 Statemen 模式，MySQL 5.7.7 开始默认使用 Row 模式

### <font color='#1FA774'>binlog 的作用</font>

redo log 文件组是循环写入，只记录内存中未刷盘的脏页，当脏页刷盘后，redo log 对应的数据就可以覆盖，称作一次 **[checkpoint](./redo-log.html#redo-log-文件组)**

而 binlog 文件记录了修改的所有数据，不会覆盖，所以 binlog 文件只会越来越大，由多个以`文件名.00000*`命名的文件组成，可以通过`max_binlog_size`参数设置每个文件的最大容量

即然 binlog 记录了全量数据，那么就可以用来**<font color='red'>备份恢复</font>**，**<font color='red'>主从复制</font>**

**<font color='red'>问题一：</font>**redo log 可以用来备份恢复或者主从复制吗？

- 由于 redo log 是循环写入，当脏页被刷盘后，redo log 对应的数据就可以覆盖，所以无法获得全量数据，进而无法用来备份恢复或主从复制

**<font color='red'>问题二：</font>**binlog 可以替代 redo log 用作断电后恢复吗？

- redo log 主要用作断电后恢复，保存着 Buffer Pool 中未刷盘的脏页中修改的数据，所以当断电后，可以从 redo log 文件组中得知哪些数据还未来得及刷新到磁盘中

- 但 binlog 中记录了全量数据，无论是未刷盘的脏页中修改的数据，还是已经在磁盘中的数据，而且从 binlog 文件无法判断哪些数据已经刷盘，哪些数据未刷盘，所以就无法只恢复未刷盘数据

**关于该问题更详细的解释可见 [为什么 redo log 具有 crash-safe 的能力，是 binlog 无法替代的？](https://cloud.tencent.com/developer/article/1757612)**

**<font color='red'>问题三：</font>**binlog 和 redo log 的区别？

- **适用对象不同**

    - binlog 是 MySQL 的 server 层实现的日志，所有存储引擎都可以使用

    - redo log 是 InnoDB 存储引擎实现的日志

- **文件格式不同**

    - binlog 是逻辑日志，记录了恢复数据的逻辑操作，有三种格式

    - redo log 是物理日志，记录了某表空间某页做了某修改

- **写入方式不同**

    - binlog 是追加写，当一个 binlog 文件写满后，新建一个文件继续写，不会覆盖以前的日志，保证着全量数据

    - redo log 是循环写，日志空间大小固定，从头部写到尾部再回到头部

- **用途不同**

    - binlog 用于备份恢复，主从复制

    - redo log 用于断电等故障后恢复

### <font color='#1FA774'>MySQL 主从复制</font>

MySQL 的主从复制依赖于 binlog 日志，因为它记录了数据库中所有的变化，并以二进制的形式存储在磁盘上，复制的过程就是将 binlog 中数据从主库传输到从库

总的来说，复制有三个步骤：

- **写入 binlog：**在主库上把数据更改记录到二进制日志 (binlog) 中
- **同步 binlog**：从库将主库上的日志复制到自己的中继日志 (relay log) 中
- **重放 binlog：**从库读取中继日志中的事件，将其重放到从库数据之上

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230503/0217281683051448GS5d7B1.svg)

在第一步中，多个事务可能是交替执行，但每个事务生成的 binlog 日志是按照顺序写入 binlog 文件中

在第二步中，主库会创建一个 binlog dump 线程负责发送 binlog 给从库，从库会创建一个 I/O 线程连接 binlog dump 线程，负责接收 binlog 并写入 relay log 中。I/O 线程并不会一直对事件轮询，当该线程追赶上主库后就进入睡眠状态，直到主库发送信号量通知它有新的事件产生时才会被唤醒 (注：主从之间通信基于事件驱动)

在第三步中，从库会创建一个 SQL 线程专门负责读 relay log 并重放数据，最终实现主从数据一致性

**<font color='red'>注意：</font>**主库会为每一个从库的 I/O 线程启动一个 binlog dump 线程，所以并不是从库越多越好，越多的从库代表着主库要创建越多的 binlog dump 线程，一般 1 个主库配 2 ～ 3 个从库

主从复制一般主要有三种模型：

- **同步复制：**主库等待所有从库复制完成才返回客户端，即各从库的 SQL 线程重放完新增 binlog。一般不会使用这种方式，主要是性能差，可用性也差，一旦主从任何一个数据库出问题，就会影响业务
- **异步复制 (默认)：**主库不用等待 binlog 同步到所有从库就直接返回客户端，即 binlog dump 线程发送 binlog 给各从库。这种方式一旦主库宕机，数据可能就丢失，因为不能保证从库完成复制
- **半同步复制：**主库只等待一个从库复制完成就返回客户端，这样主库宕机，至少可以保证还有一个从库有完整数据

**<font color='red'>问题：</font>**主从同步延迟分析

首先主从复制都是单线程，即主库的 binlog dump 线程、从库的 I/O 线程和 SQL 线程都只有一个

对于上面的前两步都是顺序 IO，速度较快，而在第三步中 SQL 线程重放 binlog 却是随机 IO，涉及到写数据库，由于记录可能分布在不同的物理页面上，所以写数据库产生的随机 IO 效率很低

当主库并发过高，会导致 relay log 中需要重写的 binlog 就更多，然而 SQL 线程重放的效率又很慢，就会导致主从同步出现延迟

如果从库数据安全性要求不高的情况下，可以将设置参数`innodb_flush_log_at_trx_commit = 0`，**详情可见 [redo log 刷盘时机](./redo-log.html#redo-log-刷盘时机)**

**关于该问题更详细的解释可见 [深入解析 Mysql 主从同步延迟原理及解决方案](https://www.cnblogs.com/fengff/p/11011702.html)**

### <font color='#1FA774'>binlog 刷盘时机</font>

每个事务生成的 binlog 日志需要按照顺序写入 binlog 文件中。假设事务 1 和事务 2 分别生成了三条 binlog 日志，编号为`1 2 3`和`4 5 6`

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230503/0309181683054558mGmQ7r2.svg)

由于从库中重放过程是单线程，而且每当执行一个新事务时，会默认提交上一个事务，所以如果按照第三种错误顺序执行，会被拆分成四个事务执行，违背了事务的 **[原子性](./MySQL事务.html#事务的特性)**，执行效果如下图所示：

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230503/0314511683054891VzGC293.svg)

MySQL 为每个线程分配了一块内存作为 binlog cache，可以通过参数`binlog_cache_size`控制每个线程 binlog cache 的大小，如果存储内容超过了参数规定的大小，就要暂停存到磁盘

在事务执行时产生的 binlog 先写入到 binlog cache 中；在事务提交时，执行器把 binlog cache 里由事务产生的所有 binlog 写入到 binlog 文件 (page cache) 中，并清空 binlog cache

![4](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230503/0508501683061730s5MxMv4.svg)

可以看到，每个线程都有自己的 binlog cache，但最终都写到了同一个 binlog 文件中

- 图中的`write`，指的是把日志写入到文件系统 page cache (内核) 中的 binlog 文件，并没有把数据持久化到磁盘，`write`速度较快，因为不涉及到磁盘 IO
- 图中的`fsync`，才是将数据持久化到磁盘的操作，频繁的调用`fsync`会增加磁盘 IO 的开销

`write`和`fsync`调用的时机由参数`sync_binlog`控制：

- `sync_binlog = 0`表示每次提交事务都只调用`write`，而不调用`fsync`，后续交由操作系统决定何时刷新到磁盘
- `sync_binlog = 1`表示每次提交事务都会先调用`write`，然后马上调用`fsync`
- `sync_binlog = N (N > 1)`表示每次提交事务都会先调用`write`，但累积 N 个事务后才调用`fsync`

`sync_binlog = 0`性能最好，但安全性最低；`sync_binlog = 1`安全性最高，但性能最低。所以在实际业务场景中，不建议设置为 0，而是设置为 100 ～ 1000 中的某个值

### <font color='#1FA774'>小结：执行一条更新操作时三种日志的处理流程</font>

到此为止，已经介绍完了最主要的三种不同类型的日志：**[undo log](./undo-log.html)**、**[redo log](./redo-log.html)** 以及 binlog，现在我们来看看执行一条`update`时这三种日志到底是如何处理滴！！

假设执行`update t set name = 'LFool' where id = 1`，根据执行计划可以判断使用的是 **[聚簇索引](./B+树索引.html#聚簇索引)**，那么具体流程如下：

- 执行器调用存储引擎的接口，从聚簇索引中找到`id = 1`的记录，如果对应的数据页在 **[Buffer Pool](./InnoDB的Buffer-Pool.html)** 中，直接返回给执行器，否则先将数据页加载到 Buffer Pool 中，再返回给执行器
- 执行器先检查更新前和更新后的记录是否一样，如果一样就不做处理，否则将更新前和更新后的记录作为参数传递给 InnoDB，让它执行真正的更新操作
- **开启事务 ...**
- 更新记录前要记录 **[undo log](./undo-log.html#更新操作对应的-undo-log-结构)** 防止需要回滚，并将 undo log 写入 Buffer Pool 中的 Undo 页，写入后变为脏页，同时也要在 redo log 中记录 Undo 页的修改，防止数据丢失
- 开始更新记录，修改 Buffer Pool 中对应的数据页，修改后变为脏页，同时也要在 redo log 中记录该数据页的修改，防止数据丢失
- 更新记录后要记录该语句对应的 binlog，此时保存到 binlog cache 中，事务提交后根据设置来决定刷盘策略
- **提交事务 ...**

主要分三个阶段：更新前记录 undo log，同时需要在 redo log 中记录 Undo 页的修改；更新时记录对应数据页的 redo log；更新后记录该操作的 binlog

Undo 页和数据页的刷盘时机由 **[Buffer Pool 中脏页刷盘时机](./InnoDB的Buffer-Pool.html#脏页刷新回磁盘)** 决定；redo log 和 binlog 刷盘时机由自身设置策略决定
