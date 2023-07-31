# Redis 缓存更新策略

缓存能够有效地加速应用的读写速度，因为它基于内存，减少了磁盘 IO 的开销。但它也存在数据不一致的问题，因为缓存层和存储层的数据存在着一定**时间窗口**的不一致性

给出加入缓存层的结构图：

![21](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230422/0351201682106680dvx17C21.svg)

下面介绍三种缓存更新策略，没有最优，只有根据业务场景最合适！！

### <font color='#1FA774'>旁路缓存模式 (Cache Aside Pattern)</font>

旁路缓存模式是平时使用较多的一个读写缓存模式，比较**<font color='red'>适合于读请求较多的场景</font>**

写操作的步骤：

- 更新 DB
- 直接删除 Cache

读操作的步骤：

- 从 Cache 中读取数据，读取到就直接返回
- 如果 Cache 读取不到，从 DB 中读取数据返回
- 将 DB 中读取到的数据写入 Cache 中

![22](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230422/0413261682108006GmtZUD22.svg)

**<font color='red'>问题一：</font>先更新 DB，再删除 Cache，能保证数据一致性吗？**

理论上来说还是可能会出现数据不一致的问题，但概率很小，因为删除 Cache 的操作速度快，使 Cache 中和 DB 中数据不一致的时间窗口很短

只有在不一致时间窗口内读数据才会出现数据不一致问题，即：读到旧值，如下图

![23](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230422/0432221682109142r85fEK23.svg)

下面举一个数据不一致的例子：线程 1 写数据 A，随后线程 2 读数据 A。正确的情况下线程 2 读到的应该是数据 A 的新值，但如果以下面的顺序执行：

- 线程 1 更新 DB
- 线程 2 读数据 A，直接从缓存中读取并返回，但缓存中是旧值
- 线程 1 删除 Cache

此时线程 2 读到的是数据 A 的旧值

**<font color='red'>问题二：</font>为什么删除 Cache，而不是更新 Cache？**

- 浪费服务端资源：不确定新写入的数据是否会被访问，而且 Cache 存放的数据一般需要经过服务端大量的计算。如果不会被访问又进行了大量的计算，无疑是白白浪费资源
- 增加数据不一致的概率：更新 Cache 比 删除 Cache 更加耗时，所以会增大不一致时间窗口的大小，进而提高数据不一致性的概率

![24](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230422/0442591682109779JwGrqV24.svg)

**<font color='red'>问题三：</font>可以先删除 Cache，后更新 DB 吗？**

显然不可以，这样会导致数据不一致的时间窗口变大，进而增加了出现数据不一致问题的概率

![25](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230422/0445131682109913fPAjn325.svg)

下面举一个数据不一致的例子：线程 1 写数据 A，随后线程 2 读数据 A。正确的情况下线程 2 读到的应该是数据 A 的新值，但如果以下面的顺序执行：

- 线程 1 删除 Cache
- 线程 2 读数据 A，此时缓存中没有数据，从 DB 中读取并返回，同时存入 Cache 中，Cache 中存入的是旧值
- 线程 1 更新 DB

此时线程 2 读到的是数据 A 的旧值，而且 Cache 存入的也是旧值。如果 Cache 没有设置过期时间且未来一段时间没有新的写操作，那么后面读数据 A 时全都是 Cache 中的旧值！！

**<font color='red'>缺陷一：</font>首次请求数据不在 Cache 中**

- 可以提前将热点数据放入 Cache 中

**<font color='red'>缺陷二：</font>写操作频繁会导致 Cache 中数据频繁被删，会影响命中率？**

- 对于要求数据强一致性的场景：更新 DB 的同时更新 Cache，但需要用分布式锁保证这两个更新的原子性，否则存在线程安全问题
- 对于可接受短暂数据不一致的场景：更新 DB 的同时更新 Cache，为 Cache 设置一个较短的过期时间

### <font color='#1FA774'>读写穿透 (Read/Write Through Pattern)</font>

**<font color='red'>核心思想：</font>**更新 DB，缓存存在就更新，不存在就不管

读写穿透将 Cache 作为主要的数据存储，从中读取并将数据写入其中，Cache 服务自己负责将此数据读取和写入 DB，从而减轻了程序员的职责

这种模式平时使用很少，大概率因为 Redis 分布式缓存并没有提供 Cache 将数据写入 DB 的功能

写操作的步骤：

- 先检查 Cache，如果 Cache 没有，直接更新 DB
- 如果 Cache 中存在，直接更新 Cache，然后 Cache 服务会自己更新 DB

读操作的步骤：

- 从 Cache 中读取数据，读取到就直接返回
- 如果 Cache 读取不到，先从 DB 中加载，写入 Cache 后返回

![26](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230422/0515471682111747Uot9bm26.svg)

### <font color='#1FA774'>异步缓存写入 (Write Back Pattern)</font>

**<font color='red'>核心思想：</font>**只更新缓存，异步更新 DB

异步缓存写入和读写穿透很相似，两者都是由 Cache 服务来负责 Cache 和 DB 的读写。它们俩最大的不同在于：读写穿透是同步更新 Cache 和 DB；而异步缓存写入是异步更新 DB

消息队列中消息的异步写入磁盘、MySQL 的 InnoDB Buffer Pool 机制都用到了这种策略

异步缓存写入下 DB 的写性能非常高，非常适合一些数据经常变化又对数据一致性没有那么高的要求的场景，如：浏览量、点赞量等