# Redis 基本数据结构

在 Redis 中一共有五种对外数据结构，分别是：String (字符串)、Hash (哈希)、List (列表)、Set (集合)、ZSet (有序集合)

之所以说是对外数据结构，因为这里类似于 Java 中接口和实现类的关系，基本对外数据结构是暴露出来的接口，具有统一的 API，而这些对外数据结构可以有不同的底层实现，Redis 中称为内部编码

Redis 创建了一个对象系统，上述五种对外数据结构分别有对应的对象：字符串对象、哈希对象、列表对象、集合对象、有序集合对象。Redis 是基于`<key, value>`的存储结构，每个 key 或 value 必须是上述五种对象的一种，其中 key 必须是字符串对象，如下图所示：

<img src="https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230416/1734021681637642ckEaig1.svg" alt="1" style="zoom:80%;" />

Rediis 中每个对象都由一个`redisObject`结构表示：

```c
struct redisObject {
    unsigned type:4;     // 对象类型，也就是五种对外数据结构中的一种
    unsigned encoding:4; // 内部编码，也就是对外数据结构的底层实现
    void *ptr;           // 指向底层实现数据结构的指针
    // ...
};
```

思考一下，Redis 为什么要使用这种设计方式呢？

这和 Java 中接口和实现类的很想，可是让使用和实现解耦，在不改变外部使用的情况下，可以更换内部编码；而且也可以根据不同场景切换不同内部编码

