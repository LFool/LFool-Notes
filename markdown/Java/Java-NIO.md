# Java NIO

在 Java 中，一共有三种 I/O 模型：BIO、NIO、AIO，**关于它们的简述可见 [Java 中三种常见的 I/O 模型](./IO模型.html#java-中三种常见的-io-模型)**。本篇文章主要介绍 Java NIO 的相关内容～～

### <font color=#1FA774>Java NIO 简述</font>

本着完整的原则，在文章开头简单介绍一下 Java NIO 的核心思想～～

Java NIO 是非阻塞的 **[I/O 多路复用](../os/IO多路复用.html)** 模型，服务端实现了一个线程管理多个客户端连接，这些连接都会注册到多路复用器 Selector 中，Selector 会轮询所管理的连接，当连接有事件发生就去处理它

在 Java BIO 中，一个线程负责一个连接，当连接没有可读可写请求时线程会阻塞，十分浪费线程资源，而 Java NIO 中一个线程管理多个连接，只要有一个连接有 I/O 事件发生就可以去处理它

Java NIO 有三大核心组件：**Buffer (缓冲区)**、**Channel (通道)**、**Selector (多路复用选择器)**，如下图所示：

![9](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230313/2347101678722430e5vw3D9.svg)

Java NIO 中所有的 I/O 请求都是从 Channel 开始：
- **从通道读数据 (服务端向客户端发送数据)：**从通道读取数据到缓冲区 **(Buffer <- Channel)**
- **向通道写数据 (客户端向服务端发送数据)：**从缓冲区写入数据到通道 **(Buffer -> Channel)**

上面三大核心组件有如下关系：

- Channel 类似于流 (但有区别！！)，每个 Channel 对应一个 Buffer 缓冲区，Buffer 底层是数组
- Channel 会注册到 Selector 上，由 Selector 根据 Channel 读写事件的发生将其交由某个空闲的线程处理
- Selector 可以对应一个或者多个线程，如果是多个线程就相当于有多个 I/O 多路复用
- Buffer 和 Channel 既可以读也可以写

Java BIO 和 Java NIO 的区别：

- BIO 是阻塞的；NIO 是非阻塞的
- BIO 基于字节流和字符流处理数据，而 NIO 基于缓冲区处理数据
- BIO 是单向传输，实现双向传输需要创建两个 Stream (InputStream 和 OutputStream)；NIO 是双向传输，Buffer 和 Channel 既可以读也可以写

### <font color=#1FA774>Buffer</font>

下面介绍 Java NIO 中第一个核心组件 Bufer (缓冲区)，它底层是一个数组。Java 中`Buffer`是一个抽象类，有七个子类，除 boolean 类型的其它基本数据类型都对应一个`Buffer`的子类

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230622/03482116873769011ZS50Y2.svg)

在`Buffer`的每个子类中，都有一个对应类型的数组，这也是说 Buffer 缓冲区底层是一个数组的原因，以`IntBuffer`为例：

```java
final int[] hb;
```

下面主要介绍为什么 Buffer 既可以读也可以写，这是如何实现的！！在`Buffer`类中，有四个属性：

```java
private int mark = -1;     // 用于临时保存 position 的值
private int position = 0;  // 下一个读或写数据的下标，position < limit
private int limit;         // 对缓冲区可读可写的极限位置，limit = 5 表示 [0, 5) 区间内数据可读可写
private int capacity;      // 缓冲区容量，也就是 hb 数组的大小
```

与此同时，还有一个`flip()`方法用于读写切换，它会改变上面四个属性：

