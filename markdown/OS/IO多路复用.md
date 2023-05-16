# I/O 多路复用

### <font color=#1FA774>文件描述符</font>

在 Linux 中所有的 I/O 设备都被抽象成文件，每个文件对应一个文件描述符，是一个`int`类型整数，从 0 开始

当操作系统创建一个进程时，会初始化 3 个打开的文件描述符，分别是：标准输入 (0)、标准输出 (1)、标准错误 (2)。后续如果进程自己打开文件，就会接着往后分配文件描述符

举个例子，如果打开一个文件，会分配文件描述符 3；再打开一个文件，会分配文件描述符 4，以此类推；如果关闭文件描述符 3，然后再打开一个文件，会分配文件描述符 3，会复用前面关闭的文件描述符

### <font color=#1FA774>socket 网络通信</font>

先从最简单的 socket 通信说起。客户端/服务端架构的通信是基于 socket 实现，socket 封装了网络通信的细节，对程序员提供简单易用的接口，两者通信的过程：

![13](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230516/2113151684242795VU1ePx13.svg)

稍微解释一下上图中的步骤：

- `socket`：创建一个套接字描述符。对于客户端/服务端来说，它们互为对方的 I/O 设备，会被抽象成一个文件，可以通过套接字描述符来操作客户端/服务端的行为
- `connect`：客户端调用 connect 函数和服务端建立连接
- `bind`：服务端调用 bind 函数将 ip 和 port 与套接字绑定
- `listen`：将服务端套接字设置为监听模式
- `accept`：服务端调用 accept 函数等待来自客户端的连接请求
- `write`：向对方写数据
- `read`：从对方读数据

下面给出一个小 demo：

```c
/************ 服务端 ************/
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <arpa/inet.h>
int main(int argc, char *argv[]) {
    if (argc != 2) { printf("Using: ./server port\nExample: ./server 5005\n\n"); return -1; }
    // 第 1 步：创建服务端 socket
    int listenfd;
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) { perror("socket"); return -1; }
    // 第 2 步：把服务端用于通信的地址和端口绑定到 socket 上
    struct sockaddr_in servaddr;    // 服务端地址信息的数据结构
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;  // 协议族，在 socket 编程中只能是 AF_INET
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);  // 任意 ip 地址
    // servaddr.sin_addr.s_addr = inet_addr("xxx.xxx.xxx.xxx");  // 指定 ip 地址
    servaddr.sin_port = htons(atoi(argv[1]));  // 指定通信端口
    if (bind(listenfd, (struct sockaddr *) &servaddr, sizeof(servaddr)) != 0) {  // 绑定
        perror("bind"); close(listenfd);  // 关闭文件
        return -1;
    }
    // 第 3 步：把 socket 设置为监听模式
    if (listen(listenfd, 5) != 0) {
        perror("listen"); close(listenfd);  // 关闭文件
        return -1;
    }
    // 第 4 步：接受客户端的连接
    while (1) {
        int connetfd;  // 客户端 socket
        int socklen = sizeof(struct sockaddr_in);  // struct sockaddr_in 的大小
        struct sockaddr_in clientaddr;  // 客户端的地址信息
        connetfd = accept(listenfd, (struct sockaddr_in *) &clientaddr, (socklen_t *) &socklen);

        printf("客户端 (%s) 已连接。\n", inet_ntoa(clientaddr.sin_addr));
        // 第 5 步：与客户端通信，接收客户端发过来的报文，回复 OK
        char buffer[1024];
        while (1) {
            int iret;
            memset(buffer, 0, sizeof(buffer));
            if ((iret = recv(connetfd, buffer, sizeof(buffer), 0)) <= 0) {  // 接收客户端的请求报文
                printf("iret = %d\n", iret); break;
            }
            printf("接收：%s\n", buffer);
            strcpy(buffer, "ok");
            if ((iret = send(connetfd, buffer, strlen(buffer), 0)) <= 0) {  // 向客户端发送响应结果
                perror("send"); break;
            }
            printf("发送：%s\n", buffer);
        }
    }
    // 第 6 步：关闭 socket，释放资源
    // close(listenfd);
    // close(connetfd);
}
/************ 客户端 ************/
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>
#include <netdb.h>
#include <sys/socket.h>
#include <arpa/inet.h>
int main(int argc, char *argv[]) {
    if (argc != 3) { printf("Using: ./client ip port\nExample: ./client 127.0.0.1 5005\n\n"); return -1; }
    // 第 1 步：创建客户端 socket
    int sockfd;
    if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) { perror("socket"); return -1; }
    // 第 2 步：向服务器发起连接请求
    struct hostent* h;
    if ((h = gethostbyname(argv[1])) == 0) {  // 指定服务端的 ip 地址
        printf("gethostbyname failed.\n"); close(sockfd); return -1;
    }
    struct sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons((atoi(argv[2])));  // 指定服务端的通信端口
    memcpy(&servaddr.sin_addr, h->h_addr, h->h_length);
    if (connect(sockfd, (struct sockaddr *) &servaddr, sizeof(servaddr)) != 0) {  // 向服务端发起连接请求
        perror("connect"); close(sockfd); return -1;
    }
    // 第 3 步：与服务端通信，发送一个报文后等待回复，然后再发送下一个报文
    char buffer[1024];
    for (int i = 0; i < 3; i++) {
        int iret;
        memset(buffer, 0, sizeof(buffer));
        sprintf(buffer, "这是第 %d 条消息", i + 1);
        if ((iret = send(sockfd, buffer, strlen(buffer), 0)) <= 0) {  // 向服务端发送请求报文
            perror("send"); break;
        }
        printf("发送：%s\n", buffer);
        memset(buffer, 0, sizeof(buffer));
        if ((iret = recv(sockfd, buffer, sizeof(buffer), 0)) <= 0) {  // 接收服务端回应报文
            printf("iret = %d\n", iret); break;
        }
        printf("接收：%s\n", buffer);
    }
    // 第 4 步：关闭 socket，释放资源
    close(sockfd);
}
```

