# Redis Sentinel

在 **[Redis 主从复制](./Redis主从复制.html#主从复制的痛点)** 文末提到了一些痛点：

- 当主节点出现故障时，无法自动故障转移，需要人工干预
- 主节点的写操作和存储能力都受到单机的限制

对于如何解决第一个痛点，正是本片文章要介绍的；对于第二个痛点，在 **[Redis 集群](./Redis-Cluster.html)** 中有得到改善

Redis Sentinel (哨兵) 核心任务就是**自动实现故障转移**，当发现主节点挂了后，需要选择一个从节点成为新的主节点，然后告之其它从节点更新自己的主节点，旧主节点恢复后成为新主节点的从节点

从 Redis Sentinel 自动故障转移的过程中提炼出它可以提供的功能：

- **监控：**Sentinel 节点会定期检查 Redis 节点以及其它 Sentinel 节点的状态是否正常
- **故障转移：**挑选一个从节点晋升为主节点，并维护后续正确的主从关系
- **通知：**Sentinel 节点会将故障转移结果通知给应用方 (系统管理员或者其它计算机程序)
- **配置提供：**客户端连接 Sentinel 节点获取主节点信息。如果发送故障转移，Sentinel 会通知新的主节点信息给客户端

### <font color='#1FA774'>Sentinel 和 Redis 的关系</font>

Sentinel 节点本质上是一种特殊的 Redis 节点，不提供读写服务，默认运行在 26379 端口上。Sentinel 拥有自己专门的命令表，也就是普通模式运行下的 Redis 无法使用这些特殊的命令

Sentinel 的启动方式有如下两种：

```bash
redis-sentinel /path/to/sentinel.conf
redis-server /path/to/sentinel.conf --sentinel
```

在没有 Sentinel 之前，Redis 还无法实现真正的高可用，虽然可以通过主从复制实现故障转移，但却无法自动化，需要人工干预。加入了 Sentinel 后，实现了 Redis 集群的高可用

Sentinel 和 Redis 的关系如下图所示：

![13](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230420/0321411681932101n0qL2L13.svg)

**<font color='red'>注意：</font>**只需要告诉 Sentinel 主节点的信息即可，不需要从节点信息，因为主节点中包含从节点信息，Sentinel 可以通过主节点获取所有从节点信息

### <font color='#1FA774'>三个定时监控任务</font>

下面要介绍的三个定时监控任务是建立在 Sentinel 节点和 Redis 节点之间可以相互通信的基础上

Sentinel 节点会和 Redis 节点之间会建立两个连接：命令连接和订阅连接，分别用于发布命令、接收命令以及发布订阅消息、接收订阅消息

Sentinel 节点会和 Sentinel 节点之间会建立命令连接，用于 Sentinel 之间发送命令和接收命令

Sentinel 和 Redis 的通信关系如下图所示：

![14](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230420/0115311681924531vMftwb14.svg)

**定时监控任务一：**每隔 10s，每个 Sentinel 节点会向主节点和从节点发送`info`命令获取最新的拓扑结构

**定时监控任务二：**每隔 2s，每个 Sentinel 节点会向 Redis 节点的`__sentinel__:hello`频道发送该 Sentinel 节点对于监控节点的判断以及当前 Sentinel 节点的信息。每个 Sentinel 节点都会订阅该频道，用来了解其它 Sentinel 节点以及它们对 Redis 节点的判断

**定时监控任务三：**每隔 1s，每个 Sentinel 节点会向主从节点和其它 Sentinel 节点发送一条`ping`命令做一次心跳检测，来确认这些节点当前状态是否正常

### <font color='#1FA774'>Sentinel 检测节点下线</font>

Sentinel 节点监控 Redis 节点就是为了能及时发现是否有节点处于下线，而检测节点下线的方法无非就是向定时向节点发送一条命令，如果在规定时间内没有回应，就可以认为下线

如果自己认为下线就单方面认为它一定下线，这种判断方式有一定局限性；只有当超过一定数量的 Sentinel 节点都认为它下线才认为它下线，这种判断方式更为合理

我们可以将下线分为两种：**主观下线**和**客观下线**

- 当 Redis 节点超过`down-after-milliseconds`时间内没有回复当前 Sentinel 节点的`ping`命令，只是单方面的认为它下线 (主观下线)
- 当超过`<quorum>`数量的 Sentinel 节点都认为某个 Redis 节点，那么它才算真正的下线 (客观下线)

当前 Sentinel 节点如何才能知道有多少其它 Sentinel 节点也认为某个 Redis 节点下线了呢？这就需要用到上一部分介绍的**定时监控任务二**

这也是为什么建议部署多个 Sentinel 节点的原因！！因为一个人认为你可以，你不一定真的可以；但超过一半人认为你可以，你才是真的可以

当一个主节点被认为客观下线后，就需要进行后续的故障转移，也就是挑选出新的主节点，以及维护正确的主从关系

这里还想再扩展一波，根据上面介绍的内容，只有主节点是高可用，从节点不是高可用。也就是主节点故障了可以挑选出新的主节点来一波故障转移，但是当从节点故障了无法转移

对于读写分离的场景，如果一个客户端只进行读操作，被分配给某个从节点，当这个从节点挂了后，客户端就和它失联，因为从节点没有故障转移

**读写分离高可用的设计思路：**维护一个从节点资源池，池中的从节点都是正常状态。当一个从节点挂了，就从资源池中删除，当一个从节点恢复了，就重新加入到资源池。客户端每次都从资源池选择一个可用的从节点，如果使用过程中从节点挂了，只需要从资源池中换一个从节点重新连接即可

![15](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230420/0217151681928235QMQcNm15.svg)

### <font color='#1FA774'>选举 Sentinel Leader</font>

在确定了主节点客观下线后，就需要进行后续的故障转移，第一步就需要挑选出一个从节点来当新的主节点

现在问题来了，由谁去挑选从节点呢？当然是 Sentinel 节点喽！

那么问题又来了，这么多 Sentinel 节点，由哪一个 Sentinel 节点呢？公平公正，选举产生，当选的 Sentinel 节点被称之为 Leader

选举 Leader 需要用到分布式领域的**共识算法**，简单来说就是让系统中的节点达成共识，达成共识的节点就是 Leader

大部分共识算法都是基于 Paxos 算法改进而来，在 Sentinel 节点选举中使用的是 Raft 算法，下面给出选举的大致思路：

- 每个 Sentinel 节点都有资格成为 Leader。当 Sentinel 节点确认主节点客观下线后，会向其它 Sentinel 节点发送`sentinel is-master-down-by-addr`命令，要求将自己设置为 Leader
- 收到命令的其它 Sentinel 节点，如果没有同意过其它节点的`sentinel is-master-down-by-addr`命令，那么就同意该请求，否则就拒绝
- 如果该 Sentinel 节点发现自己的票数已经 $\ge \max(quorum, num(sentinels) / 2 + 1)$，那么它将成为 Leader
- 如果此轮投票没有选举出 Leader，那么将进入下一轮选举

### <font color='#1FA774'>故障转移</font>

通过前两部分，既确定了下线主节点，又选举出了负责故障转移的 Sentinel Leader，现在万事具备，只欠东风！！也就是需要在从节点中选择一个最优的从节点作为新的主节点

这个最优的标准是什么呢？有三个方面：(优先级依次降低)

- **slave 优先级：**可以通过`slave-priority`为从节点设置优先级，范围`[0, 100]`，值越小优先级越高。特殊地，0 表示不参与选举。如果优先级相同，再比较下面的复制进度
- **复制进度：**Sentinel 总是希望选择出数据最完整，也就是与旧主节点数据最接近的从节点，可以通过复制偏移量计算出和旧主节点差距最小的从节点。如果复制进度相同，再比较下面的 runId
- **runId：**选择 runId 最小的从节点

**<font color='red'>注意：</font>**会先过滤掉不健康的从节点，即：主观下线、客观下线、5s 内没有回复 Sentinel 的`ping`、与主节点失联超过`down-after-milliseconds * 10`秒

选择出了成为新的主节点的从节点后，对从节点执行`salveof no one`摆脱旧主节点的束缚，然后给剩余从节点发送命令让它们更新自己的主节点

Sentinel 节点集合会将旧的主节点更新成从节点，并保持对它的关注，一旦恢复上线后，会让它去复制新的主节点

![16](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230420/0306491681931209NbF9y116.svg)

### <font color='#1FA774'>参考文章</font>

- **Redis 设计与实现**
- **Redis 开发与运维**
- **[Redis Sentinel：如何实现自动化地故障转移？](https://www.yuque.com/snailclimb/mf2z3k/ft4h1g)**