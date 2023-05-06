# RR 隔离级别下彻底解决幻读了吗？

**[幻读](./MySQL事务.html#幻读-phantom)** 指在同一个事务中，同一个查询语句，前后两次执行得到的结果集不同，如：第一次查询得到六条记录，第二次查询得到七条数据

**<font color='red'>结论：</font>**RR 隔离级别下只能最大程度的避免幻读现象的出现，但无法彻底解决幻读现象！！

MySQL 中有两种读：快照读和锁定读

- **快照读：**通过 **[MVCC](./MVCC.html)** 可以最大程度避免幻读，每次都只能看到生成 ReadView 之前已经提交事务修改的数据，所以在事务执行过程中多次读的结果是一致的
- **锁定读：**通过加锁可以最大程度避免欢度，在 RR 下会为每个查询时符合条件的记录加 **[Next-Key Lock](./锁.html#next-key-lock)**，既不允许修改记录，也不允许在记录前的间隙插入新记录，所以在事务中多次读的结果是一致的

### <font color=#1FA774>快照读 -- 出现幻读的情况</font>

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230506/1417031683353823Spll981.svg)

`update`语句修改的是最新数据。在时刻 4，由于事务 A 修改了`id = 3`的记录，记录中`trx_id`由 200 变成 100，所以该记录对事务 A 可见

### <font color=#1FA774>锁定读 -- 出现幻读的情况</font>

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230506/1433461683354826qxOft12.svg)

`select * from t`是快照读，不会对记录加锁，通过 MVCC 控制；`select * from t for update`是当前读，读最新的数据，而且会对记录加 **[Next-Key Lock](./锁.html#next-key-lock)**

**<font color='red'>建议：</font>**在开启事务后，立马执行`select ... for update`，对记录加 **[Next-Key Lock](./锁.html#next-key-lock)**，可以最大程度避免幻读