### <font color=#1FA774>事件驱动</font>

I/O 多路复用是基于事件驱动的模型，可以将 socket 网络通信过程抽象成两类事件：

- **可读事件：**当 socket 变得可读时 (客户端对 socket 执行 write 操作或者 close 操作) 或者有新的连接 socket 出现时
- **可写事件：**当 socket 变得可写时 (客户端对 socket 执行 read 操作)

由于通信过程中，客户端始终只和一个服务端建立通信，所以不需要 I/O 多路复用，但服务端会接收来自多个客户端的请求，所以需要 I/O 多路复用。所以客户端是事件的产生者，服务端接收事件并处理它

在传统的 I/O 模型中，都是基于连接驱动，在单线程中每次只能处理完一个客户端连接后才能和其它客户端建立新的连接，处理流程如下：

![14](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230516/21495916842449990v1cJ814.svg)

客户端保持连接过程中可能会产生多个事件，基于事件驱动的模型相当于使应用程序处理任务的粒度变细。假设两个连接分别会产生三个事件：A1、A2、A3、B1、B2、B3

在传统的 I/O 模型中，按照连接时间的先后顺序，假设 A 先与服务端建立连接，那么必须处理完 A1、A2、A3 三个事件后才可以断开连接，然后和 B 建立连接处理 B1、B2、B3

一旦 A1、A2、A3 三个任务中有一个任务处理时间过长，那么直接会增加后面准备连接客户端的阻塞时间。在基于事件驱动的模型中，只要 A1 处理完就可以处理 A2 或者 B1，不需要处理完 A 的所有事件

至于事件如何调度，也就是如何判断先处理哪个事件，完全依赖于哪个事件先发生，先发生的事件先处理，基于事件驱动的模型处理流程如下：

