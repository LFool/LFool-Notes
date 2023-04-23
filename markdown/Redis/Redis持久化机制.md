# Redis 持久化机制

我们知道 Redis 之所以那么快，是因为它是内存数据库，将所有数据都存在内存中。相比于 MySQL 来说，Redis 减少了每次数据需要从磁盘读入到内存中的 IO 开销

可是一旦 Redis 服务重启，那么内存中的数据就会被清空，导致数据丢失。为了保留数据，Redis 支持三种持久化方案：

- **RDB (Redis Database)：**保存数据库中的键值对
- **AOF (Append Only File)：**保存对数据库修改的命令，
- **RDB 和 AOF 的混合持久化** (Redis 4.0 新增)

### <font color=#1FA774>RDB</font>

RDB 通过创建快照 (dump.rdb 文件) 来获取 Redis 数据库在某个时间点上的副本；当 Redis 重启时可以通过加载快照文件 (dump.rdb) 来恢复上次内存中的数据

有两个 Redis 命令可以生成 RDB 文件：

- `save`：阻塞服务器，直至 RDB 文件创建完毕为止，在服务器阻塞期间不可以处理任何命令请求
- `bgsave`：执行`fork()`操作创建子进程，由子进程负责创建 RDB 文件，服务器可以继续处理命令请求

Redis 默认使用 RDB 持久化机制，且默认选择`bgsave`命令。Redis 的配置文件`redis.conf`中如下配置：

```bash
save 900 1     # 在 900s 内，如果至少有 1 个 key 发生变化，Redis 就会自动触发 bgsave 命令创建快照
save 300 10    # 在 300s 内，如果至少有 10 个 key 发生变化，Redis 就会自动触发 bgsave 命令创建快照
save 60 10000  # 在 60s 内，如果至少有 10000 个 key 发生变化，Redis 就会自动触发 bgsave 命令创建快照
```

只要满足上述三个条件中的任意一个，`bgsave`命令就会被执行

这里先说明一下子进程和父进程的内存关系。假设父进程内存中有一个变量`a`，那么子进程内存中也应该有一个变量`a`

最开始父子进程都指向内存中同一个`a`，一旦有进程对`a`修改时，就会复制一个新的`a`，此时父子进程指向不再是同一个`a`，这就是写时复制技术 (copyOnWrite)

再衍生一下，如果各自输出`a`的内存地址会发现是一样的，这是因为输出的内存地址其实是相对于进程的偏移量，并非实际的物理内存地址。由于是父子进程，所以`a`在两个进程中的偏移量是一样的

这里给出一个程序：

```c
#include <stdio.h>
#include <unistd.h>
int main() {
    // 定义一个整型变量 a
    int a = 100;
    pid_t pid = fork();
    if(pid > 0) {
        // 父进程等 0.01s 让子进程先运行
        usleep(10000);
        a -= 20;
        printf("father &a : %p\n", &a);
        printf("father : a = %d\n", a);
    }
    if(pid == 0) {
        a -= 10;
        printf("child &a : %p\n", &a);
        printf("child : a = %d\n", a);
    }
    return 0;
}
// result
child &a : 0x16cf8f408
child : a = 90
father &a : 0x16cf8f408
father : a = 80
```

回归主题，下面给出`bgsave`命令执行的流程图：

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230417/0218001681669080XV0cz22.svg)

最后再来说一下 RDB 的优缺点。先说**优点**：

- RDB 是一个紧凑压缩的二进制文件，不仅适用于重启后的恢复，而且还可以用于备份或者复制
- RDB 恢复数据的速度比 AOF 快。RDB 是数据库的一个快照，而 AOF 记录的是修改命令，AOF 恢复需要 Redis 批量执行记录的命令

再来说**缺点**：

- RDB 方式无法做到实时持久化/秒级持久化，因为创建子进程属于重量级操作，频繁的执行成本高
- 版本不同的 Redis 会用不同的 RDB 文件格式，新旧版本 RDB 文件无法兼容

### <font color=#1FA774>AOF</font>

