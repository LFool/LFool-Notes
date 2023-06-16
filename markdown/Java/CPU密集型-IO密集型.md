# CPU 密集型 & I/O 密集型

本片文章介绍如何区分 CPU 密集型任务和 I/O 密集型任务～～

在设置 **[Java 线程池](../java/Java线程池.html)** 的参数时，需要参考到底是 CPU 密集型任务，还是 I/O 密集型任务，根据任务的不同类型，设置不同线程池大小：

- **对于 CPU 密集型任务：**设置线程池大小为 CPU 核心数 + 1。比 CPU 核心多 1 是为了防止线程偶发的缺页中断，有多余线程可以补上 CPU 空闲时间
- **对于 I/O 密集型任务：**设置线程池大小为 CPU 核心数 * 2。由于密集的 I/O 处理，增加了 CPU 处于空闲的概率

那么到底如何判断是 CPU 密集型任务，还是 I/O 密集型任务呢？

从宏观上分析，CPU 密集型任务多为计算型代码，需要始终在 CPU 上运行，而 I/O 密集型任务多为文件读写、DB 读写、网络请求等，总之是一切偏向于 I/O 操作的任务

上面的判断方法还是过于主观，比如如何判断是计算型代码呢？如何判断任务中更侧重于 I/O 操作呢？有没有一些硬性指标可以更加客观的去判断呢？？

在 Linux 中可以使用`top`命令查看进程的 CPU 占用率，如果一个进程大多数时候都是满载使用 CPU，那么可认为是 CPU 密集型；反之为 I/O 密集型

#### <font color=#9933FF>Demo -> CPU 密集型任务</font>

下面给出一个 CPU 密集型任务的 Demo：

```java
public class Main {
    public static void main(String[] args) {
        while (true){}
    }
}
```

说白了，上面的例子就是一个死循环，肯定会一直占用 CPU 资源，通过`jps`命令获得该任务的 pid：

```bash
[root@localhost ~]$ jps
1953 Jps
1897 Main
```

然后通过`top -p 1897`查看该任务的执行情况：

```bash
[root@localhost ~]$ top -p 1897
top - 15:56:28 up 14 min,  3 users,  load average: 1.01, 0.75, 0.38
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s): 12.4 us,  0.0 sy,  0.0 ni, 87.4 id,  0.0 wa,  0.1 hi,  0.0 si,  0.0 st
MiB Mem :   7741.8 total,   6927.3 free,    472.6 used,    341.9 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   7025.4 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                           
   1897 root      20   0 4726724  53576  16584 S  99.3   0.7   6:02.43 java
```

从结果中可以看出该任务的 CPU 占用率为 99.3%，可认为几乎满载了！！

#### <font color=#9933FF>Demo -> I/O 密集型任务</font>

下面给出一个 I/O 密集型任务的 Demo：

```java
public static void main(String[] args)  {
    try {
        BufferedReader in = new BufferedReader(new FileReader("test.txt"), 1024);
        String str;
        while ((str = in.readLine()) != null) {
            System.out.println(str);
        }
        System.out.println(str);
    } catch (IOException e) {
    }
}
```

上面例子中，`test.txt`是一个 6G 的大文件，通过`BufferedReader`读取到内存中，并将读取到的内容输出到终端

通过系统自带的性能监控软件可以看到该任务从磁盘读取了大量数据，可认为它是 I/O 密集性，如下图所示：

![image-20230617023434629](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230617/0234351686940475ejfx0Pimage-20230617023434629.png)