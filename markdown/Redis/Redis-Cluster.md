# Redis Cluster

在 **[Redis 主从复制](./Redis主从复制.html#主从复制的痛点)** 文末提到了一些痛点：

- 当主节点出现故障时，无法自动故障转移，需要人工干预
- 主节点的写操作和存储能力都受到单机的限制

对于第一个痛点，可以通过 **[Redis Sentinel](./Redis-Sentinel.html)** 来实现自动故障转移；对于如何解决第二个痛点，正是本片文章要介绍的

即然受到了单机的瓶颈，那正常思路就是横向扩展，把数据分散存储到 n 个 Redis 节点中，且没有重复的部分。当客户端要进行读写操作时，可以根据路由规则将请求重定向到集群中正确节点上

同时为了使集群具有高可用性，可以为集群中的每个主节点配置一个或多个从节点，当主节点挂了后可以进行故障转移，这一部分和主动复制 + Sentinel 实现高可用原理一样

通过集群横向扩展后，不仅提高了系统的存储能力和并发量，而且还可以根据实际的情况动态调整集群的节点数量，如：删除节点、添加节点

通过上面的介绍，总结一下 Redis Cluster 的优势：

- 以分片的形式管理数据，可以横向扩展，提高系统的存储能力和并发量，支持动态的扩容和缩容
- 具备主从复制，故障转移等开箱即用的功能，内置 Sentinel 机制，无须单独部署 Sentinel 集群

最后来一张 Redis Cluster 结构图

![17](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230421/0440441682023244X60wpu17.svg)

### <font color='#1FA774'>数据分片</font>

数据分片的核心就是将全量数据按照一定规则划分成 n 个子集存储在 m 个节点上

![18](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230421/0232371682015557sheCtt18.svg)

Redis Cluster 采用虚拟槽分区，所有的 key 根据哈希函数映射到`[0, 16383]`整数槽内，计算公式：`slot = CRC16(key) & 16383`，其中`CRC16(key)`表示计算 key 的 CRC-16 校验码

每一个节点负责维护一部分槽以及槽所映射的键值对数据，16384 个槽必须都指派给主节点集群才会处于可用状态，也可以通过配置`cluster-require-full-coverage no`取消该限制

Redis 虚拟槽分区通过引入槽的概念实现了数据和节点的解耦合。当客户端需要写入一个数据，只需要关心该数据应存入的槽，不需要关心存放在哪个节点中，槽和节点的映射由节点自身维护

这样设计的好处在于节点扩容和收缩时非常方便，只需要以槽为单位移动数据即可，而且还能保证扩容或收缩后的数据分布的均匀性

每个节点都保存所有槽与所有节点的映射关系，也就是集群中的任意一个节点都知道某个槽被指派给哪个节点。节点和节点之间还会定时发送`ping`消息，用于检测节点是否在线以及交换彼此的状态信息

状态信息中就包含了当前节点被指派的槽，从而目标节点就可以更新自己的信息。在一定时间内，集群中的所有节点都会知道其它节点的槽映射，而且是动态更新 (P2P 去中心化通信)

下面给出一些对 slot 的常用命令：

```bash
# 将 slot 1 2 3 4 5 分配给节点
CLUSTER ADDSLOTS 1 2 3 4 5
# 将 slot 1 - 5 分配给节点
CLUSTER ADDSLOTSRANGE 1 5
# 从节点中删除 slot 1 2 3 4 5
CLUSTER DELSLOTS 1 2 3 4 5
# 从节点中删除 slot 1 - 5
CLUSTER DELSLOTSRANGE 1 5
```

在这部分的最后，抛出一个常见问题：**<font color='red'>为什么 Redis Cluster 槽的数量是 16384 个？</font>**

CRC-16 算法产生的校验码有 16 位，理论可以产生 $2^{16}=65536$ 个值，为什么 Redis Cluster 槽的数量是 $2^{14}=16384$ 个呢？

在 2015 年，Redis 的作者 antirez 巨佬本人专门回答了这个问题 -> **[why redis-cluster use 16384 slots? #2576](https://github.com/redis/redis/issues/2576)**

**原因一：**集群中节点间会定时发送心跳包检测在线状态以及交换状态信息，状态信息中包含了该节点被指派的槽信息，是一个`char myslots[16384/8]`数组，每一位的 0/1 表示一个槽的指派状态，该数组大小刚好是 2KB。如果把槽数量设置为 65536 个，那么该数组大小将变为 8KB，这无疑会增加网络带宽和流量的开销

**原因二：**基于其它设计上的权衡，Redis Cluster 不太可能扩展超过 1000 个主节点，所以 16384 个槽已经够用

**原因三：**`myslots[]`记录了当前节点被指派的槽信息，如果槽数量增多，但主节点数不超过 1000，会导致每个主节点指派槽数量增多，进而在传输时，对`myslots[]`数组的压缩率就越低

### <font color='#1FA774'>一条命令完整执行过程</font>

有了上面的铺垫，我们重构一张更详细的 Redis Cluster 结构图，以及一条命令完整执行的过程图

![19](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230421/0506341682024794Twm5LW19.svg)

当客户端以单机模式启动：`redis-cli -h 127.0.0.1 -p 6379`，客户端会直接打印出 MOVED 错误：`(error) MOVED 866 127.0.0.1 6380`

当客户端以集群模式启动：`redis-cli -c -h 127.0.0.1 -p 6379`，客户端会直接重定向到目标节点：`-> Redirected to slot [866] located at 127.0.0.1 6380`

**<font color='red'>注意：</font>**自动重定向后，客户端对应的主节点就被永久改变成 slot 对应的主节点

```bash
127.0.0.1:6379> get hello world                         # 此时客户端连接的是 6379 节点
-> Redirected to slot [866] located at 127.0.0.1:6380
OK
127.0.0.1:6380>                                         # 此时客户端连接的是 6380 节点，而且后续都是和 6380 连接
```

### <font color='#1FA774'>重新分片</font>

在 **[数据分片](./Redis-Cluster.html#数据分片)** 部分介绍了如何给主节点指派槽以及如何删除指派给主节点的槽

重新分片是指可以将任意数量已经指派给某个主节点 (源节点) 的槽更改为指派给另一个主节点 (目标节点)，并且相关槽所属的键值对也会从源节点移动到目标节点

重新分片操作可以在线进行，也就是在重新分片的过程中，集群不需要下线，并且源节点和目标节点都可以继续处理命令请求

Redis 重新分片操作是由 Redis 集群管理软件 redis-trib 负责执行的，该软件对单个槽进行重新分片的步骤如下：

- redis-trib 对目标节点发送`CLUSTER SETSLOT <slot> IMPORTING <source_id>`命令，让目标节点准备好从源节点 source_id 导入属于槽 slot 的键值对
- redis-trib 对源节点发送`CLUSTER SETSLOT <slot> MIGRATING <target_id>`命令，让源节点准备好将属于槽 slot 的键值对迁移至目标节点 target_id
- redis-trib 向源节点发送`CLUSTER GETKEYSINSLOT <slot> <count>`命令，获得最多 count 个属于槽 slot 的键值对的键名
- 对于上个步骤获取的每个键名，redis-trib 都向源节点发送一条`MIGRATE <target_ip> <port> <key_name> 0 <timeout>`命令，将被选中的键原子地迁移至目标节点
- 重复上两个步骤，直至源节点槽 slot 中所有键值对都迁移至目标节点中
- redis-trib 向集群中任意一个节点发送`CLUSTER SETSLOT <slot> NODE <target_id>`命令，将槽 slot 指派给目标节点，这一指派信息会通过消息发送至整个集群

![20](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230421/0603131682028193mMSpNp20.svg)



在重新分片期间，源节点向目标节点迁移一个槽的过程中，可能槽中部分键值对还在源节点中，部分键值对已经被迁移到目标节点中

此时客户端对源节点发送关于键 key 的命令，会有两种情况：「key 还在源节点中」or「key 已经迁移至目标节点中」

- 如果 key 还在源节点中，源节点直接响应客户端的命令
- 如果 key 已经迁移至目标节点中，源节点会返回给客户端一个`ASK`错误，让客户端去目标节点中找
- 客户端收到`ASK`错误后，会向目标节点发送一条`ASKING`命令，目标节点会破例执行一次命令，但`ASKING`命令是一次性的，也就是下次还需要执行需要重新发`ASK`命令
- 目标节点收到`ASKING`命令后，会检查 key 是否还在导入且没有导入完成，如果是，就返回重试错误`TRYAGAIN`
- 客户端向目标节点发送真正需要请求的命令
- ASK 重定向并不会同步更新客户端缓存的哈希槽指派信息，也就是客户端对正在迁移的相同哈希槽的请求依旧会发送到源节点而不是目标节点

**<font color='red'>比较 MOVED 重定向和 ASK 重定向</font>**

- MOVED 重定向是一种永久重定向，也就是后续命令都会发送给新的节点
- ASK 重定向是一种临时重定向，也就是后续命令依旧发动给原来的节点

在文章开头提到集群支持动态的扩容和缩容，其实核心就是重新分片。集群扩容就是将已有节点的部分槽迁移到新加入的节点；集群缩容就是将要删除的节点的全部槽均匀迁移至其它节点

### <font color='#1FA774'>节点通信</font>

Redis Cluster 采用 P2P 的 Gossip (流言) 协议，节点彼此不断的通信交换信息，一段时间后所有节点都会知道集群完整信息，类似于流言传播

集群中每个节点都会单独开辟一个 TCP 通道，用于节点之间彼此通信，通信端口号在基础端口号上加 10000

每个节点在固定周期内通过特定规则选择几个节点发送`ping`消息，接收到`ping`消息的节点用`pong`消息作为响应

常用的 Gossip 消息有四种：`meet`消息、`ping`消息、`pong`消息、`fail`消息

- `meet`消息：用于通知新节点加入。消息发送者通知消息接收者加入当前集群
- `ping`消息：集群内最频繁的消息，集群内每个节点每秒向多个其它节点发送`ping`消息，用于检测节点是否在线和交换彼此状态信息。`ping`消息封装了自身节点和部分其它节点的状态数据
- `pong`消息：用作`meet/ping`消息的回复。`pong`消息封装了自身状态数据，也可以通过向集群广播自身的`pong`消息来通知其它节点更新自身状态
- `fail`消息：当节点判定集群内另一个节点下线时，会向集群内广播一个`fail`消息

所有的消息格式划分为：消息头和消息体。消息头包含发送节点自身状态数据，接收节点根据消息头就可以获取到发送节点的相关数据；消息体包含发送节点对其它节点的状态数据

```c
typedef struct {
    char sig[4];        /* 信息标志 */
    uint32_t totlen;    /* 消息总长度 */
    uint16_t ver;       /* 协议版本 */
    uint16_t port;      /* 端口号 */
    uint16_t type;      /* 消息类型，用于区分 meet, ping, pong 等消息 */
    uint16_t count;     /* 消息体包含的节点数量，仅用于 meet, ping, pong 消息类型 */
    uint64_t currentEpoch;  /* 当前发送节点的配置纪元 */
    uint64_t configEpoch;   /* 主节点 / 从节点的主节点 的配置纪元 */
    uint64_t offset;    /* 复制偏移量 */
    char sender[CLUSTER_NAMELEN]; /* 发送节点的 node id */
    unsigned char myslots[CLUSTER_SLOTS/8]; /* 发送节点被指派的槽 */
    char slaveof[CLUSTER_NAMELEN];/* 如果发送节点时从节点，记录主节点的 node id */
    /* 省略部分属性 */
    unsigned char state; /* 集群状态 */
    unsigned char mflags[3]; /* 消息标识 */
    union clusterMsgData data; /* 消息的内容 */
} clusterMsg;  /* 消息 */

union clusterMsgData {
    /* PING, MEET and PONG */
    struct {
        /* Array of N clusterMsgDataGossip structures */
        clusterMsgDataGossip gossip[1];
    } ping;

    /* FAIL */
    struct {
        clusterMsgDataFail about;
    } fail;
    
    /* 省略部分属性 */
};

typedef struct {
    char nodename[CLUSTER_NAMELEN]; /* 节点的 node id */
    uint32_t ping_sent;         /* 最后一次向该节点发送 ping 消息的时间 */
    uint32_t pong_received;     /* 最后一次接收该节点 pong 消息的时间 */
    char ip[NET_IP_STR_LEN];    /* IP */
    uint16_t port;              /* base port last time it was seen */
    uint16_t flags;             /* 该节点的标识 */
    /* 省略部分属性 */
} clusterMsgDataGossip;
```

接收节点收到`meet/ping`消息时，执行解析消息头和消息体的流程：

- **解析消息头的过程：**消息头包含了发送节点的信息，如果发送节点时新节点且是`meet`消息，则加入到发送节点的集群；如果是已知节点，则尝试更新发送节点的状态，如：槽映射关系、主从角色等关系
- **解析消息体的过程：**如果消息体的`clusterMsgDataGossip`数组包含的是新节点，则尝试发起与新节点的`meet`握手流程；如果是已知节点，则根据`flags`字段判断该节点是否下线，用于故障转移

### <font color='#1FA774'>故障转移</font>

**<font color='red'>注意：</font>**在集群中，主节点才负责读写请求和集群槽等关键信息的维护，而从节点仅仅只是主节点数据和状态信息的复制，只用于故障转移时被选举成为新的主节点

Redis Cluster 也可以实现自动故障转移，而且和 **[Redis Sentinel](./Redis-Sentinel.html)** 的故障转移很相似，但却不见哨兵的影子。因为实现哨兵功能的其实是集群中的主节点，也就是主节点扮演了哨兵的角色

集群中也有 **[主观下线和客观下线](./Redis-Sentinel.html#sentinel-检测节点下线)** 的概念，但更准确的来说是 pfail (疑似下线) 和 fail (下线)

当集群内一个节点向其它部分节点发送了`ping`消息，但是在规定时间内没有收到对方的`pong`回复，就可认为对方处于 pfail 状态，该节点的状态会跟随消息在集群内传播

`ping/pong`消息的消息体会携带集群 1/10 的其它节点状态数据，当接收节点发现消息中含有 pfail 状态的节点时，会在本地找到该节点，保存到该节点的下线报告链表中

需要注意的是，如果是从节点发送的消息包含 pfail 节点，直接忽略，即不会加入到下线报告链表中；如果下线报告链表中已经有某主节点，也不会加入链表，但是会更新

集群中也没有类似 **[选举 Sentinel Leader](./Redis-Sentinel.html#选举-sentinel-leader)** 的概念，Sentinel Leader 主要用于选择一个最优的从节点作为新的主节点，而集群中新的主节点是根据所有主节点在从节点中投票产生的，也就是新主节点的产生和 Sentinel Leader 的产生很相似！

集群中的节点每次接收到其它节点的 pfail 状态时，都会尝试触发 fail，也就是判断下线报告是否有超过一半的主节点数量 (包括下线的主节点)，如果成立，该节点会向集群内广播一条`fail`消息

该`fail`消息有两个作用：

- 通知集群内其它节点将该节点标记为下线且立刻生效
- 通知 fail 节点的从节点触发故障转移流程

只有主节点才有投票的机会，且每轮只有一次投票机会。当一个从节点获得超过一半主节点的票，那么它就被选举成为新的主节点；如果该轮投票中没有从节点获得超过一半主节点的票，则进行下一轮投票

新当选主节点的从节点会做三件事情：

- 执行`slaveof no one`命令摆脱旧的主节点，自己当主节点
- 把指派给旧主节点的槽指派给自己，也就是重新分片的过程
- 向集群中其它节点发送`pong`消息，让其它节点知道自己成为了新的主节点，且接管了旧主节点负责的槽

**<font color='red'>注意：</font>**必须至少有 3 个主节点，否则当一个主节点故障后，无法顺利故障转移。假设主节点数量 $N = 2$，当一个主节点下线后，它的从节点必须获得 $N/2 + 1 = 2$ 票才能成为新的主节点，可是集群中只有一个可用的主节点，所以无法顺利故障转移

### <font color='#1FA774'>参考文章</font>

- **Redis 设计与实现**
- **Redis 开发与运维**
- **[Redis Cluster：缓存的数据量太大怎么办？](https://www.yuque.com/snailclimb/mf2z3k/ikf0l2)**