如果说 RDB 是保存某个时间点的数据库状态 (快照)，那么 AOF 就是保存导致数据库状态的过程，通过重新执行一遍过程使数据库到达目标状态

更官方一点：**AOF 持久化通过保存 Redis 服务器所执行的增删改命令来记录数据库状态，重新执行保存命令即可恢复数据**

与 RDB 相比，AOF 的实时性更好。假设某个时间段内仅有一条命令使数据库改变，那么 RDB 需要重新生成一份数据库快照，而 AOF 只需要记录该条修改命令即可

默认情况下 Redis 并没有开启 AOF，如果要开启，需要修改配置文件`appendonly yes`，默认文件名为`appendonly.aof`

AOF 的工作流程主要有四个步骤：命令写入 (append)、文件同步 (sync)、文件重写 (rewrite)、重启加载 (load)

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230417/15090916817153499DuvqH3.svg)

#### <font color=#9933FF>命令写入 (append)</font>

Redis 服务器可以解析 RESP 标准的命令，每次客户端发送命令到服务器也是将其封装成 RESP 格式，具体格式如下：(CRLF 表示`\r\n`)

```
*<参数数量> CRLF
$<参数 1 的字节数量> CRLF
<参数 1> CRLF
...
$<参数 n 的字节数量> CRLF
<参数 n> CRLF
```

对于`set hello world`命令，它在 RESP 标准下的格式为：

```
*3\r\n$3\r\nset\r\n$5\r\nhello\r\n$5\r\nworld\r\n
```

而 AOF 也是基于 RESP 标准存储命令

AOF 并没有直接把命令写入到文件中，而是先写到 AOF 缓冲区 aof_buf 中，至于缓冲区何时真正同步到文件中取决于采取的策略，这也是下个步骤要做的事情

#### <font color=#9933FF>文件同步 (sync)</font>

文件同步是将 AOF 缓冲区的内容同步到文件中，这一步才算真正的将内存数据持久化。Redis 提供了多种 AOF 缓冲区同步文件策略，由参数`appendfsync`控制，不同值的含义如下表：

| appendfsync 选项的值 |                             行为                             |
| :------------------: | :----------------------------------------------------------: |
|        always        | 主线程调用 write 把命令写入 aof_buf 缓冲区<br />后台线程 (aof_fsync 线程) 立即调用 fsync 同步到 AOF 文件 (刷盘)，fsync 完成后线程返回 |
|       everysec       | 主线程调用 write 把命令写入 aof_buf 缓冲区<br />后台线程每秒调用一次 fsync 同步到 AOF 文件 (刷盘) |
|          no          | 主线程调用 write 把命令写入 aof_buf 缓冲区<br />刷盘操作由操作系统决定，通常同步周期最长 30s |

关于系统调用`write`和`fsync`的说明：

- `write`写到系统内核缓冲区之后就直接返回，不会立即同步到磁盘。虽然提高了效率，但也带来了数据丢失的风险。同步磁盘操作通常依赖系统调度机制，Linux 内核通常为 30s 同步一次
- `fsync`用于强制磁盘同步，将阻塞直至写入到磁盘为止，保证了数据持久化。该方法由专门的后台线程`aof_fsync`调用

如果选择`always`同步策略，将牺牲效率换取安全性；如果选择`no`同步策略，将牺牲安全性换取效率；所以一般来说选择`everysec`同步策略较好

#### <font color=#9933FF>文件重写 (rewrite)</font>

随着 Redis 服务器的运行，AOF 文件只会越来越大，而且还可能会存在很多冗余命令，比如服务器执行了下面两条命令：

```bash
set hello world
del hello
```

由于这两条命令对数据库进行了修改，所以会将它们存入 AOF 文件中，待重启加载时执行。如果将这两条指令看作一个整体，相当于没有对数据库进行修改 (添加后删除)

所以文件重写就是新创建一个 AOF 文件，然后交由子进程去遍历数据库中的键值对，根据数据库状态生成最简执行命令，最后将新 AOF 文件替换旧 AOF

