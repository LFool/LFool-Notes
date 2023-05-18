# CPU 缓存一致性

### <font color=#1FA774>存储器层次结构</font>

在计算机系统中，最常见的存储器有内存和磁盘。一个进程运行过程中往往需要将磁盘中的数据加载到内存中才可以访问

CPU 的运算速度和内存的访问速度相差过大，为了缓解这种差距，在 CPU 和内存之间增加了高速缓存。由于 CPU 是基于寄存器的指令集，所以在 CPU 和高速缓存之间还存在寄存器

存储器离 CPU 越近，那么读写速度就越快，但能存储的容量也就越小。一个存储器的层次结构如下图所示：

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230519/0139571684431597VAWjOA1.svg)

在现在的 CPU 中一般都有多个核心，寄存器、L1 高速缓存、L2 高速缓存都是每个核心独有，而 L3 高速缓存是 CPU 独有，如下图所示：

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230519/0148571684432137nims2E2.svg)

### <font color=#1FA774>CPU Cache 结构</font>

每个 CPU Cache 都是由缓存行构成，而且 CPU 和 CPU Cache 的交互单位也是缓存行，也就是如果 Cache 中没有数据，CPU 会一次性从内存中读取和缓存行相同大小的数据存入 Cache 中

CPU Cache 的结构如下图所示：

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230519/0200591684432859n0JMaD3.svg)

在使用 CPU Cache 的过程中，可能会出现伪共享问题，这也是下面要介绍的 MESI 协议导致的。**关于伪共享的分析可见 [伪共享](../java/伪共享.html)**

### <font color=#1FA774>CPU Cache 数据写入</font>

对于读操作来说很简单，每次先去 Cache 中检查是否有数据，如果有数据就直接返回，如果没有数据就去内存中读，并将读到的数据加到 Cache 中以便下次可以快速读

但是对于写操作可能就麻烦一点，这里可以借鉴 **[Redis 缓存更新策略](../redis/Redis缓存更新策略.html)**，下面主要介绍两种写入策略：

- **写穿透 (Write Through)：**如果 Cache 中有数据，先更新 Cache，后更新内存；如果 Cache 中没有数据，直接更新内存
- **写回 (Write Back)：**先只更新 Cache，随后异步更新内存

### <font color=#1FA774>CPU 缓存一致性问题</font>

不同线程在不同核心上执行，每个核心存在自己独有的 Cache。由于不可见性，同一个共享变量，可能出现在不同 Cache 中不一致的问题，**原理可见 [处理器重排序](../java/Java内存模型.html#处理器重排序)**

要解决缓存不一致的问题，必须保证下面两点：

- 某个 CPU 核心更新 Cache 中的数据时，其它 CPU 核心必须知道该数据被更新了，这被称为**写传播 (Write Propagation)**
- 所有 CPU 核心看到 Cache 中数据的修改操作顺序必须是一致性的，这被称为**事务的串行化 (Transaction Serialization)**

对于第一点使用总线嗅探机制，当某个 CPU 核心更新 Cache 中的数据时，会向总线广播这一事件，所有 CPU 核心都会监听总线上的广播事件，当发现广播事件中包含的数据也存在于自己的 Cache 中时，就会将自己 Cache 中的数据更新成广播中的数据

但仅仅做到第一点并不能完全解决缓存不一致的问题，假设存在两个 CPU 核心先后将`i`修改成 100 和 200，但无法保证这两个修改的广播事件的先后顺序，可能出现某个 CPU 核心看到的顺序：先将`i`修改为 200，再将`i`修改为 100，这也是第二点要保证事务串行化执行的原因

### <font color=#1FA774>MESI 协议</font>

MESI 协议基于总线嗅探机制实现了事务串行化，该协议保证了缓存一致性。首先，MESI 协议规定了四种状态：

- **Modified** 已修改：表示 Cache 中的数据已经被修改过，也就是脏数据
- **Exclusive** 独占：表示 Cache 中的数据被自己独占，其它 CPU Cache 中没有该数据，此时数据是干净的
- **Shared** 共享：表示多个 CPU Cache 中都有该数据，此时数据是干净的
- **Invalidated** 已失效：表示 Cache 中的数据无效

下面通过表格给出这四种状态之间的转化关系：

![4](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230519/0343541684439034g86EGQ4.svg)

**<font color='red'>注意：</font>**如果在独占状态或者已修改状态下写数据，不需要广播该事件。如果是独占状态，表示其它核心的 Cache 中没有该数据，所以不需要广播；如果是已修改状态，表示包含该数据的核心的 Cache 中对应的 Cache Line 已经是已失效状态，所以也不需要广播

这里推荐一个可视化网站，可以看到 Cache 四个状态的转换 **[VivioJS MESI animation help](https://www.scss.tcd.ie/Jeremy.Jones/VivioJS/caches/MESIHelp.htm)**