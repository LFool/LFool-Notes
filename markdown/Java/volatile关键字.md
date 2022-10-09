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

**关于「Java 内存模型」更详细的介绍可见 [Java 内存模型](./Java内存模型.html)**

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

**关于这些概念更详细的介绍可见 [Java 内存模型](./Java内存模型.html)**

#### <font color=#9933FF>可见性</font>

可见性是指一个线程对一片内存区域执行写入操作，其他线程立即可见该内存区域的改变

由于每个线程都有自己的工作内存，且为线程私有，相互不可见，线程对变量的所有操作「读取、赋值等」都必须在工作内存中进行

如果一个线程修改了共享变量，但没有马上写回到主内存中，这导致其他线程对该共享变量的修改不可见

#### <font color=#9933FF>指令重排序</font>

除了增加高速缓存外，为了使处理器的运算单元可以被更充分的利用，处理器会对输入代码进行**乱序执行优化**，处理器对乱序执行的结果进行重组，保证该结果与**单线程下**顺序执行的结果是一致的，但并不保证各个执行语句的先后顺序与输入代码的顺序一致，即只保证**「最终一致性」**

Java 虚拟机的即时编译器中也有指令重排序优化

「指令重排序优化」在单线程中不会出现任何问题，但是如果在多线程中，会出现意想不到的问题，具体例子可见 **[双重校验锁实现单例模式](./单例模式.html#双重校验锁)**

#### <font color=#9933FF>内存屏障</font>

这里先说明一下，下面说到的 Load 和 Store 分别指「从主内存中读」和「往主内存中写」。根据上面提到的内存间的交互操作，Load 等价于 read + load；Store 等价于 store + write 

JVM 根据读、写两种操作提供了四种屏障

- LoadLoad：「Load1 LoadLoad Load2」用于保证访问 Load2 的读取操作一定不能排到 Load1 的前面
- StoreStore：「Store1 StoreStore Store2」用于保证 Store1 及其之后写出的数据一定先于 Store2，即其他线程一定先看到 Store1 的数据，再看到 Store2 的数据
- LoadStore：「Load1 LoadStore Store2」用于保证 Store2 及其之后写出的数据被其他线程看到之前，Load1 读取的数据一定先读入了缓存
- StoreLoad：「Store1 StoreLoad Load2」用于保证 Store1 写出的数据被其他线程看到后才能读取 Load2 的数据到缓存

### <font color=#1FA774>volatile 的特性</font>

当我们声明共享变量为 volatile 后，对这个变量的读/写将会很特别

理解 volatile 特性的一个好方法是把对 volatile 变量的**单个读/写**，看成是使用同一个锁对这些单个读/写操作做了同步

下面我们通过具体的示例来说明，请看下面的示例代码：

```java
class VolatileFeaturesExample {
    volatile long vl = 0L;            // 使用 volatile 声明 64 位的 long 类型变量
    public void set(long l) {
        vl = l;                       // 单个 volatile 变量的写
    }
    public void getAndIncrement() {
        vl++;                         // 复合 volatile 变量的读/写
    }
    public long get() {
        return vl;                    // 单个 volatile 变量的读
    }
}
```

假设有多个线程分别调用上面程序的三个方法，这个程序在语义上和下面程序等价：

```java
class VolatileFeaturesExample {
    long vl = 0L;                           // 64 位的 long 类型普通变量
    public synchronized void set(long l) {  // 对单个普通变量的写用同一个锁同步
        vl = l;
    }
    public void getAndIncrement() {         // 普通方法调用
        long temp = get();                  // 调用已同步的读方法
        temp += 1L;                         // 普通写操作
        set(temp);                          // 调用已同步的写方法
    }
    public synchronized long get() {        // 对单个普通变量的读用同一个锁同步
        return vl;
    }
}
```

如上面示例程序所示，对一个 volatile 变量的**单个读/写操作**，与对一个普通变量的读/写操作使用同一个锁来同步，它们之间的执行效果相同

锁的 happens-before 规则保证释放锁和获取锁的两个线程之间的内存可见性

这意味着对一个 volatile 变量的读，总是能看到 (任意线程) 对这个 volatile 变量最后的写入

锁的语义决定了临界区代码的执行具有原子性。这意味着即使是 64 位的 long 类型和 double 类型变量，只要它是 volatile 变量，对该变量的读写就将具有原子性

如果是多个 volatile 操作或类似于 volatile++ 这种复合操作，这些操作整体上不具有原子性

简而言之，volatile 变量自身具有下列特性：

- **<font color='red'>可见性：对一个 volatile 变量的读，总是能看到 (任意线程) 对这个 volatile 变量最后的写入</font>**
- **<font color='red'>原子性：对任意单个 volatile 变量的读/写具有原子性，但类似于 volatile++ 这种复合操作不具有原子性</font>**

### <font color=#1FA774>volatile 写-读建立的 happens-before 关系</font>

**关于 happens-before 规则更详细的介绍可见 [Java 内存模型](./Java内存模型.html)**

上面讲的是 volatile 变量自身的特性，对程序员来说，volatile 对线程的内存可见性的影响比 volatile 自身的特性更为重要，也更需要我们去关注

从 JSR-133 开始 (即从 JDK5 开始)，volatile 变量的写-读可以实现线程之间的通信

从内存语义的角度来说，volatile 的写-读与锁的释放-获取有相同的内存效果：

- 「volatile 写」和「锁的释放」有相同的内存语义；「volatile 读」与「锁的获取」有相同的内存语义
- 「volatile 写」和「锁的释放」都把把修改同步到内存；「volatile 读」和「锁的获取」都会从内存中获取最新数据

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

这里 A 线程写一个 volatile 变量后，B 线程读同一个 volatile 变量

**<font color='red'>A 线程在写 volatile 变量之前所有可见的共享变量，在 B 线程读同一个 volatile 变量后，将立即变得对 B 线程可见</font>**

### <font color=#1FA774>volatile 写-读的内存语义</font>

**volatile 写的内存语义如下：**

- **<font color='red'>当写一个 volatile 变量时，JMM 会把该线程对应的本地内存中的共享变量值刷新到主内存</font>**

以上面示例程序 VolatileExample 为例，假设线程 A 首先执行`writer()`方法，随后线程 B 执行`reader()`方法，初始时两个线程的本地内存中的`flag`和`a`都是初始状态。下图是线程 A 执行 volatile 写后，共享变量的状态示意图:

![16](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221008/2147341665236854CxW2Oq16.svg)

如上图所示，线程 A 在写`flag`变量后，本地内存 A 中被线程 A 更新过的两个共享变量的值被刷新到主内存中。此时，本地内存 A 和主内存中的共享变量的值是一致的

**volatile 读的内存语义如下：**

- **<font color='red'>当读一个 volatile 变量时，JMM 会把该线程对应的本地内存置为无效，线程接下来将从主内存中读取共享变量</font>**

下面是线程 B 读同一个 volatile 变量后，共享变量的状态示意图：

![17](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221008/2148361665236916MswL8W17.svg)

如上图所示，在读`flag`变量后，本地内存 B 包含的值已经被置为无效。此时，线程 B 必须从主内存中读取共享变量。线程 B 的读取操作将导致本地内存 B 与主内存中的共享变量的值也变成一致的了

如果我们把 volatile 写和 volatile 读这两个步骤综合起来看的话，**在读线程 B 读一个 volatile 变量后，写线程 A 在写这个 volatile 变量之前所有可见的共享变量的值都将立即变得对读线程 B 可见**

下面对 volatile 写和 volatile 读的内存语义做个总结：

- 线程 A 写一个 volatile 变量，实质上是线程 A 向接下来将要读这个 volatile 变量的某个线程发出了 (其对共享变量所在修改的) 消息
- 线程 B 读一个 volatile 变量，实质上是线程 B 接收了之前某个线程发出的 (在写这个 volatile 变量之前对共享变量所做修改的) 消息
- 线程 A 写一个 volatile 变量，随后线程 B 读这个 volatile 变量，这个过程实质上是线程 A 通过主内存向线程 B 发送消息

### <font color=#1FA774>volatile 内存语义的实现</font>

现在让我们来看看 JMM 如何实现 volatile 写/读的内存语义

在 **[Java 内存模型](./Java内存模型.html)** 中提到过重排序分为编译器重排序和处理器重排序。为了实现 volatile 内存语义，JMM 会分别限制这两种类型的重排序类型。下面是 JMM 针对编译器制定的 volatile 重排序规则表：

|             | 普通读/写 | volatile 读 | volatile 写 |
| :---------: | :-------: | :---------: | :---------: |
|  普通读/写  |           |             |     NO      |
| volatile 读 |    NO     |     NO      |     NO      |
| volatile 写 |           |     NO      |     NO      |

**注：**NO 表示禁止重排序！

举例来说，第二行最后一个单元格的意思是：在程序顺序中，当第一个操作为普通变量的读或写时，如果第二个操作为 volatile 写，则编译器不能重排序这两个操作

从上表我们可以看出：

- 当第二个操作是 volatile 写时，不管第一个操作时什么，都不能重排序。这个规则**<font color='red'>确保 volatile 写之前的操作不会被编译器重排序到 volatile 写之后</font>**
- 当第一个操作是 volatile 读时，不管第二个操作是什么，都不能重排序。这个规则**<font color='red'>确保 volatile 读之后的操作不会被编译器重排序到 volatile 读之前</font>**
- 当第一个操作是 volatile 写，第二个操作是 volatile 读时，不能重排序

为了**实现 volatile 的内存语义**，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能，为此，JMM 采取**保守策略**。下面是基于保守策略的 JMM 内存屏障插入策略：

- 在每个 volatile 写操作的**前**面插入一个 StoreStore 屏障
- 在每个 volatile 写操作的**后**面插入一个 StoreLoad 屏障
- 在每个 volatile 读操作的**后**面插入一个 LoadLoad 屏障
- 在每个 volatile 读操作的**后**面插入一个 LoadStore 屏障

下面是保守策略下，volatile 写插入内存屏障后生成的指令序列示意图：

![5](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220909/2116501662729410M3Iwe65.svg)

上图中的 StoreStore 屏障可以保证在 volatile 写之前，其前面的所有普通写操作已经对任意处理器可见了。这是因为 StoreStore 屏障将保障上面所有的普通写在 volatile 写之前刷新到主内存

这里比较有意思的是 volatile 写后面的 StoreLoad 屏障。这个屏障的作用是避免 volatile 写与后面可能有的 volatile 读/写操作重排序

因为编译器常常无法准确判断在一个 volatile 写的后面，是否需要插入一个 StoreLoad 屏障 (比如，一个 volatile 写之后方法立即 return)。为了保证能正确实现 volatile 的内存语义，JMM 在这里采取了保守策略：在每个 volatile 写的后面或在每个 volatile 读的前面插入一个 StoreLoad 屏障

从整体执行效率的角度考虑，JMM 选择了在每个 volatile 写的后面插入一个 StoreLoad 屏障。因为 volatile 写-读内存语义的常见使用模式是：**一个写线程写 volatile 变量，多个读线程读同一个 volatile 变量**。当读线程的数量大大超过写线程时，选择在 volatile 写之后插入 StoreLoad 屏障将带来可观的执行效率的提升

从这里我们可以看到 JMM 在实现上的一个特点：**首先确保正确性，然后再去追求执行效率**

**<font color='red'>问题 1：为什么 volatile 写前面不用插入 LoadStore 屏障来禁止和普通读重排序？？</font>**

![22](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221009/1503111665298991hIgdux22.svg)

在 x86 CPU 中，只允许「Stores can be reordered after loads」。所以在 volatile 写前面没有插入 LoadStore 屏障

但如果在其他 CPU 下，可能就需要加，如：ARM CPU

**<font color='red'>问题 2：为什么要禁止 volatile 写和前面的普通写重排序？？</font>**

```java
class ReorderExample {
    int a = 0;
    volatile boolean flag = false;
    public void writer() {
        a = 1;                // 1 (普通写)
        flag = true;          // 2 (volatile 写)
    }
    public void reader() {
        if (flag) {           // 3
            int i = a * a;    // 4
            // other ...
        }
    }
}
```

这里假设有两个线程 A 和 B，A 首先执行`writer()`方法，随后 B 线程接着执行`reader()`方法

由于操作 1 和操作 2 没有数据依赖关系，编译器和处理器可以对这两个操作重排序，假设不禁止重排序，那么上述程序中的执行顺序可能是：

![8](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221007/2044001665146640PpJC6i8.svg)

如上图所示，操作 1 和操作 2 做了重排序。程序执行时，线程 A 首先写标记变量`flag`，随后线程 B 读这个变量。由于条件判断为真，线程 B 将读取变量 a。此时，变量 a 还根本没有被线程 A 写入，在这里多线程程序的语义被重排序破坏了！

**<font color='red'>问题 3：为什么要禁止 volatile 读和后面的普通读/写重排序？？</font>**

```java
class ReorderExample {
    volatile int v = 10;  // 可以理解为初始化好的某种资源 
    boolean flag = false; // 用于判断某种操作是否完成

    public void use(){
        int a = v;        // 1 可以理解为这里是在利用某种资源 (volatile 读)
        flag = true;      // 2 利用完volatile资源之后，用于判断操作完成 (普通写)
    }

    public void release() {
        while (flag) {
            v = 0;        // 可以理解为释放资源
        }
    }
}
```

这里假设有两个线程 A 和 B，A 首先执行`use()`方法，随后 B 线程接着执行`release()`方法

由于操作 1 和操作 2 没有数据依赖关系，编译器和处理器可以对这两个操作重排序，假设不禁止重排序，那么上述程序中的执行顺序可能是：

![19](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221009/1357141665295034igR5nI19.svg)

```java
class ReorderExample {
    int a = 0;
    volatile boolean flag = false;
    public void writer() {
        a = 1;                // 1 (普通写)
        flag = true;          // 2 (volatile 写)
    }
    public void reader() {
        if (flag) {           // 3 (volatile 读)
            int i = a * a;    // 4 (普通读)
            // other ...
        }
    }
}
```

![9](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221007/20440916651466491iFQwZ9.svg)

在程序中，操作 3 和操作 4 存在控制依赖关系。当代码中存在控制依赖性时，会影响指令序列执行的并行度。为此，编译器和处理器会采用猜测 (Speculation) 执行来克服控制相关性对并行度的影响。以处理器的猜测执行为例，执行线程 B 的处理器可以提前读取并计算`a * a`，然后把计算结果临时保存到一个名为重排序缓冲 (reorder buffer ROB) 的硬件缓存中。当接下来操作 3 的条件判断为真时，就把该计算结果写入变量`i`中

从图中我们可以看出，猜测执行实质上对操作 3 和 4 做了重排序。重排序在这里破坏了多线程程序的语义！

### <font color=#1FA774>JSR-133 为什么要增强 volatile 的内存语义</font>

在 JSR-133 之前的旧 Java 内存模型中，虽然不允许 volatile 变量之间重排序，但旧的 Java 内存模型允许 volatile 变量与普通变量重排序。在旧的内存模型中, VolatileExample 示例程序可能被重排序成下列时序来执行：

![23](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221009/1519191665299959mg3x0k23.svg)

在旧的内存模型中，当 1 和 2 之间没有数据依赖关系时，1 和 2 之间就可能被重排序 (3 和 4 类似)。其结果就是：读线程 B 执行 4 时，不一定能看到写线程 A 在执行 1 时对共享变量的修改

因此在旧的内存模型中，volatile 的写-读没有锁的释放-获所具有的内存语义。为了提供一种比锁更轻量级的线程之间通信的机制，JSR-133 专家组决定增强 volatile 的内存语义：严格限制编译器和处理器对 volatile 变量与普通变量的重排序，**<font color='red'>确保 volatile 的写-读和锁的释放-获取具有相同的内存语义</font>**。从编译器重排序规则和处理器内存屏障插入策略来看，只要 volatile 变量与普通变量之间的重排序可能会破坏 volatile 的内存语义，这种重排序就会被编译器重排序规则和处理器内存屏障插入策略禁止

由于 volatile 仅仅保证对单个 volatile 变量的读/写具有原子性，而锁的互斥执行的特性可以确保对整个临界区代码的执行具有原子性。**<font color='red'>在功能上，锁比 volatile 更强大；在可伸缩性和执行性能上，volatile 更有优势</font>**

### <font color=#1FA774>实例：双重校验锁</font>

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

### <font color=#1FA774>volatile 原子性？？</font>

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

因为本质上`race++`是读、写三次操作

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

- Java 并发编程实战
- 深入理解 Java 虚拟机
- 深入理解 Java 内存模型
- [内存屏障及其在-JVM 内的应用（上）](https://segmentfault.com/a/1190000022497646)
- [内存屏障及其在 JVM 内的应用（下）](https://segmentfault.com/a/1190000022508589)
- [关键字: volatile详解](https://pdai.tech/md/java/thread/java-thread-x-key-volatile.html)
- [The JSR-133 Cookbook for Compiler Writers](https://gee.cs.oswego.edu/dl/jmm/cookbook.html)
