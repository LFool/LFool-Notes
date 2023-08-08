# 浅记 RocketMQ 消息丢失问题

本篇文章主要浅浅介绍一下 RocketMQ 消息丢失问题及解决办法～～

RocketMQ 核心流程：**「生产者发送消息到 Broker」**->**「Broker 刷盘/同步」**->**「消费者从 Broker 中消费消息」**

下面情况可能会出现消息丢失：

- 生产者往 Broker 中发送消息
- Broker 刷盘
- Broker 主从同步
- 消费者从 Broker 中消费消息
- 整个 RocketMQ 宕机

### <font color=#1FA774>生产者往 Broker 中发送消息</font>

#### <font color=#9933FF>同步发送</font>

生产者往 Broker 中同步发送消息时可能会出现以下几种情况：

- 消息在发往 Broker 的途中丢失
- 消息已经发送到 Broker，但 ACK 丢失
- 消息已经发送到 Broker，同时 Broker 刷盘、主从同步成功 (SEDN_OK)
- 消息已经发送到 Broker，但 Broker 刷盘超时 (FLUSH_DISK_TIMEOUT)
- 消息已经发送到 Broker，但 Broker 主从同步超时 (FlUSH_SLAVE_TIMEOUT)
- 消息已经发送到 Broker，但 Broker 从节点不可用 (SLAVE_NOT_AVAILABLE)

对于前两种情况，生产者会在一定时间后重发消息，类似于 TCP 的超时重传机制；对于后四种情况，生产者会收到对应的状态，可根据状态采取不同策略处理消息

- 如果业务要求不严格，可认为后四种情况均为发送成功

- 如果业务要求严格，如：只有当消费者成功消费才算不丢失，只有 SEDN_OK 状态才算发送成功，其它三种状态均为失败，可将这些失败的消息存储到数据库，启动定时任务重发

#### <font color=#9933FF>事务消息</font>

除了使用同步发送方式保证消息不丢失外，还可以使用 RocketMQ 分布式事务保证消息不丢失

首先会向 Broker 发送一个 half 消息，如果规定时间内 Broker 没有收到本地事务的状态，会间隔规定时间去回查本地事务的状态

**<font color='red'>问题一：</font>**如果 half 消息发送失败怎么处理？

- 如果 half 消息发送失败可以认为 RocketMQ 服务有问题，可先记录一下，等 RocketMQ 服务可以正常服务后重新执行

**<font color='red'>问题二：</font>**如果本地事务订单入库失败怎么处理？

- 订单入库失败会抛出异常，如果需要实现自动订单入库可将异常订单缓存一下，等过段时间再次执行

**<font color='red'>问题三：</font>**如何优化的处理下单后等待支付成功？

- 当订单入库后，设置支付状态为等待支付，并给 Broker 返回一个 UNKNOWN 状态
- Broker 会定期回查本地事务状态，回查过程中检查订单的状态即可，通过设置回查次数和间隔时间可以设置等待支付时间

### <font color=#1FA774>Broker 刷盘</font>

Broker 刷盘有两种方式，同步刷盘和异步刷盘

- **同步刷盘：**当 Broker 收到消息后，并等待刷盘成功后再给生产者返回 ACK，保证了消息的可靠性，但会影响性能
- **异步刷盘：**当 Broker 收到消息后，直接写入 Page Cache 即可给生产者返回 ACK，保证了性能，但存在消息丢失的可能性

### <font color=#1FA774>Broker 主从同步</font>

和 Broker 刷盘一样，Broker 的主从同步也有两种方式，同步主从同步和异步主从同步

- **同步主从同步：**当 Broker 收到消息后，同步复制到从节点中，然后再给生产者返回 ACK，保证了主节点宕机后，从节点有完整的数据
- **异步主从同步：**当 Broker 收到消息后，异步复制到从节点中，直接给生产者返回 ACK，保证了性能，但主节点宕机后存在消息丢失的可能性

### <font color=#1FA774>消费者从 Broker 中消费消息</font>

当 Broker 推送消息给消费者，但如果规定时间内没有收到消费者返回的 ACK，表示可能是消息消费失败或者 ACK 丢失，那么 Broker 会重新推送消息给消费者

上面机制一定能保证消息成功被消费者消费，但要注意的是必须基于同步消费的前提下，如果是异步消费就可能消费失败的情况

### <font color=#1FA774>整个 RocketMQ 宕机</font>

当整个 RocketMQ 宕机时，只能设计一个降级方案，比如将消息先缓存到某个地方，当 RocketMQ 恢复后再重新处理

### <font color=1FA774>零丢失方案</font>

- 站在生产者角度，可以使用同步发送或事务消息
- 站在 Broker 角度，可以使用同步刷盘和同步主从同步
- 站在消费者角度，可是使用同步消费消息

### <font color=1FA774>参考文章</font>

- **[RocketMQ如何保证消息不丢失? 如何快速处理积压消息？](https://blog.csdn.net/qq_45076180/article/details/113828472)**
- **[[面试官再问我如何保证 RocketMQ 不丢失消息,这回我笑了！](https://www.cnblogs.com/goodAndyxublog/p/12563813.html)](https://www.cnblogs.com/goodAndyxublog/p/12563813.html)**