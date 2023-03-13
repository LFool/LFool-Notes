# IO 模型

### <font color=#1FA774>IO 是什么？</font>

从字面意思上来看，它就是输入/输出的意思，但从计算机和应用程序的角度它有不一样的含义

根据冯.诺依曼结构，计算机结构分为五部分：运算器、控制器、存储器、输入设备、输出设备。如下图所示：

![11](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/1524381678692278zVkfI311.svg)

输入设备向计算机输入数据，如：鼠标、键盘等；输出设备接收计算机输出数据，如：显示器等

从计算机结构的角度来看：**<font color='red'>IO 描述了计算机系统和外部设备之间通信的过程</font>**

操作系统为了稳定性和安全性，将进程的地址空间分为：**用户空间**和**内核空间**。平时运行的程序都是在用户空间，而一些和系统资源相关的操作都属于内核空间，由操作系统统一管理

应用程序不能直接访问内核空间，当要执行 IO 操作时，只能发起系统调用交由操作系统帮助完成。每次系统调用都会从用户态切换成内核态，当系统调用完成后，再切换回来，需要一定的开销

平时开发程序中接触最多的就是：**磁盘 IO (读写文件)** 和**网络 IO (网络请求和响应)**

从应用程序的角度来看：**<font color='red'>应用程序对系统内核发起 IO 调用 (系统调用)，操作系统负责执行这些系统调用。也就是应用程序只是发起了 IO 调用的请求，具体的 IO 执行是由操作系统内核执行</font>**

应用程序一次完整的 IO 调用包括：

- **数据准备阶段：**内核等待 IO 设备准备好数据
- **数据拷贝阶段：**将数据从内核缓冲区拷贝到用户缓冲区

对于准备阶段，关于读请求的具体含义为：**等待系统调用的完整请求数据，并将外围设备的数据读入内核缓冲区**；关于写请求的具体含义为：**等待系统调用的完成请求数据，并将用户缓冲区数据写入内核缓冲区**

![2](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/1606431678694803njJhiy2.svg)

应用程序在发起 IO 调用至内核执行 IO 返回之前，应用程序进程/线程所处状态，就是下面一部分内容

### <font color=#1FA774>UNIX 系统中的 IO 模型</font>

写在前面，关于「请求」的同步/异步

- **同步：**发起请求后，必须等待结果返回
- **异步：**发起请求后，不用等待结果返回，等结果返回后会通过回调通知调用者

关于阻塞/非阻塞

- **阻塞：**发起请求后，调用者一直等待结果的返回，当前线程会被挂起，无法干其它事情，只有当条件就绪后才可以继续
- **非阻塞：**发起请求后，调用者不用一直等待结果的返回，可以去干其它事情

先分享一个很有意思的小故事！！！从前有个人叫老张，他喜欢喝开水，一言不合就煮开水喝

**情况一：**老张把水壶放到火上，站在旁边啥也不干等着水烧开 -> **<font color='red'>同步阻塞</font>**

**情况二：**老张把水壶放到火上，不站在旁边干等着，但会时不时来看水有么有烧开 -> **<font color='red'>同步非阻塞</font>**

**情况三：**老张买了个新的水壶，水开了就会叫，老张把水壶放到火上，站在旁边啥也不干等着水烧开 -> **<font color='red'>异步阻塞</font>**

**情况四：**老张买了个新的水壶，水开了就会叫，老张把水壶放到火上，就可以去干其它事情，也不用时不时来看，当水壶叫了就知道水开了 -> **<font color='red'>异步非阻塞</font>**

**<font color='red'>注意：</font>**「异步」和「阻塞」放一起有些相互矛盾，异步就已经代表非阻塞了，所以基本没有这种说法

说了这么多，下面不说废话了，直接开门见山！！

在 UNIX 系统下，常见的 IO 模型有五种：**阻塞 IO (BIO)**、**非阻塞 IO (NIO)**、**IO 多路复用**、**信号驱动 IO**、**异步 IO**

#### <font color=#9933FF>BIO</font>

下图中的`recvfrom`函数视为系统调用 (下同)。进程调用`recvfrom`，直到数据报准备好且被复制到应用进程的缓冲区中或者发生错误才返回。用户进程从发起系统调用到结果返回均处于阻塞状态

**<font color='red'>缺点：</font>**如果使用多线程来提高效率，每个线程会对应一个 socket，对于长连接会造成大量的资源占用，可能后续来了更多连接后造成性能上的瓶颈

![3](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/1709101678698550A8dpcY3.svg)

#### <font color=#9933FF>NIO</font>

进程发起 IO 调用，无论结果如何都会直接返回，如果返回值是一个错误`EWOULDBLOCK`，表示数据报还未准备好。进程将继续不断轮询，直到数据报准备好以及完成复制

下图中前三次 IO 调用都返回了错误，直到第四次数据报才准备好，然后完成了数据复制过程

**<font color='red'>缺点：</font>**进程在完成 IO 调用前会不断轮询操作是否就绪，会消耗大量 CPU 时间

