# SYN 泛洪攻击

### <font color=#1FA774>半连接 & 全连接</font>

在 TCP 三次握手的时候，Linux 内核会维护两个队列：

- 半连接队列，也称 SYN 队列，只进行了一次握手的连接
- 全连接队列，也称 Accept 队列，进行了三次握手的连接

![9](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230529/1125171685330717iGBujO9.svg)

关于出队和入队的详细流程如下：

- 当服务器收到客户端的 SYN 报文时，会创建一个半连接对象，并加入到内核的 SYN 队列中
- 接着服务器发送 SYN-ACK 报文给客户端，等待客户端的 ACK 报文
- 当服务器收到客户端的 ACK 报文时，会取出一个半连接对象，然后创建一个新的连接对象加入 Accept 队列中
- 应用程序通过调用 Socket 提供的`accept()`方法，从 Accept 队列中取出一个连接对象

**<font color='red'>注意：</font>**不管是半连接队列还是全连接队列，都有最大长度限制，超过限制时都会丢弃报文！！

### <font color=#1FA774>SYN 泛洪攻击是什么</font>

客户端和服务器使用 TCP 协议通信前需要进行三次握手建立连接，当服务器收到客户端第一次握手的 SYN 报文后，会发送 SYN-ACK 报文，之后服务器处于 SYN_RECV 状态。**更详细可见 [三次握手](./TCP-三次握手-四次挥手.html#tcp-三次握手过程)**

**SYN 泛洪攻击：**攻击者会伪造大量不同 IP 的 SYN 报文，服务器收到 SYN 报文后会向客户端发送 SYN-ACK 报文，但攻击者不会回应该报文，这样会逐渐占满半连接队列，使得后续正常连接请求直接被丢弃

### <font color=#1FA774>如何避免 SYN 泛洪攻击</font>

#### <font color=#9933FF>方法一：调大 netdev_max_backlog</font>

当网卡接收数据的速度大于内核处理的速度时，会有一个队列保存这些数据包。控制该队列的大小，默认值为 1000，可以通过参数调整为 100000

```bash
net.core.netdev_max_backlog = 10000
```
#### <font color=#9933FF>方法二：增加 TCP 半连接队列大小</font>

增大 TCP 半连接队列，要同时增大下面这三个参数：

- 增大 net.ipv4.tcp_max_syn_backlog
- 增大 listen() 函数中的 backlog
- 增大 net.core.somaxconn

#### <font color=#9933FF>方法三：开启 net.ipv4.tcp_syncookies</font>

开启 net.ipv4.tcp_syncookies 后可以不需要加入到 SYN 半连接队列中，直接成功建立连接

#### <font color=#9933FF>方法四：减少 SYN + ACK 重发次数</font>

攻击者不会回应服务器的 SYN-ACK 包，服务器会超时重发，在 Linux 默认重发 5 次，可以适当降低重发的次数使处于 SYN_RECV 状态的服务器尽快断开连接