下面给出官方对数据结构的介绍：**[Redis Data Structures](https://redis.com/redis-enterprise/data-structures/)**、**[Redis data types tutorial](https://redis.io/docs/data-types/tutorial/)**

同时推荐一个网站 **[Redis 命令参考](http://doc.redisfans.com/)**

### <font color=#1FA774>字符串</font>

字符串类型是 Redis 中最基础的数据结构，key 必须是字符串对象，而且字符串对象是唯一可以被嵌套的对象，也就是其它四种类型中可以包含字符串类型

字符串类型是一种二进制安全的数据结构，可以用来存储任何类型的数据，比如：字符串、整数、浮点数、图片、序列化后的对象

在 C 语言中，字符串通过判断末尾是否为`'\0'`来判断字符串是否结束，这样就会导致不能存储`'\0'`，否则就会提前判断字符串结束

而在 Redis 中，每个字符串对象都有三个属性：字节数组、已使用长度、未使用长度，这样就通过已使用的长度来判断字符串的结束位置，从而可以避免 C 语言中的二进制不安全的情况

不仅如此，Redis 中的字符串可以杜绝缓冲区溢出，因为它可以监测是否有足够的空间容纳新的字符，否则就会扩容。而且为了减少扩容或缩容的内存分配开销，Redis 采用了空间预分配和惰性空间释放策略

#### <font color=#9933FF>常用命令</font>

这里就不一一介绍每种命令的用法，可以去 **[String（字符串）](http://doc.redisfans.com/string/index.html)**查看详细介绍！！但是这里给出常用命令的时间复杂度：

|              命令              |        时间复杂度        |           命令            | 时间复杂度 |
| :----------------------------: | :----------------------: | :-----------------------: | :--------: |
|         set key value          |           O(1)           |   incrby key increment    |    O(1)    |
|            get key             |           O(1)           |   decrby key decrement    |    O(1)    |
|       del key [key ...]        | O(k), k 是键的个数，下同 | incrbyfloat key increment |    O(1)    |
| mset key value [key value ...] |           O(k)           |     append key value      |    O(1)    |
|       mget key [key ...]       |           O(k)           |        strlen key         |    O(1)    |
|            incr key            |           O(1)           | setrange key offset value |    O(1)    |
|            decr key            |           O(1)           |  getrange key start end   |    O(n)    |

#### <font color=#9933FF>内部编码</font>

字符串类型有三种编码，会根据当前值的类型和长度来决定使用哪种内部编码

- **int：**8 byte 的长整型
- **embstr：**<= 39 byte 的字符串
- **raw：**> 39 byte 的字符串

#### <font color=#9933FF>应用场景</font>

**缓存功能：**在 Web 服务和 MySQL 之间加一层缓存，优先去 Redis 缓存中查询，如果没有再去 MySQL 中查询，并将查询结果存储到 Redis 中，以便下次可以快速查询到

**计数功能：**由于将计数更新到 MySQL 缓慢，可以先在 Redis 更新，等一段时间后再统一持久化到 MySQL 中，实现最终一致性

**共享 Session：**在分布式系统中，由于负载均衡可能导致用户两次连续请求被分配到不同服务器中，进而无法获取用户 Session，可以将 Session 数据统一放到 Redis 中集中管理

**限速功能：**只允许用户一分钟内获取 5 次验证码，可以通过`set key value ex 60 nx`命令实现，在 key 不存在情况下，生成一个 60s 过期的 key-value，然后通过自增不允许超过 5 次

### <font color=#1FA774>哈希</font>

几乎所有编程语言都提供了哈希类型。如果使用`hashtable`的内部编码实现哈希类型，那么内部维护了两个哈希表，其中一个扩容时使用，每次以一条链为单位移动元素 (渐进式 rehash)

当出现哈希冲突时，采用拉链法解决冲突，而且出于速度的考虑，使用头插法添加到链表的表头位置

#### <font color=#9933FF>常用命令</font>

这里就不一一介绍每种命令的用法，可以去 **[Hash（哈希表）](http://doc.redisfans.com/hash/index.html)**查看详细介绍！！但是这里给出常用命令的时间复杂度：

|                  命令                   |          时间复杂度           |               命令               | 时间复杂度 |
| :-------------------------------------: | :---------------------------: | :------------------------------: | :--------: |
|          hset key field value           |             O(1)              |        hexists key field         |    O(1)    |
|             hget key field              |             O(1)              |            hkeys key             |    O(n)    |
|       hdel key field [field ...]        | O(k), k 是 field 的个数，下同 |            hvals key             |    O(n)    |
|                hlen key                 |             O(1)              |      hsetnx key field value      |    O(1)    |
|               hgetall key               |  O(n)，n 是 field 总数，下同  |   hincrby key field increment    |    O(1)    |
|       hmget key field [field ...]       |             O(k)              | hincrbyfloat key field increment |    O(1)    |
| hmset key field value [field value ...] |             O(k)              |        hstrlen key field         |    O(1)    |

#### <font color=#9933FF>内部编码</font>

哈希类型有两种编码

- **ziplist：**当哈希表中的元素个数小于设置值 (默认 512)，同时所有键值的字符串长度都小于设置值 (默认 64 byte)，Redis 会使用 ziplist 作为哈希的内部编码
- **hashtable：**当哈希类型无法满足 ziplist 的要求时，会选择 hasttable 作为内部编码

#### <font color=#9933FF>应用场景</font>

**对象数据存储：**假设一个对象有三个属性，可以使用`hmset user:1 name tom age 23 city beijing`来存储

如果使用字符串类型，可能需要先将对象序列化，使用时在反序列化。相比于字符串类型，哈希类型更加的直观，使用简单，但占用的内存也更多

适用于字符串存储的情况：

- 每次都需要访问大量字段
- 存储结构具有多层嵌套

适用于哈希存储的情况：

- 大多数只需要访问少数字段
- 始终知道具体需要使用的字段，防止使用 mget 时获取不到想要的数据

### <font color=#1FA774>列表</font>

列表类型用来存储多个有序的字符串，允许重复，是一个双向队列，可以从前或从后插入元素

#### <font color=#9933FF>常用命令</font>

这里就不一一介绍每种命令的用法，可以去 **[List（列表）](http://doc.redisfans.com/list/index.html)**查看详细介绍！！但是这里给出常用命令的时间复杂度：

|                 命令                 |        时间复杂度        |         命令         | 时间复杂度 |
| :----------------------------------: | :----------------------: | :------------------: | :--------: |
|     rpush key value [value ...]      | O(k), k 是元素个数，下同 |       lpop key       |    O(1)    |
|     lpush key value [value ...]      |           O(k)           |       rpop key       |    O(1)    |
| linsert key before/after pivot value | O(n)，n 是距离头尾的距离 | lrem key count value |    O(n)    |
|        LRANGE key start stop         |         O(s + n)         | ltrim key start end  |    O(n)    |
|           lindex key index           |           O(n)           | lset key index value |    O(n)    |
|               llen key               |           O(1)           |     blpop/brpop      |    O(1)    |

#### <font color=#9933FF>内部编码</font>

列表类型有两种编码

- **ziplist：**当列表中的元素个数小于设置值 (默认 512)，同时元素长度都小于设置值 (默认 64 byte)，Redis 会使用 ziplist 作为列表的内部编码
- **linkedlist：**当列表类型无法满足 ziplist 的要求时，会选择 linkedlist 作为内部编码

#### <font color=#9933FF>应用场景</font>

**信息流展示：**由于列表有序，可以使用`lpush/lrange`获得最新的文章或者消息

**消息队列：**生产者从左边将消息放入队列，消费者从右边获取消息进行处理。但是实现过于简单，一般不建议这样使用

### <font color=#1FA774>集合</font>

集合类型是用来存储多个无序的字符串，集合内元素不允许重复。Redis 除了支持集合内的增删改查外，还支持多个集合取交集、并集、差集等

#### <font color=#9933FF>常用命令</font>

这里就不一一介绍每种命令的用法，可以去 **[Set（集合）](http://doc.redisfans.com/set/index.html)**查看详细介绍！！但是这里给出常用命令的时间复杂度：

|             命令             |        时间复杂度        |         命令         | 时间复杂度 |
| :--------------------------: | :----------------------: | :------------------: | :--------: |
| sadd key member [member ...] | O(k), k 是元素个数，下同 |       spop key       |    O(1)    |
| srem key member [member ...] |           O(k)           |     smembers key     |    O(n)    |
|          scard key           |           O(1)           | sinter key [key ...] |  O(n * m)  |
|     sismember key member     |           O(1)           | sunion key [key ...] |    O(n)    |
|   srangemember key [count]   |         O(count)         | sdiff key [key ...]  |    O(n)    |

#### <font color=#9933FF>内部编码</font>

集合类型有两种编码

- **intset：**当集合中的元素都是整数且元素个数小于设置值 (默认 512)，Redis 会使用 intset 作为列表的内部编码
- **hashtable：**当集合类型无法满足 intset 的要求时，会选择 hashtable 作为内部编码

#### <font color=#9933FF>应用场景</font>

**需要存放数据不能重复的场景：**网站 UV (独立用户访问) 统计 (数据量过大时 HyperLogLog 更合适)、文章点赞、动态点赞等场景

**需要获取多个数据交集、并集、差集的场景：**共同好友、共同粉丝、共同关注等场景

**需要随机获取数据源中元素的场景：**抽奖系统、随机点名等场景

### <font color=#1FA774>有序集合</font>

有序集合和集合类似，不同点在于每个元素都有一个分值属性，集合内的元素根据分值排序，还可以根据分值范围获取元素的列表

#### <font color=#9933FF>常用命令</font>

这里就不一一介绍每种命令的用法，可以去 **[SortedSet（有序集合）](http://doc.redisfans.com/sorted_set/index.html)**查看详细介绍！！但是这里给出常用命令的时间复杂度：

|                           命令                            | 时间复杂度  |                     命令                      |     时间复杂度      |
| :-------------------------------------------------------: | :---------: | :-------------------------------------------: | :-----------------: |
| zadd key score member [[score member] [score member] ...] | O(m * logn) |     zrevrange key start stop [withscores]     |     O(logn + m)     |
|                         zcard key                         |    O(1)     |    zrangebyscore key min max [withscores]     |     O(logn + m)     |
|                     zscore key member                     |    O(1)     |   zrevrangebyscore key min max [withscores]   |     O(logn + m)     |
|                     zrank key member                      |   O(logn)   |              zcount key min max               |     O(logn + m)     |
|                    zrevrank key member                    |   O(logn)   |        zremrangebyrank key start stop         |     O(logn + m)     |
|               zrem key member [member ...]                | O(m * logn) |         zremrangebyscore key min max          |     O(logn + m)     |
|               zincrby key increment member                |   O(logn)   | zinterstore destination numkeys key [key ...] | O(nk) + O(m * logm) |
|            zrange key start stop [withscores]             | O(logn + m) | zunionstore destination numkeys key [key ...] | O(n) + O(m * logm)  |

#### <font color=#9933FF>内部编码</font>

有序集合类型有两种编码

- **ziplist：**当有序集合中的元素个数小于设置值 (默认 128)，同时元素长度都小于设置值 (默认 64 byte)，Redis 会使用 ziplist 作为有序集合的内部编码
- **skiplist：**当有序集合类型无法满足 ziplist 的要求时，会选择 skiplist 作为内部编码

#### <font color=#9933FF>应用场景</font>

**需要随机获取数据源中根据分值排序的场景：**各种排行榜，如：直播送礼排行榜、微信步数排行榜等场景

**存储的任务有优先级或重要程度的场景：**优先级任务队列

### <font color=#1FA774>Bitmap</font>

从这里开始，下面内容主要介绍三个特殊的数据类型！！

Bitmap 本身不是一种数据结构，实际上它就是字符串，但是它可以对字符串的位进行操作。Bitmap 基本操作演示：

```bash
# setbit key offset value 设置指定偏移量 offset 的值为 value
SETBIT myKey 7 1
(integer) 0
# getbit key offset 获取指定偏移量 offset 的值
getbit myKey 7
(integer) 1
# bitcount key [start end] 获取指定范围内 1 的个数
bitcount myKey 0 7
(integer) 1
# bitop op destkey key [key ...] 对一个或多个 Bitmap 做 op 运算，结果存储到 destkey 中
bitop and myKey myKey newKey
```

#### <font color=#9933FF>应用场景</font>

**需要保存状态信息 (0/1 即可表示) 的场景：**用户签到情况、活跃用户情况、用户行为统计

### <font color=#1FA774>HyperLogLog</font>

HyperLogLog 并不是一种数据结构，实际类型是字符串类型。它是一种基数算法，通过 HyperLogLog 可以利用极小的内存空间完成独立总数的统计，大概只需要 12k 就能存储 $2^{64}$ 个不同元素

Redis 对 HyperLogLog 的存储结构进行了优化，采用两种方式计数：

- **稀疏矩阵：**计数较少时，占用空间很小
- **稠密矩阵：**计数达到某个阈值时，占用 12k 空间

基数计数概率算法为了节省内存并不会直接存储元数据，而是通过一定概率统计的方法预估基数值 (集合中包含元素的个数)

因此，HyperLogLog 的计数结果并不是一个准确值，存在一定误差，标准误差为 0.81%。开发者在考虑使用 HyperLogLog 时，需要考虑如下两点：

- 只为了计数，不需要获取单条数据
- 可以容忍一定的误差率

HyperLogLog 基本操作演示：

```bash
# pfadd key element [element ...] 添加一个或多个元素到 HyperLogLog 中
pfadd test 1 2 3 3
(integer) 1
# pfcount key [key ...] 获取一个或多个 HyperLogLog 的唯一计数
pfcount test
(integer) 3
# pfmerge destkey sourcekey [sourcekey ...] 将多个 HyperLogLog 合并到 destkey 中
pfmerge hll test
OK
PFCOUNT test
(integer) 3
PFCOUNT hll
(integer) 3
```

**<font color='red'>注意：</font>**如果把重复的元素加入 HyperLogLog 中不会重复计数

#### <font color=#9933FF>应用场景</font>

**数量巨大的计数场景：**热门网站每日/每周/每月访问 IP 数量统计、热门帖子 UV 统计

### <font color=#1FA774>GEO</font>

Geospatial index (地理空间索引，GEO) 主要用于存储地理位置信息，基于 SortedSet 实现

通过 GEO 可以轻松实现两个位置距离的计算、获取指定位置附近的元素等功能，如：微信附近的人、摇一摇

GEO 常用命令如下：

```bash
# longitude 经度; latitude 纬度; member 成员
# 将经纬度和成员添加到对应的 key 中，如果已经存在该成员返回 0，否则返回 1
geoadd key longitude latitude member [longitude latitude member ...]
# 返回给定成员的经纬度信息
geopos key member [member ...]
# 返回两个成员之间的距离
geodist key member1 member2
# 获取给定经纬度方圆 radius 距离的成员，M|KM|FT|MI 是单位
georadius key longitude latitude radius M|KM|FT|MI
# 获取给定成员方圆 radius 距离的成员
georadiusbymember key member radius M|KM|FT|MI
```

#### <font color=#9933FF>应用场景</font>

**需要使用地理空间数据的场景：**附近的人等场景