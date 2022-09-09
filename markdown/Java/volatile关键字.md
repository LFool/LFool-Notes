[toc]

# volatile 关键字

正式介绍 volatile 之前，先来一点铺垫！！

### <font color=#1FA774>缓存一致性协议 (MESI)</font>

我们知道处理器 (CPU) 的速度是很快的，但是绝大多数任务仅仅依靠 CPU 是很难完成的，CPU 至少需要与内存交互，如读取运算数据、存储运算结果等

计算机的存储设备与 CPU 的运算速度差的不止一点，所以现在计算机就在 CPU 和内存之间加入了一层或多层读写速度更接近 CPU 运算速度的**高速缓存 (Cache)**

基于高速缓存的存储交互很好地解决了处理器与内存速度之间的矛盾，但也引入了一个新的问题：缓存一致性问题

在多核 CPU 系统中，每个 CPU 都有自己的高速缓存，而它们又共享同一主内存，如下图所示：

![3](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220909/1723181662715398PJCsTO3.svg)

当多核处理器的运算任务都涉及同一块主内存区域时，将可能导致各自的缓存数据不一致，为解决这一问题，需要各处理器访问缓存时遵循一些协议，其中主要的协议就是 MESI

MESI 表示 Cache Line 的四种状态：

- modify：CPU 拥有该 Cache Line 且将其做了修改，CPU 需要保护在重用该 Cache Line 存其他数据前，将修改的数据写入主内存，或者将 Cache Line 转交给其他 CPU 所有
- exclusive：跟 modify 类似，也表示 CPU 拥有某个 Cache Line，但还未来得及对它做出修改，CPU 可直接将里面数据丢弃或者转交给其他 CPU
- shared：Cache Line 的数据是最新的，可以丢弃或转交给其它 CPU，但当前 CPU 不能对其进行修改。要修改的话需要转为 exclusive 状态后再进行
- invalid：Cache Line 内的数据为无效，也相当于没存数据。CPU 在找空 Cache Line 缓存数据的时候就是找 invalid 状态的 Cache Line