AOF 重写过程可以通过调用`bgrewriteaof`命令手动触发，也可以自动触发，满足以下两个条件时自动触发

```bash
# 如果 AOF 文件小于该值，则不触发，默认 64mb
auto-aof-rewrite-min-size 64mb
# 当前 AOF 文件比上一次重写后 AOF 文件至少大一倍 (默认 100%) 时触发
auto-aof-rewrite-percentage 100
```

由于文件重写是`fork()`出来的子进程去完成，而父进程依旧可以处理客户端命令，所以新创建出来的 AOF 文件和实际的数据库存在一个状态差

![4](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230417/2158451681739925AIC96e4.svg)

Redis 设置了一个 AOF 重写缓冲区，在文件重写操作触发后，将 Redis 服务器执行的命令同时追加到 AOF 缓冲区 (aof_buf) 和 AOF 重写缓冲区 (aof_rewrite_buf) 中

当子进程完成文件重写任务后，会向父进程发送一个信号，父进程接收到该信号后会进行两个操作：**(这两个操作会阻塞父进程，也就是 Redis 服务进程)**

- 将 aof_rewrite_buf 中的内容全部写入到新 AOF 文件中，这时新 AOF 文件状态和数据库当前状态一致
- 对新的 AOF 文件进行改名，原子地覆盖现有的 AOF 文件，完成新旧 AOF 文件的替换

下面给出文件重写完整的流程图：

![5](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230417/2308311681744111yusuxu5.svg)

这里再强调一遍：**父进程把重写期间的增删改命令缓存到 aof_rewrite_buf，等重写完成后由父进程追加到新 AOF 文件中，追加过程是阻塞滴！！**

如果重写期间父进程执行的命令较多，那么 aof_rewrite_buf 中会积攒大量的命令，而等子进程重写完成后会由父进程阻塞式将 aof_rewrite_buf 追加到新 AOF 中，会导致客户端的长时间得不到响应

为了改善上面的问题，Redis 通过在父子进程间建立管道，在子进程重写的后期阶段，父进程会将 aof_rewrite_buf 中积攒的命令通过管道发送给子进程，由子进程将这些数据追加到新 AOF 文件中

可能由于 aof_rewrite_buf 命令过多，导致子进程无法全部消费完，最后 aof_rewrite_buf 中剩余部分再由父进程阻塞式的追加到新 AOF 中。此时的 aof_rewrite_buf 相比于最初的小很多了

