# HTTP 队头阻塞问题

### <font color=#1FA774>什么是队头阻塞？</font>

抛开 HTTP 协议来说，队头阻塞可以定义为：**在有序的处理的场景下，前面某个很慢的对象阻塞了后面对象的执行**

举个简单的例子，对于先来先服务的进程调度算法来说，如果前面某个进程执行的时间特别长，会直接导致后面进程长时间得不到 CPU 调度，如下图所示：

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230816/0334501692128090UDtTIF1.svg)

一种更好的进程调度策略是时间片轮询，为每个线程分配大小相等的时间片，这样从宏观角度上来看所有线程都在执行，没有发生阻塞，如下图所示：

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230816/03385216921283322KHX3x2.svg)

### <font color=#1FA774>HTTP/1.1 队头阻塞</font>

对于 HTTP/1.1 来说，必须按顺序将请求和响应的数据发送出去，如下图所示：

![4](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230816/0403161692129796kY8yZJ4.svg)

如上图所示，如果 TCP Packet 1 中的数据特别大，而 TCP Packet 2 和 TCP Packet 3 中的数据特别小，那么也必须等待 TCP Packet 1 中的数据下载完成

如果 TCP Packet 2 和 TCP Packet 3 比 TCP Packet 1 早到接收方，按照最佳理论来说应用层应该可以使用并渲染 HTTP Packet 2，但事实上不行，必须等待 HTTP Packet 1 处理完

这就类似于执行时间很长的进程阻塞了后面的进程，那能不能学时间片轮询一样，将应用层数据交叉封装到 TCP 中，那么从宏观上来看 HTTP Packet 1 和 HTTP Packet 2 可以同时处理，但事实上不行

![5](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230816/0425061692131106k5IQdQ5.svg)

如上图所示，TCP Packet 1 封装部分 HTTP Packet 1 的数据，TCP Packet 2 封装部分 HTTP Packet 2 的数据，TCP Packet 3 封装剩余 HTTP Packet 1 和 HTTP Packet 2 的数据

封装完成后，按照 TCP Packet 1、TCP Packet 2、TCP Packet 3 的顺序发送出去，但对于接受方来说无法识别完整的 HTTP 报文，很可能出现 HTTP 报文混乱的情况

回到队头阻塞问题上，对于 HTTP 来说，队头阻塞可以细分为两种情况：

- **请求的队头阻塞：**对于客户端发送请求到服务端，如果前一个请求的响应没有返回，那么当前请求就不允许发送，阻塞等待
- **响应的队头阻塞：**对于服务端接收来自客户端的请求，必须按照顺序处理并响应，如果前一个请求还没有处理完毕，那么当前请求就不允许处理，阻塞等待

在 HTTP/1.0 时，这两种情况都存在，因为它使用短连接，每一次「请求-响应」都需要重新建立连接和断开连接，相当于每一次请求都需要等待收到上一次请求的响应后才能发出

在 HTTP/1.1 时，使用长连接，且支持管道网络传输，可以发送多次请求而不需要等待响应，也支持累积确认；但对于接收方来说，必须按顺序处理并响应请求。如下图所示：

![5](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230527/0807021685146022JLQvHF5.svg)

**<font color='red'>所以，在 HTTP/1.1 时，解决了请求的队头阻塞，但没有解决响应的队头阻塞！！</font>**

对于管道网络传输来说，HTTP/1.1 默认并没有开启管道网络传输技术，而且浏览器基本都没有支持该功能！！

假设一个客户端开启了两个 TCP 连接，发送了三次请求 (A、B、C)，其中请求 A 使用了第一个 TCP 连接，而请求 C 使用第二个 TCP 连接，且 A 请求的文件特别大，B、C 请求的文件比较小

如果请求 B 使用了第二个 TCP 连接，其实问题不到，但如果请求 B 使用了第一个 TCP 连接，那么将会造成较长时间的阻塞。所以基于类似的场景，使用管道网络传输还会降低整体的响应时间！！

### <font color=#1FA774>HTTP/2.0 队头阻塞</font>