这里推荐一个可视化网站，可以看到 Cache 四个状态的转换 **[VivioJS MESI animation help](https://www.scss.tcd.ie/Jeremy.Jones/VivioJS/caches/MESIHelp.htm)**

### <font color=#1FA774>Java 内存模型 (JMM)</font>

Java 内存模型规定所有的变量都存储在**主内存**中，每个线程都有自己的**工作内存**

线程的工作内存中保存了被该线程使用的变量的**主内存副本**，线程对变量的所有操作「读取、赋值等」都必须在工作内存中进行，而不能直接读写主内存中的数据

线程之间也无法访问对方工作内存中的变量，即工作内存属于线程私有。线程间变量的传递需要通过主内存来完成

是不是感觉和上面的关系十分相似，对，没错，JMM 就是参考计算机硬件的交互关系

线程、主内存、工作内存三者的交互关系如下图所示：

![1](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220909/1634241662712464U71uGT1.svg)

与此同时，JMM 还定义了一套内存间的交互操作

- lock (锁定)：作用于主内存的变量，把一个变量标识为一个线程独占的状态
- unlock (解锁)：作用于主内存的变量，把一个处于锁定状态的变量释放出来，释放后的变量才可被其他线程锁定
- read (读取)：作用于主内存的变量，把一个变量的值从主内存传输到线程的工作内存中，以便随后的 load 操作使用
- load (载入)：作用于工作内存的变量，把 read 操作从主内存中得到的变量值放入工作内存的变量副本中
- use (使用)：作用于工作内存的变量，把工作内存中的一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用的变量的值的字节码指令时将会执行这个操作
- assign (赋值)：作用于工作内存的变量，把一个从执行引擎接收的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作
- store (存储)：作用于工作内存的变量，把工作内存中一个变量的值传递到主内存中，以便随后的 write 操作使用
- write (写入)：作用于主内存的变量，把 store 操作从工作内存中得到的变量的值放入主内存的变量中

具体的工作流程如下图所示：

![2](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220909/1711161662714676UvyIsQ2.svg)

### <font color=#1FA774>基本概念引入</font>

#### <font color=#9933FF>可见性</font>

可见性是指多个线程对同一片内存区域的写入操作是否立即可见

由于每个线程都有自己的工作内存，且为线程私有，相互不可见，线程对变量的所有操作「读取、赋值等」都必须在工作内存中进行

如果一个线程修改了共享变量，但没有马上写回到主内存中，这导致其他线程对该共享变量的修改不可见

#### <font color=#9933FF>指令重排序</font>

除了增加高速缓存外，为了使处理器的运算单元可以被更充分的利用，处理器会对输入代码进行**乱序执行优化**，处理器对乱序执行的结果进行重组，保证该结果与顺序执行的结果是一致的，但并不保证各个执行语句的先后顺序与输入代码的顺序一致，即只保证**「最终一致性」**

Java 虚拟机的即时编译器中也有指令重排序优化

「指令重排序优化」在单线程中不会出现任何问题，但是如果在多线程中，会出现意想不到的问题，具体例子可见 **[双重校验锁实现单例模式](./单例模式.html#双重校验锁)**

#### <font color=#9933FF>内存屏障</font>

这里先说明一下，下面说到的 Load 和 Store 分别指「从主内存中读」和「往主内存中写」。根据上面提到的内存间的交互操作，Load 等价于 read + load；Store 等价于 store + write 

JVM 根据读、写两种操作提供了四种屏障

- LoadLoad：「Load1 LoadLoad Load2」用于保证访问 Load2 的读取操作一定不能排到 Load1 的前面
- StoreStore：「Store1 StoreStore Store2」用于保证 Store1 及其之后写出的数据一定先于 Store2，即其他线程一定先看到 Store1 的数据，再看到 Store2 的数据
- LoadStore：「Load1 LoadStore Store2」用于保证 Store2 及其之后写出的数据被其他线程看到之前，Load1 读取的数据一定先读入了缓存
- StoreLoad：「Store1 StoreLoad Load2」用于保证 Store1 写出的数据被其他线程看到后才能读取 Load2 的数据到缓存

### <font color=#1FA774>volatile 的作用</font>

铺垫了这么多，现在进入正题！！

#### <font color=#9933FF>volatile 可见性</font>

当一个变量定义为 volatile 后，那么该变量**<font color='red'>对所有线程立即可见</font>**。更具体的：

- 当一个线程修改了这个变量的值，新值对其他 CPU 来说是可以立即得知的
- 修改 volatile 变量之前的操作在其他 CPU 看到 volatile 的最新值后也一定能被看到

volatile 变量的内存可见性是基于**内存屏障**实现

- 读取 volatile 变量先检查本地缓存是否失效，如果失效就去内存中读取最新值
- 禁止读 volatile 变量后续操作被重排到读 volatile 变量前面 (如果后续操作重排到前面，说明后续操作读了旧值)
- 保证写 volatile 变量之前操作优先于写 volatile 变量之前发生

下面给出一段「双重校验锁实现单例模式」的部分汇编代码：

```assembly
0x01a3de0f: mov $0x3375cdb0,%esi 	  ;...beb0cd75 33
									  ;   {oop('Singleton')}
0x01a3de14: mov %eax,0x150(%esi) 	  ;...89865001 0000
0x01a3de1a: shr $0x9,%esi 			  ;...c1ee09
0x01a3de1d: movb $0x0,0x1104800(%esi) ;...c6860048 100100
0x01a3de24: lock addl $0x0,(%esp) 	  ;...f0830424 00
									  ;*putstatic instance
									  ; - Singleton::getInstance@24
```

在赋值操作后，多了一条指令：`lock addl $0x0,(%esp)`，它的作用相当于一个内存屏障，重排序时不能把后面的指令重排序到内存屏障之前的问题

`addl $0x0,(%esp)`是一个空操作，关键在于`lock`，它的作用：

- 将当前处理器缓存行的数据写回到内存
- 写回内存的操作会使其他处理器中缓存了该内存地址的数据无效

**下面开始说人话！！**

为了提高处理速度，处理器并不是直接和内存通信，而是先将系统内存的数据读到自己的工作内存中 (缓存) 后再进行其他操作，但操作完不知道何时会写到内存

如果 volatile 变量进行写操作，JVM 就会向处理器发送一条 lock 前缀的指令，将这个变量所在的缓存行的数据写回到系统内存

当处理器发现本地缓存失效后，就会从内存中重新读取该变量数据，即可以获得当前最新值

#### <font color=#9933FF>volatile 的 happens-before 关系</font>

happens-before 规则中有一条是 volatile 变量规则

**<font color='red'>Volatile variable rule. A write to a volatile field happens‐before every subsequent read of that same field.</font>**

**解释：**对 volatile 变量的写入操作必须在对该变量的读操作之前执行

```java
// 假设线程 A 执行 writer 方法，线程 B 执行 reader 方法
public class VolatileExample {
    int a = 0;
    volatile boolean flag = false;
    public void writer() {
        a = 1;                // 1 线程 A 修改共享变量
        flag = true;          // 2 线程 A 写 volatile 变量
    }
    public void reader() {
        if (flag) {           // 3 线程 B 读同一个 volatile 变量
            int i = a;        // 4 线程 B 读共享变量
            // other
        }
    }
}
```

根据 happens-before 规则，上面过程会建立 3 类 happends-before 关系：

- 根据程序次序规则：1 happens-before 2 且 3 happens-before 4
- 根据 volatile 规则：2 happens-before 3
- 根据 happens-before 的传递性规则：1 happens-before 4

![4](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220909/2048331662727713YHS2Ii4.svg)

#### <font color=#9933FF>禁止指令重排序优化</font>

Java 编译器会在生成指令系列时在适当的位置会插入内存屏障指令来禁止特定类型的处理器重排序

JMM 会针对编译器制定 volatile 重排序规则表

|             | 普通读/写 | volatile 读 | volatile 写 |
| :---------: | :-------: | :---------: | :---------: |
|  普通读/写  |           |             |     NO      |
| volatile 读 |    NO     |     NO      |     NO      |
| volatile 写 |           |     NO      |     NO      |

注：NO 表示禁止重排序！

为了实现 volatile 内存语义时，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序

对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎是不可能的。因此，JMM 采取了保守的策略

- 在每个 volatile 写操作的前面插入一个 StoreStore 屏障
- 在每个 volatile 写操作的后面插入一个 StoreLoad 屏障
- 在每个 volatile 读操作的后面插入一个 LoadLoad 屏障
- 在每个 volatile 读操作的后面插入一个 LoadStore 屏障

![5](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220909/2116501662729410M3Iwe65.svg)

#### <font color=#9933FF>volatile 原子性？？</font>

volatile 不能保证完全的原子性，只能保证单次的读/写操作具有原子性

**<font color='red'>问题 1：i++ 为什么不能保证原子性？？</font>**

```java
public class VolatileTest {
    public static volatile int race = 0;
    public static void increase() {
        race++;
    }
    private static final int THREADS_COUNT = 20;
    public static void main(String[] args) {
        Thread[] threads = new Thread[THREADS_COUNT];
        for (int i = 0; i < THREADS_COUNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    increase();
                }
            });
            threads[i].start();
        }
        while (Thread.activeCount() > 2) {
            Thread.yield();
        }
        System.out.println(race);
    }
}
```

如果按照我们的设想，应该输出 200000，但是很不幸，并不是！！

因为本质上`race++`是读、写两次操作

- 首先读取 race 的值
- 然后对 race 加 1
- 最后将 race 的值写回内存

volatile 无法保证这三个操作是原子性的，如果想让结果正确，可以在`increase()`上加 synchronized 关键字

```java
public static synchronized void increase() {
    race++;
}
```

**<font color='red'>问题 2：共享的 long 和 double 变量为什么要用 volatile？？</font>**

Java 内存模型要求，变量的读取操作和写入操作都必须是原子操作，但对于非 volatile 类型的 long 和 double 变量，JVM 允许将 64 位的读操作或写操作分解为两个 32 位操作

当读取一个非 volatile 类型的 long 变量时，如果对该变量的读操作和写操作作为不同的线程执行，那么很可能会读取到某个值的高 32 位和另一个值的低 32 位

因此普通的 long 或 double 类型读/写可能不是原子的，所以鼓励大家将共享的 long 和 double 变量设置为 volatile 类型，这样能保证任何情况下对 long 和 double 单次读/写操作都具有原子性

### <font color=#1FA774>参考文章</font>

- 深入理解 Java 虚拟机

- Java 并发编程实战

- [内存屏障及其在-JVM 内的应用（上）](https://segmentfault.com/a/1190000022497646)

- [内存屏障及其在 JVM 内的应用（下）](https://segmentfault.com/a/1190000022508589)

- [关键字: volatile详解](https://pdai.tech/md/java/thread/java-thread-x-key-volatile.html)

- [The JSR-133 Cookbook for Compiler Writers](https://gee.cs.oswego.edu/dl/jmm/cookbook.html)