**利用管道优化 AOFRW 更详细分析可见 [Redis · 原理介绍 · 利用管道优化aofrewrite](http://mysql.taobao.org/monthly/2018/12/06/)**

你以为到这里就完了吗？？其实并没有，优化过后的重写操作依旧存在一些问题：

- **内存开销：**父进程会把同样的命令存两份 (aof_rewrite_buf 和 aof_buf)，几乎浪费了一半的内存；而且父子进程之间通过管道传输数据的开销也不小
- **磁盘 IO 开销：**aof_rewrite_buf 最终会写到新 AOF 文件，aof_buf 最终会写到原 AOF 文件，这两个缓冲区中数据绝大部分一样，导致同一份数据会产生两次磁盘 IO
- **CPU 开销：**命令写入缓冲区以及父子进程传输数据时都会占用一定的 CPU 时间

**阿里开发者团队在 Redis7.0 中发布了 Multi part AOF，详细可见 [Redis 7.0 Multi Part AOF的设计和实现](https://developer.aliyun.com/article/866957)**

从名字可以看出，Multi part AOF 就是将原来的单个 AOF 文件拆分成多个 AOF 文件。在 MP-AOF 中，将 AOF 分为三种类型：

- **BASE AOF：**基础 AOF，它一般由子进程通过重写产生，该文件最多只有一个
- **INCR AOF：**增量 AOF，它一般会在 AOFRW 开始执行时被创建，该文件可能存在多个
- **HISTORY AOF：**历史 AOF，它由 BASE 和 INCR AOF 变化而来，每次 AOFRW 成功完成时，本次 AOFRW 之前对应的 BASE 和 INCR AOF 都将变为 HISTORY，会被 Redis 自动删除

为了管理这些 AOF 文件，引入了一个 manifest (清单) 文件来跟踪、管理这些 AOF 文件

下面给出 MP-AOF 完整的流程图：

![6](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230417/23354016817457407rBafu6.svg)

从图中可以看出，在 AOFRW 期间不再需要 aof_rewrite_buf，省去了对应的内存消耗；父子进程之间也不需要传输数据和控制交互，省去了对应的 CPU 和磁盘 IO 开销

#### <font color=#9933FF>重启加载 (load)</font>

每次 Redis 服务启动时，它的加载流程如下：

![7](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230417/2349141681746554yT3oVr7.svg)

### <font color=#1FA774>RDB + AOF 混合持久化</font>

Redis 4.0 开始支持 RDB 和 AOF 混合持久化，默认关闭，可以通过配置`aof-use-rdb-preamble yes`开启

由于 AOF 文件重写时，子线程会遍历数据库，生成一个新 AOF 文件，而 RDB 正好是数据库的快照。所以不谋而合，直接将 RDB 写到 AOF 文件开头，形成`[RDB file][AOF tail]`

**好处：**快速加载避免丢失过多的数据

**缺点：**AOP 文件格式可读性差

### <font color=#1FA774>RDB 和 AOF 比较</font>

关于 RDB 和 AOF 的优缺点，官网也给了详细的说明 **[Redis persistence](https://redis.io/docs/management/persistence/)**

RDB 比 AOF 优秀的地方

- **RDB 文件小：**RDB 是经过压缩的二进制文件，文件小，适合做备份、灾难恢复；AOF 记录执行的命令，文件越来越大，但会在后台自动重写 AOF，Redis7.0 之前在重写期间可能会有使用大量内存空间
- **RDB 恢复快：**RDB 恢复直接解析还原数据即可；AOF 恢复时需要一条一条的执行命令

AOF 比 RDB 优秀的地方

- **AOF 实时/秒级持久化：**生成 RDB 文件过程比较繁重，需要遍历数据库；AOF 支持秒级数据丢失 (选择 everysec 策略)，仅仅是追加命令到 AOF 文件
- **RDB 文件格式不兼容：**版本不同的 Redis 会用不同的 RDB 文件格式，新旧版本 RDB 文件无法兼容 
- **AOF 文件格式便于理解：**AOF 基于 RESP 标准存储命令，可读性强

综上所述

- 如果 Redis 数据丢失一点也不要紧的话，可以选择 RDB
- 不建议单独使用 AOF，如果要求安全性高，可以同时开启 RDB 和 AOF，或者混合模式

### <font color=#1FA774>RDB 和 AOF 对过期键的处理</font>

**RDB 生成：**在执行`save`或`bgsave`命令创建一个新的 RDB 文件时，会对数据库中的键进行检查，已过期的键不会保存到新创建的 RDB 文件中

**RDB 载入：**如果是主服务器载入，会对键进程检查，不会载入已过期的键；如果是从服务器载入，无论是否过期，都会载入，因为主从同步时会清空从服务器

**AOF 写入：**如果数据库中某个键已经过期，但还没有被惰性删除或定期删除，那么对 AOF 文件不会有任何影响，当过期键被惰性删除或定期删除后，会向 AOF 中追加`del`命令

**AOF 重写：**在重写过程中，会对数据库的键进行检查，已过期的键不会被保存到重写后的 AOF 文件中

### <font color=#1FA774>参考文章</font>

- **Redis 设计与实现**
- **Redis 开发与运维**
- **[Redis持久化机制详解](https://javaguide.cn/database/redis/redis-persistence.html)**
- **[Redis · 原理介绍 · 利用管道优化aofrewrite](http://mysql.taobao.org/monthly/2018/12/06/)**
- **[Redis 7.0 Multi Part AOF的设计和实现](https://developer.aliyun.com/article/866957)**
- **[Redis persistence](https://redis.io/docs/management/persistence/)**