在 HTTP/1.1 中，之所以无法将多个 HTTP 报文交替的封装到一个 TCP 报文中，是因为无法区分哪一个块属于哪个 HTTP 报文

HTTP/2.0 在每个块之前添加了一个帧，它有两个作用：

- 表明后续块的大小
- 表明后续的块属于报文头 (Header) 还是报文体 (Data)

对于同一个 HTTP 报文的不同块，它们前面帧的 stream id 相同，通过该 id 可以判断属于同一个 HTTP 报文的块，如下图所示：

![6](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230816/0547161692136036nxNDkR6.svg)

通过添加帧，发送方和接收方都可以不按照顺序的发送数据和处理响应数据，也就是并发传输，也可被称为 I/O 多路复用，即：一个 TCP 中可以同时发送多个请求或响应

从这个角度来看，**<font color='red'>HTTP/2.0 既解决了请求的队头阻塞，也解决了响应的队头阻塞！！</font>**但仅限于 HTTP 层面，对于 TCP 层来说，依旧存在队头阻塞问题

HTTP/2.0 底层使用 TCP 协议，为了实现可靠传输，采用序列号和确认应答机制，只有当接收方成功接收到序列号 n 之前的所有数据后，才会将对应的数据递交给应用层

也就是当缺失了序列号 n 之前某个区间的数据时，那么该区间之后所有成功接收到的数据都只能暂存在 TCP 缓冲区中，如下图所示：

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230816/0352211692129141WF0Px53.svg)

对于上面的图，如果 TCP Packet 1 丢失，那么 TCP Packet 2 和 TCP Packet 3 并不会递交给应用层，而是继续留在 TCP 缓冲区中

按照正常思路来说，TCP 可以将 TCP Packet 2 和 TCP Packet 3 数据递交给应用层，因为其中包含 HTTP Packet 2 的完整数据，可以让应用层先处理 HTTP Packet 2

但由于 TCP 面向流传输，也就是 TCP 对应用层数据一无所知，即不知道数据类型，也不知道数据边界，更无法识别每个块前面的帧信息

**<font color='red'>所以，HTTP/2.0 仅解决了 HTTP 层面的队头阻塞问题，但依旧存在 TCP 层面的队头阻塞！！</font>**

### <font color=#1FA774>HTTP/3.0 队头阻塞</font>

由于 HTTP/2.0 依旧存在 TCP 层面的队头阻塞问题，所以 HTTP/3.0 底层没有使用 TCP 协议，而是使用 QUIC 协议

QUIC 协议不仅具有 TCP 协议的所有功能，如：可靠传输、流量控制、拥塞控制等，而且还拥有一些其它特性

可以将 QUIC 协议理解为把应用层的帧信息下移到传输层，也就是传输层也可以通过帧信息识别应用层的一些数据，从而避免队头阻塞问题，如下图所示：

![7](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230816/0614441692137684JsM2ne7.svg)

如果 QUIC Packet 2 丢失，那么 QUIC 协议可以通过流帧 (Stream) 判断出 QUIC Packet 1 和 QUIC Packet 3 中`stream id = 1`的数据没有间隙，可以递交给应用层处理

### <font color=#1FA774>总结</font>

HTTP/1.1 虽然通过管道解决了请求的队头阻塞，但没有解决响应的队头阻塞，而且一般也不开启管道网络传输技术

HTTP/2.0 在 HTTP 数据块前面加帧，表明块的类型和大小，实现并发传输，即：一个 TCP 中可以发送多个请求或响应，但只解决 HTTP 层面的队头阻塞，一旦发生丢包，TCP 层面还是会出现队头阻塞

HTTP/3.0 没有使用 TCP 协议，而是改用 QUIC 协议，将 HTTP 数据块前面加的帧信息下推到传输层，QUIC 协议可以根据流帧 (Stream) 避免队头阻塞

### <font color=#1FA774>参考文章</font>

- **[关于队头阻塞（Head-of-Line blocking），看这一篇就足够了](https://zhuanlan.zhihu.com/p/330300133)**