```java
public Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

下面模拟一下 Buffer 读、写、读写切换的过程。假设起初有一个容量为 5 的 Buffer，那么 position = 0，limit = capacity = 5，如下图所示：

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230622/0408481687378128HVDtCb3.svg)

如果向 Buffer 中添加了三个数`0, 1, 2`，此时 position = 3。假设现在开始读数据，先调用`flip()`方法，那么 position = 0，limit = 3，capacity = 5，如下图所示：

![4](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230622/0421371687378897TMPqiI4.svg)

从上面例子可知，当为写模式时，只能在`[0, limit)`区间内写数据；当为读模式时，只能在`[0, limit)`区间内读数据

下面给出上面例子对应的代码实现：

```java
public class BasicBuffer {
    public static void main(String[] args) {
        // 定义一个容量为 5 的 IntBuffer
        IntBuffer intBuffer = IntBuffer.allocate(5);
        // 向 IntBuffer 中写数据
        for (int i = 0; i < 3; i++) intBuffer.put(i);
        // Buffer 读写切换
        intBuffer.flip();
        // 从 IntBuffer 中读数据
        while (intBuffer.hasRemaining()) {
            System.out.print(intBuffer.get() + " ");
        }
    }
}
```

### <font color=#1FA774>Channel</font>

下面介绍 Java NIO 中第二个核心组件 Channel (通道)，它是双向的，既可以从里面读数据，也可以往里面写数据，而且所有的数据交互都需要用到 Buffer 和 Channel

假设客户端向服务器发送一条消息，那么整个流程为：

- 客户端有一个 Buffer 和 Channel，先将数据写入 Buffer 中，然后调用`write()`将 Buffer 中数据写入 Channel 中
- 服务器也有一个 Buffer 和 Channel，客户端和服务器的 Channel 可认为是连通的，此时服务器的 Channel 中已经有客户端写入的数据，然后调用`read()`将 Channel 中数据读到 Buffer 中

可能描述比较抽象，直接上图：

![5](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230624/0305481687547148AV81mu5.svg)

从上图可以看出两个方法；

- `Channel.write(ByteBuffer)`：将 Buffer 中数据写入到 Channel 中
- `Channel.read(ByteBuffer)`：将 Channel 中数据读取到 Buffer 中

**<font color='red'>技巧：</font>**这两个方法一开始很难分清到底是往谁写、往谁读，其实只要把 Channel 作为动作的发出者即可，`write()`就是往 Channel 中写，`read()`就是从 Channel 中读，而交互的目标都是 Buffer

介绍了 Channel 在 Java NIO 中的作用后，下面来看看到底有哪几种具体的 Channel：

![6](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230624/0319371687547977N1AcM36.svg)

- **FileChannel：**文件通道，用于文件的读写
- **DatagramChannel：**用于 UDP 连接的接收和发送
- **SocketChannel：**用于 TCP 连接的接收和发送，但面向于客户端
- **ServerSocketChannel：**用于 TCP 连接的接收和发送，但面向于服务器

上面介绍过，客户端和服务器两边其实都有 Channel，ServerSocketChannel 面向于服务器，主要负责处理连接请求，当有一个新连接时，ServerSocketChannel 会监听到该请求，然后创建一个 SocketChannel 负责后续和该连接进行读写操作，文末给出的 Demo 会更加清楚阐述这一区别

### <font color=#1FA774>Selector</font>

下面介绍 Java NIO 中第三个核心组件 Selector (多路复用器)，它的作用和 **[I/O 多路复用](../os/IO多路复用.html)** 中的`select()`相同，一个线程可以管理多个 Channel (**注意：**Channel 和连接等价)

下面简述一下 Selector 的原理：

- 服务器创建一个 ServerSocketChannel 并加入到 Selector 中，负责监听连接请求
- 有新客户端连接后，会创建一个对应的 SocketChannel 并加入到 Selector 中，负责监听读写请求
- Selector 调用`select()`方法，当 Selector 中管理的 Channel 有事件发生时 (连接、可读、可写事件)，从该方法中返回，然后遍历处理发生的事件

Selector 其实是底层是一个`Set<SelectionKey>`集合，每个 SelectionKey 都对应一个 Channel，可以通过 SelectionKey 获取对应的 Channel

### <font color=#1FA774>Demo</font>

下面给出一个 Java NIO 的 Demo ～

先给出服务端的代码：

```java
public class Server {
    public static void main(String[] args) throws IOException {
        // 创建一个 ServerSocketChannel，绑定端口号 8888，设置为非阻塞
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(new InetSocketAddress(8888));
        serverSocketChannel.configureBlocking(false);
        // 创建一个 Selector，将 ServerSocketChannel 注册到 Selector 中，并设置监听 OP_ACCEPT 事件 (新连接事件)
        Selector selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            // Selector 监听管理的 Channel，当 Channel 有事件发送就从 select() 返回，返回值表示有事件发送的数量
            int select = selector.select();
            // 获取有事件发生的集合，并遍历
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                handle(key);
            }
        }
    }
    private static void handle(SelectionKey key) throws IOException {
        if (key.isAcceptable()) {
            // 发生连接事件
            // 获取 ServerSocketChannel，并为新连接生成一个 SocketChannel，注册到 Selector 中，监听读事件
            System.out.println("有新的客户端连接！！");
            ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();
            SocketChannel socketChannel = serverSocketChannel.accept();
            socketChannel.configureBlocking(false);
            socketChannel.register(key.selector(), SelectionKey.OP_READ, ByteBuffer.allocate(1024));
        } else if (key.isReadable()) {
            // 发生可读事件
            // 获取可读事件对应的 SocketChannel 和 ByteBuffer，将 SocketChannel 中数据读到 ByteBuffer
            SocketChannel socketChannel = (SocketChannel) key.channel();
            ByteBuffer byteBuffer = (ByteBuffer) key.attachment();
            int read = socketChannel.read(byteBuffer);
            if (read != -1) {
                System.out.println("服务器收到客户端消息：" + new String(byteBuffer.array(), 0, read));
            }
            key.cancel();
            byteBuffer.clear();
        }
    }
}
```

再给出客户端的代码：

```java
public class Client {
    public static void main(String[] args) throws IOException, InterruptedException {
        // 创建一个 SocketChannel，设置为非阻塞
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        InetSocketAddress inetSocketAddress = new InetSocketAddress("127.0.0.1", 8888);
        if (!socketChannel.connect(inetSocketAddress)) {
            while (!socketChannel.finishConnect()) {
                System.out.println("连接需要一定时间，等待过程中可以处理其它事情 ...");
            }
        }
        // 连接成功
        ByteBuffer byteBuffer = ByteBuffer.wrap("Hello, I am Client".getBytes());
        socketChannel.write(byteBuffer);
        socketChannel.close();
    }
}
```

### <font color=#1FA774>参考文章</font>

- **[I/O 模型](./java/IO模型.html)**
- **[I/O 多路复用](../os/IO多路复用.html)**
- **[Java NIO：Buffer、Channel 和 Selector](https://www.javadoop.com/post/java-nio)**