![4](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/1856531678705013ErXAKZ4.svg)



#### <font color=#9933FF>IO 多路复用</font>

应用进程首先会系统调用`select`，然后应用进程进入阻塞状态

`select`会不断轮询所负责的所有 socket，当某个 socket 数据报准备好了，就会通知应用进程。应用进程会再系统调用`recvfrom`完成数据报的复制工作

**<font color='red'>IO 多路复用的特点：</font>**在单线程里同时监控多个 socket，而这些 socket 任一个进入读就绪状态，`select`函数就可以返回

![5](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/2048271678711707lCocJa5.svg)

#### <font color=#9933FF>信号驱动 IO</font>

应用进程通过`sigaction`系统调用安装一个信号处理函数，该系统调用将立即返回，应用进程继续工作，并没有阻塞

当数据报准备好读取时，内核就为该进程产生一个 SIGIO 信号。我们既可以在信号处理函数中调用`recvfrom`读取数据报，并通知主循环数据已经准备好待处理，也可以立即通知主循环，让它读取数据报

**<font color='red'>优点：</font>**等待数据报准备好期间应用进程不被阻塞，主循环可以继续执行，只要等待来自信号处理函数的通知

![6](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/2142531678714973OHwxPi6.svg)

#### <font color=#9933FF>异步 IO</font>

**<font color='red'>工作机制：</font>**告知内核启动某个操作，并让内核在整个操作 (包括将数据从内核复制到应用缓冲区中) 完成后通知应用进程

**<font color='red'>和信号驱动 IO 的区别：</font>**信号驱动 IO 是由内核通知应用进程何时可以启动一个 IO 操作，而异步 IO 是由内核通知应用进程 IO 操作何时完成

![7](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/2155341678715734ZvHSBD7.svg)

#### <font color=#9933FF>五种模型比较</font>

![8](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/2212331678716753VF2WJO8.svg)

### <font color=#1FA774>Java 中三种常见的 IO 模型</font>

### <font color=#9933FF>BIO</font>

当应用程序发起 read 调用后，会一直阻塞，直到内核把数据拷贝到用户空间

**<font color='red'>适用场景：</font>**连接数目比较小且固定的架构，对服务器资源要求较高，但程序简单

```java
// 服务端
public class SocketServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8888);
        while (true) {
            System.out.println("等待连接 ...");
            // 阻塞方法
            Socket socket = serverSocket.accept();
            System.out.println("有客户端连接 ...");
            // 多线程处理
            new Thread(() -> {
                try {
                    handler(socket);
                } catch (IOException | InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();

        }
    }
    private static void handler(Socket socket) throws IOException, InterruptedException {
        System.out.println("Thread id = " + Thread.currentThread().getId());
        byte[] bytes = new byte[1024];
        System.out.println("准备 read ...");
        // 接受客户端的数据，阻塞方法，没有数据可读时就阻塞
        int read = socket.getInputStream().read(bytes);
        System.out.println("read 完毕 ...");
        if (read != -1) {
            System.out.println("接收客户端的数据：" + new String(bytes, 0, read));
            System.out.println("Thread id = " + Thread.currentThread().getId());
        }
        // 当服务端未准备好数据时，客户端一直处于阻塞等待
        Scanner in = new Scanner(System.in);
        String info = in.next();
        socket.getOutputStream().write(info.getBytes());
        socket.getOutputStream().flush();
    }
}
// 客户端
public class SocketClient {
    public static void main(String[] args) throws IOException, InterruptedException {
        Socket socket = new Socket("127.0.0.1", 8888);
        // 向服务端发送数据
        socket.getOutputStream().write("Hello Serve".getBytes());
        socket.getOutputStream().flush();
        System.out.println("向服务端发送数据结束");
        byte[] bytes = new byte[1024];
        // 接收服务端回传的数据
        int read = socket.getInputStream().read(bytes);
        if (read != -1) {
            System.out.println("接收服务端的数据：" + new String(bytes, 0, read));
        }
        socket.close();
    }
}
```

### <font color=#9933FF>NIO</font>

服务器实现模式为一个线程可以处理多个请求 (连接)，客户端发送的请求都会注册到**多路复用器 selector** 中，多路复用器轮询到连接有 IO 请求就进行处理

BIO 是一旦有客户端连接就在服务端为它分配一个线程，而 NIO 是先将所有客户端连接注册到 selector 中，当 selector 中的连接有 IO 请求时才去为它分配线程

![9](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/2347101678722430e5vw3D9.svg)

- channel 类似于流，每个 channel 对应一个 buffer 缓冲区，buffer 底层是数组

- channel 会注册到 selector 上，由 selector 根据 channel 读写事件的发生将其交由某个空闲的线程处理

- selector 可以对应一个或者多个线程

- NIO 的 buffer 和 channel 既可以读也可以写

NIO 中所有 IO 都是从 channel 开始：