![15](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230516/2200201684245620kRXfAx15.svg)

### <font color=#1FA774>select</font>

说完了事件驱动，也就间接的引出了 I/O 多路复用，本部分主要侧重于从细节和实现角度介绍 select。**关于 select 实现 I/O 多路复用的概念可见 [I/O 多路复用](../java/IO模型.html#io-多路复用)**

在多线程/多进程的 Socket 服务端模式下：(**更多可见 [Socket 服务端模式](../java/IO模型.html#socket-服务端模式)**)

- **客户端：**阻塞在 read 函数
- **服务端：**阻塞在 accpet 函数

在 I/O 多路复用下，客户端和服务端全都阻塞在 select 函数，当有事件发生，select 函数返回

即然 select 可以在有事件发生时立刻返回，那么肯定需要保存所有建立连接的 socket，然后根据是否有事件发生来决定 select 是否返回，所以有一个`fd_set`集合专门保存 socket

另外一个问题，哪些 socket 需要放到`fd_set`集合中呢？

- 服务端创建的监听 socket，主要用于发现新客户端连接事件
- 所有和服务端建立连接的客户端 socket，主要用于发现客户端产生的可读可写事件

下面是 select 的处理流程：

![16](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230516/22403716842480371OFM2A16.svg)

下面给出一个小 demo：

```c
/************ 服务端 ************/
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/fcntl.h>

// 初始化服务端的监听端口
int initserver(int port);

int main(int argc,char *argv[]) {
    if (argc != 2) { printf("usage: ./tcpselect port\n"); return -1; }

    // 初始化服务端用于监听的 socket
    int listenfd = initserver(atoi(argv[1]));
    printf("listenfd = %d\n", listenfd);

    if (listenfd < 0) { printf("initserver() failed.\n"); return -1; }

    fd_set readfds;     // 读事件 socket 的集合，包括监听 socket 和客户端连接上来的 socket
    FD_ZERO(&readfds);  // 初始化读事件 socket 的集合
    FD_SET(listenfd, &readfds);  // 把 listenfd 添加到读事件 socket 的集合中
    int maxfd = listenfd; // 记录集合中 socket 的最大值
    
    while (1) {
        // 调用 select 函数时，会改变 socket 集合的内容，所以要把 socket 集合保存下来，传一个临时的 fd_set 给 select
        fd_set tmpfds = readfds;
        struct timeval timeout;
        timeout.tv_sec = 10; timeout.tv_usec = 0;

        int infds = select(maxfd + 1, &tmpfds, NULL, NULL, &timeout);  // 阻塞在此处
        // 失败
        if(infds < 0) { perror("select() failed"); break; }
        // 超时
        if (infds == 0) { printf("select() timeout.\n"); continue; }

        // 如果 infds > 0，表示有事件发生；infds 表示有事件发生的 socket 数量
        for (int eventfd = 0; eventfd <= maxfd; ++eventfd) {  // 有事件发生时，从 0 遍历到最大值，找出发生事件的 socket
            if (FD_ISSET(eventfd, &tmpfds) <= 0) continue;    // 如果没有事件，continue

            // 如果发生事件的是 listenfd，表示有新的客户端连上来
            if (eventfd == listenfd) {
                struct sockaddr_in client;
                socklen_t len = sizeof(client);
                int clientfd = accept(listenfd, (struct sockaddr *) &client, &len);  // 获取新的客户端的 socket
                if (clientfd < 0) { perror("accept() failed"); continue; }

                printf("accept client (socket = %d) ok.\n", clientfd);

                // 把新客户端的 socket 加入可读 socket 的集合，注意是原集合，而非临时集合
                FD_SET(clientfd, &readfds);
                if (maxfd < clientfd) maxfd = clientfd;  // 更新 maxfd 的值
            } else {
                // 如果是客户端连接的 socket 有事件发生，表示有报文发过来或者连接已断开
                char buffer[1024];  // 存放从客户端读取的数据
                memset(buffer, 0, sizeof(buffer));
                if (read(eventfd, buffer, sizeof(buffer)) <= 0) {
                    // <= 0 表示客户端连接已断开
                    printf("client (eventfd = %d) disconnected.\n", eventfd);
                    close(eventfd);  // 关闭客户端的 socket
                    FD_CLR(eventfd, &readfds);  // 把已关闭客户端的 socket 从可读 socket 的集合中删除

                    // 重新计算 maxfd 的值。注意，只有当 eventfd = maxfd 才需要计算，表示移除了最大 socket
                    if (eventfd == maxfd) {
                        for (int i = maxfd; i > 0; i--) {
                            if (FD_ISSET(i, &readfds)) { maxfd = i; break; }
                        }
                    }
                } else {
                    // > 0 表示有客户端发送报文
                    printf("recv (eventfd = %d) : %s\n", eventfd, buffer);
                    // 把接收到的报文内容原封不动的发回去
                    // select 写会直接写到操作系统缓冲区，速度很快，一般不需要 I/O 多路复用
                    fd_set tmpfds;
                    FD_ZERO(&tmpfds);
                    FD_SET(eventfd, &tmpfds);
                    if (select(eventfd + 1, NULL, &tmpfds, NULL, NULL) <= 0) {
                        perror("select() failed");
                    } else {
                        write(eventfd, buffer, strlen(buffer));
                    }
                }
            }
        }
    }

    return 0;
}

// 初始化服务端的监听端口
int initserver(int port) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) { perror("socket() failed"); return -1; }
    int opt = 1;
    unsigned int len = sizeof(opt);
    setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &opt, len);

    struct sockaddr_in servaddr;
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(port);

    if (bind(sock, (struct sockaddr *) &servaddr, sizeof(servaddr)) < 0) { perror("bind() failed"); close(sock); return -1; }
    if (listen(sock, 5) != 0) { perror("listen() failed"); close(sock); return -1; }
    return sock;
}
```

#### <font color=#9933FF>fd_set</font>

fd_set 是专门用来存放 socket 的集合，是一个位图，每一位表示一个 socket，0 表示 socket 没有事件发生，1 表示 socket 有事件发生

fd_set 一共有 1024 位，也就是说 select 最多处理 1024 个 socket

```c
#define __DARWIN_FD_SETSIZE     1024
#endif /* FD_SETSIZE */
#define __DARWIN_NBBY           8                               /* bits in a byte */
#define __DARWIN_NFDBITS        (sizeof(__int32_t) * __DARWIN_NBBY) /* bits per mask */
#define __DARWIN_howmany(x, y)  ((((x) % (y)) == 0) ? ((x) / (y)) : (((x) / (y)) + 1)) /* # y's == x bits? */
typedef struct fd_set {
	__int32_t       fds_bits[__DARWIN_howmany(__DARWIN_FD_SETSIZE, __DARWIN_NFDBITS)];
} fd_set;
```

**<font color='red'>问题：</font>**为什么需要将 fd_set 拷贝一份，然后把拷贝后的 fd_set 传入 select 中？

首先 fd_set 中保存了所有的连接到服务端的套接字描述符，而 select 要从 fd_set 中找出有事件发生的描述符，然后交由服务端处理

select 会直接在传入的 fd_set 集合中修改，将没有发生事件的描述符置 0，保留发生事件的描述符为 1，所以如果没有传入副本，将丢失记录的套接字描述符

为了画图方便，假设 fd_set 只有 32 位，但实际有 1024 位。假设目前 fd_set 有套接字描述符 0、1、2、3、5、10，且套接字描述符 0、3、10 有事件发生，那么如下图所示：

![17](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230517/0052511684255971Gr3ln117.svg)

#### <font color=#9933FF>缺点</font>

- select 支持的描述符数量少，默认 1024。虽然可以调整，但描述符数量越大，效率越低
- 每次调用 select，都需要将 fd_set 从用户态拷贝到内核态，因为 fd_set 定义在用户内存空间，而系统调用是操作系统执行
- 对于仅有少量客户端有事件发生的情况，也需要遍历整个 fd_set 集合找到发生事件的描述符

对于第 1 点和第 3 点是此长彼消的关系，当支持描述符多了意味着遍历更耗时，当遍历耗时低了意味着支持描述符少了

#### <font color=#9933FF>问题：为什么 I/O 多路复用要搭配 NIO？</font>

在 select 实现的 I/O 多路复用中，所有连接服务端的 socket 都会加入到 fd_set 中，并阻塞在 select 系统调用处，当有 socket 发生事件时，也就是可读或可写时，会从 select 函数返回

假设有一个可读事件发生，表示应用程序可读数据，那么这个读操作是设置成阻塞的还是非阻塞的呢？有人觉得设置成阻塞或非阻塞效果都一样，因为 select 返回表示数据可读

但是「select 返回」和「正式开始读数据」之间存在时间窗口，可能存在数据可读后由于校验错误等原因会将数据丢弃，此时就无法读到数据，如果是阻塞 I/O，这种情况就会阻塞整个应用程序

在 Linux 中通过`man select `命令可以看到 select 的介绍，在 BUGS 部分解释了为什么可读但不一定读的到数据！！

```
Under Linux, select() may report a socket file descriptor as "ready for reading", while nevertheless a subsequent read blocks.  This could for example happen when data has arrived but upon examination has wrong checksum and is  discarded. There may be other circumstances in which a file descriptor is spuriously reported as ready.  Thus it may be safer to use O_NONBLOCK on sockets that should not block.
```

**关于该问题更详细的解释可见 [为什么 IO 多路复用要搭配非阻塞 IO?](https://www.zhihu.com/question/37271342)**

### <font color=#1FA774>poll</font>

poll 和 select 几乎一模一样，唯一不同的在于存储套接字描述符集合的数据结构

- select 中使用位图，而且每次调用 select 时都需要创建一个副本传入
- poll 中使用结构体 pollfd 数组，记录了套接字描述符是否有发生事件，所以不需要创建副本传入，直接传原始数组即可

```c
struct pollfd {
	int     fd;        // 文件描述符
	short   events;    // 等待的事件
	short   revents;   // 实际发生的事件
};
```

下面给出一个小 demo：

```c
/************ 服务端 ************/
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <poll.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/fcntl.h>

#define MAXNFDS 1024

// 初始化服务端的监听端口 (同 select)
int initserver(int port);

int main(int argc, char *argv[]) {

    if (argc != 2) { printf("usage: ./tcppoll port\n"); return -1; }

    // 初始化服务端用于监听的 socket
    int listenfd = initserver(atoi(argv[1]));
    printf("listenfd = %d\n", listenfd);

    if (listenfd < 0) { printf("initserver() failed.\n"); return -1; }

    struct pollfd fds[MAXNFDS];  // fds 存放需要监视的 socket
    for (int i = 0; i < MAXNFDS; i++) fds[i].fd = -1;   // 初始化数组，把全部的 fd 设置为 -1

    // 把 listenfd 和读事件添加到数组中
    fds[listenfd].fd = listenfd;
    // POLLIN 表示数据可读；POLLOUT 表示数据可写
    fds[listenfd].events = POLLIN;

    int maxfd = listenfd;  // fds 数组中需要监视的 socket 的大小

    while (1) {
        int infds = poll(fds, maxfd + 1, 5000);  // 最后一个参数单位：毫秒

        // 失败
        if (infds < 0) { perror("poll() failed"); break; }

        // 超时
        if (infds == 0) { printf("poll() timeout.\n"); continue; }

        // 如果 infds > 0，表示有事件发生的 socket 数量
        for (int eventfd = 0; eventfd <= maxfd; eventfd++) {
            if (fds[eventfd].fd < 0) continue;    // 如果 fd 为负数，忽略它
            if ((fds[eventfd].revents & POLLIN) == 0) continue;  // 如果没有事件，直接 continue
            fds[eventfd].revents = 0;  // 先把 revents 清空

            // 如果发生事件的是 listenfd，表示有新的客户端连接
            if (eventfd == listenfd) {
                struct sockaddr_in client;
                socklen_t len = sizeof(client);
                int clientfd = accept(listenfd, (struct sockaddr *) &client, &len);
                if (clientfd < 0) { perror("accept() failed"); continue; }

                printf("accept client(socket = %d) ok.\n", clientfd);

                // 修改 fds 数组中 clientfd 位置的元素
                fds[clientfd].fd = clientfd;
                fds[clientfd].events = POLLIN;
                fds[clientfd].revents = 0;
                if (maxfd < clientfd) maxfd = clientfd;    // 更新 maxfd 的值
            } else {
                // 如果是客户端连接的 socket 有事件，表示有报文发过来或者连接已断开
                char buffer[1024];  // 存放从客户端读取的数据
                memset(buffer, 0, sizeof(buffer));
                if (read(eventfd, buffer, sizeof(buffer)) <= 0) {
                    // 如果客户端的连接已断开
                    printf("client (eventfd = %d) disconnected.\n", eventfd);
                    close(eventfd); // 关闭客户端的 socket
                    fds[eventfd].fd = -1;

                    // 重新计算 maxfd 的值，注意，只有当 eventfd = maxfd 时才需要计算
                    if (eventfd == maxfd) {
                        for (int i = maxfd; i > 0; i--) {  // 从后面往前找
                            if (fds[i].fd != -1) { maxfd = i; break; }
                        }
                    }
                } else {
                    // 如果客户端有报文发过来
                    printf("recv (eventfd = %d): %s\n", eventfd, buffer);
                    write(eventfd, buffer, strlen(buffer));
                }
            }
        }
    }
    return 0;
}
```

#### <font color=#9933FF>缺点</font>

- 虽然不需要拷贝 pollfd 数组，但也需要将它传入内核；而且也需要通过遍历数组寻找发生的事件

### <font color=#1FA774>epoll</font>

epoll 是在 2.6 内核中提出，是 select/poll 的增强版

相比于 select/poll 更加的灵活，没有描述符数量的限制，使用 1 个描述符管理多个描述符，并将这些描述符存放到内核中，这样就可以只在用户空间和内核空间 copy 一次，select/poll 需要两次

epoll 提供了三个函数调用接口：

```c
// 创建 epoll 句柄
int epoll_create(int size);
// 对指定描述符 fd 执行 op 操作，op 操作有：
// - EPOLL_CTL_ADD 添加
// - EPOLL_CTL_DEL 删除
// - EPOLL_CTL_MOD 修改
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 等待 epfd 上发生的事件，最多返回 maxevents 个，存到 events 中
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

下面给出一个小 demo：

```c
/************ 服务端 ************/
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/fcntl.h>
#include <sys/epoll.h>
#include <errno.h>

// 初始化服务端的监听端口 (同 select)
int initserver(int port);

int main(int argc,char *argv[]) {
    if (argc != 2) { printf("usage: ./tcpepoll port\n"); return -1; }

    // 初始化服务端用于监听的 socket
    int listenfd = initserver(atoi(argv[1]));
    printf("listenfd = %d\n",listenfd);

    if (listenfd < 0) { printf("initserver() failed.\n"); return -1; }
    // 创建 epoll 句柄
    int epollfd = epoll_create(1);

    // 为监听的 socket 准备可读事件
    struct epoll_event ev;  // 声明事件的数据结构
    ev.events = EPOLLIN;    // 读事件 默认水平触发。边缘触发：ev.events = EPOLLIN | EPOLLET
    ev.data.fd = listenfd;  // 指定事件的自定义数据，会随着 epoll_wait() 返回

    // 把监听的 listenfd 加入 epollfd 中
    epoll_ctl(epollfd, EPOLL_CTL_ADD, listenfd, &ev);

    struct epoll_event evs[10];  // 存放 epoll 返回的事件

    while (1) {
        // 等待监视的 socket 有事件发生。最后一个参数是超时时间 ms
        int infds = epoll_wait(epollfd, evs, 10, -1);

        // 失败
        if(infds < 0) { perror("select() failed"); break; }

        // 超时
        if (infds == 0) { printf("select() timeout.\n"); continue; }

        // 如果 infds > 0，表示有事件发生的 socket 数量。遍历 epoll 返回的已发生事件的数组
        for(int i = 0; i < infds; ++i){
            // 如果发生事件的是 listenfd，表示有新的客户端连上来
            if (evs[i].data.fd == listenfd && (evs[i].events & EPOLLIN)) {
                struct sockaddr_in client;
                socklen_t len = sizeof(client);
                int clientfd = accept(listenfd, (struct sockaddr *) &client, &len);
                if (clientfd < 0) { perror("accept() failed"); continue; }
                // 为新客户端准备可读事件，并添加到 epoll 中
                memset(&ev, 0, sizeof(struct epoll_event));
                ev.data.fd = clientfd;
                ev.events = EPOLLIN;
                epoll_ctl(epollfd, EPOLL_CTL_ADD, clientfd, &ev);
            } else if (evs[i].events & EPOLLIN) {
                // 如果是客户端连接的 socket 有事件，表示有报文发过来或者连接已断开
                char buffer[1024];  // 存放从客户端读取的数据
                memset(buffer, 0, sizeof(buffer));

                ssize_t isize = read(evs[i].data.fd, buffer, sizeof(buffer));
                // 发生错误或者 socket 被对方关闭
                if (isize <= 0) {
                    printf("client(eventfd = %d) disconnected.\n", evs[i].data.fd);
                    // 把断开的客户端从 epoll 中删除
                    memset(&ev, 0, sizeof(struct epoll_event));
                    ev.data.fd = evs[i].data.fd;
                    ev.events = EPOLLIN;
                    epoll_ctl(epollfd, EPOLL_CTL_DEL, evs[i].data.fd, &ev);
                    close(evs[i].data.fd);
                    continue;
                }
                printf("recv(eventfd = %d, size = %ld) : %s\n", evs[i].data.fd, isize, buffer);
                // 把接收到的报文内容原封不动的发回去
                write(evs[i].data.fd, buffer, strlen(buffer));
            }
        }
    }
    close(epollfd);
    return 0;
}
```

#### <font color=#9933FF>对比 select/poll</font>

在 select/poll 中，只有在进程调用一定的方法后，内核才对所有监视的描述符扫描，寻找有事件发生的描述符；而 epoll 是先通过`epoll_ctl()`注册一个描述符，一旦基于某个描述符就绪时，内核就会采用类似 callback 的回调机制，迅速激活这个描述符，唤醒`epoll_wait()`处阻塞的进程

在 select/poll 中，会有两次用户空间和内核空间的 copy 过程，而 epoll 中由于描述符直接存放在内核中，所以只会有一次用户空间和内核空间的 copy 过程

在 select/poll 中，需要遍历整个集合寻找发生事件的描述符，而 epoll 中`epoll_wait()`返回的数组中只有发生事件的描述符

### <font color=#1FA774>参考文章</font>

- **操作系统导论**
- **深入理解计算机系统**
- **Redis 设计与实现**
- **[C/C++网络编程，从socket到epoll](https://www.bilibili.com/video/BV11Z4y157RY)**
- **[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)**
- **[epoll早期使用了mmap， 后面没有用了？](https://segmentfault.com/q/1010000022582226)**
- **[epoll源码解析翻译------说使用了mmap的都是骗子](https://www.cnblogs.com/l2017/p/10830391.html)**