# redo log

### <font color=#1FA774>redo log 是什么？</font>

InnoDB 是以页为基本单位来管理内存空间，为了减少磁盘 IO 的开销，增加了 **[Buffer Pool](./InnoDB的Buffer-Pool.html)**，每次访问页面后不着急刷新回磁盘，而是缓存到 Buffer Pool 中，以便下次继续访问

Buffer Pool 在内存中，所以会导致后续对页面的修改并不能及时刷新回磁盘，如果发生断电或者一些其它的原因会使内存中的数据丢失，重启后啥也没了，必须持久化到磁盘才安全

一个很简单的方法就是每次修改后立刻将页面刷新回磁盘，这样做的弊端在于：

- 如果是少量的修改就刷新一个页面回磁盘，太过于浪费
- 一个修改可能会牵连多个页面的变化，而且这些页面物理上极大可能不在一起，这样刷新回磁盘需要进行很多随机 IO，开销更大

基于上面的场景，就有了 redo log，它记录了**<font color='red'>对哪个表空间哪个页进行了哪些修改</font>**，可以简单理解为就算数据丢失，但只要有对应的 redo log，也可以恢复丢失的数据

redo log 也需要及时的刷新回磁盘，但它相比于刷新整个页面的优势在于：

- redo log 占用的空间非常小
- redo log 是顺序写入磁盘，比随机 IO 快很多

### <font color=#1FA774>redo log 写入过程</font>

为了更好的管理 redo log，InnoDB 将 redo log 存储到大小为 512 字节的 block 中 (block 可以理解为页) 的 log blcok body 部分。每个 block 的结构如下所示：

![15](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230501/0510001682889000LolYCa15.svg)

和 Buffer Pool 一样，为了缓解磁盘 IO 速度慢的问题，InnoDB 引入了 log buffer，它是内存中一片连续的区域，默认大小为 16MB

redo log 是顺序写入 log buffer 中，从前往后，有一个 buf_free 指向空闲区域的起始位置，具体结构如下图所示：

![16](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230501/0519441682889584T85M4Q16.svg)

### <font color=#1FA774>redo log 刷盘时机</font>

log buffer 在内存中，如果不及时的将 log buffer 刷盘，也会导致 redo log 丢失，所以 log buffer 会在出现下面情况时刷盘

- **log buffer 空间不足时：**如果 redo log 已经占满了 log buffer 一半的空间，就会把 redo log 刷新回磁盘
- **事务提交时：**在事务提交时，将对应的 redo log 刷新回磁盘。由于 redo log 是顺序写入，所以之前的也会一起刷新回磁盘，不可能只刷新回某一个事务的 redo log
- **脏页刷新回磁盘时：**将某个脏页刷新回磁盘时，会保证先将该脏页对应的 redo log 刷新回磁盘
- 后台线程每隔 1s 将 log buffer 中的 redo log 刷新回磁盘
- MySQL 正常关闭时也会将 buffer log 刷新回磁盘

对于上面第二点刷盘情况：**事务提交时**，由于这个要求过于严格，会降低数据库新能，所以可以通过参数`innodb_flush_log_at_trx_commit`调整：

- **0：**每次事务提交时，不主动刷盘，交由后台线程来处理，所以可能丢失上一秒所有事务的数据
- **1：**每次事务提交时，都将 redo log 同步到磁盘，可以保证事务的持久性 (默认值)
- **2：**每次事务提交时，都只把 redo log 写入到 page cache (操作系统的缓存)。如果数据库挂了，但操作系统没挂，依旧可以保证事务的持久性

上面三个参数的安全性和性能的关系：

- 安全：1 > 2 > 0
- 性能：0 > 2 > 1

**<font color='red'>重点：</font>**对于安全性要求高的系统，只能选择 1；对于可以接受 1s 数据丢失的信息，可以选择 0；如果既要求性能也要保证一定的安全性，可以选择 2，折中方案

### <font color=#1FA774>redo log 文件组</font>

当 log buffer 触发刷盘时，会存到 redo log 文件中，它是以日志文件组的形式出现，即：一组中有多个大小相等的日志文件，默认情况下有两个文件：ib_logfile0 和 ib_logfile1

redo log 是为了记录内存中还没有刷盘的脏页，防止当 MySQL 挂了后脏页丢失，导致修改数据的丢失。所以当脏页已经被刷盘，那么对应的 redo log 就没有用

随着系统的运行，redo log 只会越来越多，而日志文件组的数量和大小都是固定的，所以它采用循环写的方式，从头开始写，当写到末尾又回到开头，这样可以在有限的日志文件组中无限的写入 redo log

之所以可以覆盖日志文件组开头的数据，是因为开头的数据可能会随着脏页的刷盘而变成无用。用 write pos 表示新 redo log 刷盘时要写入的位置；用 checkpoint 表示当前要擦除的位置

![17](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230501/1903361682939016BZkSK817.svg)

图中：

- write pos 和 checkpoint 都是顺时针移动
- write pos - checkpoint 之间的部分 (红色)，用来存新的 redo log
- checkpoint - write pos 之间的部分 (蓝色)，表示待刷盘的脏页对应的 redo log

当 write pos 追上 checkpoint 时，表示 redo log 文件组满了，不能再将 redo log 刷盘到文件组中

此时 MySQL 会阻塞，将 Buffer Pool 中的脏页刷新到磁盘，然后标记 redo log 文件组可以被擦除的部分，最后将可以被擦除的部分擦除，更新 checkpoint (顺时针向前移动)

所以，一次 checkpoint 的过程就是将脏页刷新到磁盘中，更新 redo log 日志组中可以被覆盖的位置