- 从通道进行读数据：创建一个缓冲区，然后请求通道读取数据 **(buffer <- channel)**
- 从通道开始写数据：创建一个缓冲区，填充数据，并要求通道写入数据 **(buffer -> channel)**

**<font color='red'>适用场景：</font>**高负载、高并发的 (网络) 应用

```java
// 服务端
public class NIOServer {
    public static void main(String[] args) throws IOException {
        // 创建一个在本地端口进行监听的服务 Socket 通道，并设置为非阻塞方式
        ServerSocketChannel ssc = ServerSocketChannel.open();
        // 必须配置为非阻塞才能往 selector 上注册，否则会报错，selector 本身就是非阻塞模式
        ssc.configureBlocking(false);
        ssc.socket().bind(new InetSocketAddress(8888));
        // 创建一个选择器 selector
        Selector selector = Selector.open();
        // 把 ssc 注册到 selector 中，并且 selector 对客户端 accept 连接操作感兴趣
        ssc.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
            System.out.println("等待事件发生 ...");
            // 轮询监听 channel 里的 key，selector 是阻塞的，accept() 也是阻塞的
            int select = selector.select();
            System.out.println("有事件发生了 ...");
            // 有客户端请求，被轮询监听到
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();
            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();
                handle(key);
            }
        }
    }
    private static void handle(SelectionKey key) throws IOException {
        if (key.isAcceptable()) {
            System.out.println("有客户端连接事件发生了 ...");
            ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
            // NIO 非阻塞的体现：accept 方法是阻塞的，但这里因为发生了连接事件，所以这个方法会马上执行，不会阻塞
            // 处理完连接请求不会继续等待客户端的数据发送
            SocketChannel sc = ssc.accept();
            sc.configureBlocking(false);
            // 通过 Selector 监听 Channel 时对读事件感兴趣
            sc.register(key.selector(), SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            System.out.println("有客户端数据可读事件发生了 ...");
            SocketChannel sc = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            // NIO 非阻塞的体现：read 方法不会阻塞
            int len = sc.read(buffer);
            if (len != -1) {
                System.out.println("读取到客户端发送的数据：" + new String(buffer.array(), 0, len));
            }
            ByteBuffer bufferToWrite = ByteBuffer.wrap("Hello Client".getBytes());
            sc.write(bufferToWrite);
            key.interestOps(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
        } else if (key.isWritable()) {
            System.out.println("有客户端数据可写事件发生了 ...");
            SocketChannel sc = (SocketChannel) key.channel();
            key.interestOps(SelectionKey.OP_READ);
        }
    }
}
// 客户端
public class NIOClient {
    private Selector selector;
    public static void main(String[] args) throws IOException {
        NIOClient client = new NIOClient();
        client.initClient("127.0.0.1", 8888);
        client.connect();
    }
    private void initClient(String ip, int port) throws IOException {
        // 获得一个 Socket 通道
        SocketChannel channel = SocketChannel.open();
        // 设置通道为非阻塞
        channel.configureBlocking(false);
        // 获得一个通道管理器
        this.selector = Selector.open();
        channel.connect(new InetSocketAddress(ip, port));
        channel.register(selector, SelectionKey.OP_CONNECT);
    }
    private void connect() throws IOException {
        while (true) {
            selector.select();
            Iterator<SelectionKey> it = this.selector.selectedKeys().iterator();
            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();
                if (key.isConnectable()) {
                    SocketChannel channel = (SocketChannel) key.channel();
                    if (channel.isConnectionPending()) {
                        channel.finishConnect();
                    }
                    channel.configureBlocking(false);
                    ByteBuffer buffer = ByteBuffer.wrap("Hello Server".getBytes());
                    channel.write(buffer);
                    channel.register(this.selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    read(key);
                }
            }
        }
    }
    private void read(SelectionKey key) throws IOException {
        SocketChannel channel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int len = channel.read(buffer);
        if (len != -1) {
            System.out.println("客户端收到消息：" + new String(buffer.array(), 0, len));
        }
    }
}
```

### <font color=#9933FF>AIO</font>

AIO 基于事件和回调机制实现的，应用程序 IO 调用后会立马返回，不会堵塞，当后台处理完成，操作系统会通知线程继续后续的操作

**<font color='red'>适用场景：</font>**连接数目多且连接比较长 (重操作) 的架构

**<font color='red'>注意：</font>**目前 AIO 的应用不广泛，因为性能并没有很大提高

### <font color=#1FA774>参考文章</font>

- **[如何完成一次 IO](https://llc687.top/126.html)**
- **[Java IO模型详解](https://javaguide.cn/java/io/io-model.html)**
- **[IO 模型知多少 | 理论篇 ](https://www.cnblogs.com/sheng-jie/p/how-much-you-know-about-io-models.html)**
- **[I/O 多路复用底层原理前篇 - 五种IO模型](https://mp.weixin.qq.com/s/T-hP3wt4whtvVh1H1LBU3w)**
- **[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)**