# AbstractQueuedSynchronizer (AQS)

### <font color=#1FA774>Lock 接口</font>

锁一般用来同步多个线程访问共享资源，保证线程安全。锁的内存语义和 volatile 内存语义相同：

- 获取锁 <-> volatile 读：每次都从主内存中读取
- 释放锁 <-> volatile 写：写完后立刻刷新回主内存

这也是为什么 volatile 和锁都具有可见性的原因！！

我们经常使用 synchronized 关键字来充当锁，它可以锁一个对象或者一个类，但它是隐式地获取锁和释放锁，也就是当程序进入被 synchronized 修饰的同步代码块时获取锁，离开同步代码块时释放锁

本部分要介绍的 Lock 属于显示地获取锁和释放锁，也就是需要主动调用`lock()`获取锁，调用`unlock()`释放锁。Lock 接口具有 synchronized 关键字所没有的一些特性：

|        特性        |                             描述                             |
| :----------------: | :----------------------------------------------------------: |
| 尝试非阻塞地获取锁 | 如果线程没有成功获取锁，可以选择自旋等待一段时间，类似于轻量级锁 |
|  能被中断地获取锁  | 获取锁的线程可以响应中断，如果发生中断会抛出异常，同时释放锁 |
|     超时获取锁     |      在指定时间内获取锁，如果超时仍没有获取锁，直接返回      |

Lock 是一个接口，它定义了获取锁和释放锁的基本操作，API 如下：(推荐直接去看对应的源码)

|                           方法名称                           |                     描述                      |
| :----------------------------------------------------------: | :-------------------------------------------: |
|                        `void lock()`                         |                    获取锁                     |
|    `void lockInterruptibly() throws InterruptedException`    |           中断的获取锁，可响应中断            |
|                     `boolean tryLock()`                      | 非阻塞的获取锁，成功返回 true，失败返回 false |
| `boolean tryLock(long time, TimeUnit unit) throws InterruptedException` |                  超时获取锁                   |
|                       `void unlock()`                        |                    释放锁                     |
|                  `Condition newCondition()`                  |               获取等待通知组件                |

### <font color=#1FA774>队列同步器</font>

本篇文章的重点来咯！！

AbstractQueuedSynchronizer 根据字面意思：抽象地队列同步器，简称同步器，它是实现锁的关键，大部分同步组件也是基于 AQS 设计实现的

锁是面向使用者的，它定义了使用者与锁交互的接口，隐藏了实现细节；同步器是面向锁的实现者，简化了锁的实现方式，屏蔽了同步状态管理、线程排队、等待唤醒等底层操作。具体的结构如下：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230329/0214421680027282kRdHPP1.svg" alt="1" style="zoom:88%;" />

对于锁的使用者，它交互的对象的是上面介绍的 Lock 接口提供的方法；对于锁的开发者，它交互的对象是 AQS (同步器)，继承 JDK 提供的 AQS 实现自定义同步器，按需重写五个方法即可

- `protected boolean tryAcquire(int arg)`：独占式获取同步状态，查询并判断同步状态是否符合预期，然后通过 CAS 设置同步状态
- `protected boolean tryRelease(int arg)`：独占式释放同步状态，释放后其它等待获取同步状态的线程有机会获取同步状态
- `protected int tryAcquireShared(int arg)`：共享式获取同步状态，可多个线程同时获取同步状态，常见的有读线程，不会修改共享变量
- `protected boolean tryReleaseShared(int arg)`：共享式释放同步状态
- `protected boolean isHeldExclusively()`：当前同步器是否被线程独占

可以看到上面五个方法都是非抽象的，也就表示可以只重写其中部分方法，无须全部重写。一般要么重写独占式的两个方法，要么重写共享式的两个方法

介绍五个可重写方法的时候多次提到同步状态，它是一个 volatile 修饰的整型变量`state`，具有单操作对其它线程立即可见。AQS 提供了三个具有原子性的方法用来访问和修改`state`

```java
// 读
protected final int getState() {
    return state;
}
// 写
protected final void setState(int newState) {
    state = newState;
}
// CAS
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

从底层的设计可以看出实现锁的内存语义至少可以有两种方式：

- 利用 volatile 变量的写-读所具有的内存语义
- 利用 CAS 所附带的 volatile 读和 volatile 写的内存语义

现在来总结升华一波：对于锁开发者，只需要继承 AQS 类并重写所需的方法即可实现自定同步器，进而实现自定义锁，这个过程几乎不用涉及到操作系统底层，都已经封装好了

**锁开发者实现自定义锁几乎就是依赖同步状态，只需要实现同步状态的获取和释放即可**。可重入锁就是一个很好的例子，它就是通过控制`state`实现同一个线程可多次获取同一个锁

当一个线程尝试获取同步状态，会先判断是否为 0，不为 0 表示有其它线程获取了同步状态，然后再判断获取该同步状态的线程是否是当前线程，如果是，同步状态直接 +1，该线程获取到同步状态

释放同步状态也是同理，当线程释放同步状态会 -1，直至变为 0 才表示该线程彻底放弃了该同步状态，其它线程可以获取该同步状态

通过这样一个逻辑实现了可重入的功能，这些仅仅只是重写了`tryAcquire()`和`tryRelease()`方法，是不是对锁开发者很友好！！



最后再扩展一下 AQS 使用到的设计模式！！为什么只重写`tryAcquire()`方法就可以实现同步状态的获取？？这是因为 AQS 使用了模版方法设计模式

模版方法顾名思义就是已经设计好了一个方法的模版，即给出了每一个具体的步骤，可以自己实现部分功能定制化某个步骤以到达实现自定义该方法的效果

举个很常用的例子 🌰，Java 中的`Arrays.sort()`方法可以传入一个自定义的比较器实现不同效果的排序。它的底层原理就是使用了模版方法，设计好了排序所需的每个步骤，传入的自定义比较器定制化其中一个步骤，即：比较规则，最后实现了不同效果的排序，如：升序，降序，先根据 key 排序，key 相同再根据 value 排序等

这里结合模版方法浅析一下 AQS 获取同步状态的实现步骤：

```java
// 调用 lock() 获取锁，其实底层调用的是 acquire()
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&    // 自己重写的方法
        // AQS 已经帮忙实现好了 acquireQueued(), addWaiter(), selfInterrupt() 方法
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

### <font color=#1FA774>同步队列结构</font>

到目前为止我们介绍的内容还浮于表面，只是介绍了一下接口层的东西，并未深入到底层，比如 AQS 的结构？如何管理同步状态？未获取到同步状态的线程如何处理？排队？自旋？阻塞？如何等待唤醒线程？对于这些问题，我们一个一个来～

如果同步状态已经被线程 A 独占，那么线程 B 尝试获取失败时，会将线程 B 加入到一个同步队列中 (FIFO 双向队列)，队列中的每个节点就是一个线程，结构如下：

![2](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230329/0344531680032693Sdtvl72.svg)

因为它是一个 FIFO 的双向队列，所以主要操作是从队尾入队、从头部出队。当一个线程调用`tryAcquire()`获取同步状态失败时，会调用`compareAndSetTail(pred, node)`将节点插入到队尾

```java
// expect 是 tail 节点；update 是即将要插入的新节点。该方法的作用就是将 tail 指向新的 node 节点
// compareAndSwapObject 保证了更新尾节点操作的原子性
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}
```

![3](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230329/1916101680088570PUgvHW3.svg)

同步队列遵循 FIFO，首节点是成功获取同步状态的线程，首节点的线程释放同步状态时，会唤醒后继节点，而后继节点如果成功获取同步状态就会将自己设置为首节点，原首节点出队

由于设置新首节点是由成功获取同步状态的线程完成的，而成功获取同步状态的线程只可能有一个，即原首节点的后续节点，所以设置首节点不需要通过 CAS 来保证原子性

```java
// node 是原首节点的后续节点，即新首节点
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

![4](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230329/20010316800912635BZRRW4.svg)

这里还要提一个细节：

```java
setHead(node);
// p 是原首节点，会把 p.next 置为 null，防止 p 和引用链上的对象相连，有助于 GC
p.next = null; // help GC
```

### <font color=#1FA774>节点结构</font>

上面说过同步队列中的每一个节点都代表一个线程，那么节点中到底有什么呢？先来看看它的结构：(建议直接看 AbstractQueuedSynchronizer 源码对应的部分)

```java
static final class Node {
    static final Node SHARED = new Node();   // 共享式节点
    static final Node EXCLUSIVE = null;      // 独占式节点
    
    // 节点的状态，后面详细说明
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
    
    // 记录节点的状态
    volatile int waitStatus;
    
    // prev 和 next 指针
    volatile Node prev;
    volatile Node next;
    // 节点对应的线程
    volatile Thread thread;
    // 等待队列中的后续节点
    Node nextWaiter;
    // 是否为共享式
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
    // 前继节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    // 三个构造函数
    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

其它的没什么好说，最主要的就是节点的状态，有 5 种：

- **CANCELLED (1)：**表示当前节点已取消调度。当超时或被中断 (响应中断的情况下)，会触发变更为此状态，进入该状态后的节点不会再变化
- **SIGNAL (-1)：**表示后继节点在等待当前节点唤醒。后继节点入队时会将前继节点的状态更新为 SIGNAL，如果前继节点释放了同步状态，将会唤醒后继节点继续运行
- **CONDITION (-2)：**表示节点等待在 Condition 上，当其它线程调用了 Condition 的`signal()`方法后，CONDITION 状态的节点将从等待队列中转移到同步队列中，加入到对同步状态的获取中
- **PROPAGATE (-3)：**共享模式下，前继节点不仅会唤醒后继节点，同时也可能会唤醒后继的后继节点，以此类推
- **(0)：**新节点入队时的默认初始状态

**<font color='red'>总结：</font>负值表示节点处于有效的等待状态，而正值表示节点已被取消。所以源码中很多地方都用到`< 0 or 0 <`来判断节点的状态是否正常**

### <font color=#1FA774>独占式同步状态获取与释放</font>

首先解释一下独占的意思：**同步状态任何时候都只允许一个线程获取，只有当线程释放同步状态后，其它线程才可以尝试获取**

如果使用`synchronized`，那么当线程竞争失败后会进入阻塞状态；如果使用 AQS，那么当线程竞争失败后会进入等待状态，因为底层是调用`LockSupport.park()`

#### <font color=#9933FF>获取同步状态</font>

通过上面对 AQS 结构的介绍，可以知道获取同步状态其实就是调用`acquire(int arg)`方法

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&    // 自己重写的方法
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

函数流程如下：

- `tryAcquire()`尝试直接去获取同步状态，如果成功直接返回 (这里体现了非公平锁)。这个方法需要被重写
- `addWaiter()`将该线程加入到同步队列的队尾，并标记为独占模式
- `acquireQueued()`使线程以「死循环」的方式获取同步状态，如果获取不到则阻塞线程，只能依赖前继节点唤醒自己，一直到成功获取为止。如果整个过程该线程被中断过，则返回 true，否则 false
- `selfInterrupt()`如果线程处于等待状态时被中断过，它是不会响应的，只有成功获取同步状态后再将中断补上

下面直接上这些函数的源码，具体细节看注释：

```java
// 将线程加入到同步队列的队尾，enq() 会保证一定成功
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);  // 创建当前线程对应的节点
    Node pred = tail;     // 记录原尾节点
    if (pred != null) {   // 表示同步队列已经初始化
        node.prev = pred; // 当前节点的 prev 指向原尾节点
        if (compareAndSetTail(pred, node)) {  // 等价于 tail = node，具有原子性
            pred.next = node;  // 原尾节点的 next 指向当前节点
            return node;       // 返回当前节点
        }
    }
    // 没有成功插入队尾
    enq(node);   // 保证一定成功插入队尾，具体见下方
    return node; // 返回当前节点
}
// node 表示当前要插入的节点
private Node enq(final Node node) {
    // 死循环：CAS 自旋，只有成功插入才会退出循环
    for (;;) {
        Node t = tail;   // 记录原尾节点
        if (t == null) { // 表示同步队列未初始化，必须先执行初始化
            if (compareAndSetHead(new Node()))  // 创建一个空节点作为首节点，表示正在访问同步状态的线程，即已经成功获取同步状态
                tail = head;  // tail 也要指向它，但注意此时并没有退出循环，还会再循环一次！！
        } else {
            node.prev = t;  // 当前节点的 prev 指向原尾节点
            if (compareAndSetTail(t, node)) {  // 等价于 tail = node，具有原子性，后面的同 addWaiter()
                t.next = node;
                return t;
            }
        }
    }
}
// 线程执行到该方法表示没有成功获取同步状态，且该线程对应的节点已经加入到同步队列的队尾
// 此时该线程进入等待休息，直至前继节点唤醒自己，然后重新尝试获取同步状态，成功获取后就可以执行后续访问共享资源的流程
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;  // 出队时是否失败
    try {
        boolean interrupted = false;  // 等待过程中有无被其它线程中断，记录下来等唤醒后再处理，处于等待状态时是不会响应中断的
        // 死循环，入队后只有成功获取到资源或者超时抛出异常才会退出循环
        for (;;) {
            final Node p = node.predecessor();   // 当前节点的前继节点，只有前继节点是首节点才能尝试获取同步状态 ，其它节点都处于等待状态，这也体现了 FIFO 的特点
            if (p == head && tryAcquire(arg)) {  // 前继节点是首节点，然后尝试获取同步状态 -> 通过调用 tryAcquire()
                setHead(node);  // 设置头节点，可见上方介绍同步队列结构
                p.next = null;  // help GC
                failed = false; // 成功获取同步状态，failed 置为 false
                return interrupted;  // 返回中断标记，可以判断等待状态中有无被中断
            }
            // 表示获取同步状态失败，可以处于等待状态，等待前继节点把自己唤醒
            if (shouldParkAfterFailedAcquire(p, node) &&  // 判断是否应该等待
                parkAndCheckInterrupt())  // 调用 park() 会使当前线程处于等待
            interrupted = true;   // 记录有无被中断
        }
    } finally {  // finally 代码块一定会被执行
        if (failed)  // 如果等待过程中没有成功获取同步状态 (如：超时、可中断的情况下被中断)，那么直接取消节点在队列中等待，出队！！
            cancelAcquire(node);
    }
}
// pred 表示前继节点；node 表示当前节点
// 返回节点是否应该等待，主要是根据前继节点的状态，如果是 SIGNAL 则应该等待；否则应该先置为 SIGNAL
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;  // 前继节点的状态
    if (ws == Node.SIGNAL)     // 该状态表示如果前继节点释放了同步状态就会唤醒自己的后继节点
        return true;
    if (ws > 0) {  // 表示前继节点被取消调度，应该出队
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);  // 将前面应该出队的节点一次性全部出队
        pred.next = node;  // 接上 node
    } else {  // ws < 0 但不等于 SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);  // 将前继节点的状态置为 SIGNAL (原子性)
    }
    return false;
}
// 使线程等待，返回是否被中断过
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);        // 该方法使线程处于等待状态
    return Thread.interrupted();   // 判断当前线程是否被中断过，如果中断过会将中断标志清除 (The interrupted status of the thread is cleared by this method)
}
// 取消节点在队列中等待
private void cancelAcquire(Node node) {
    // 直接忽略
    if (node == null)
        return;
    node.thread = null;  // 断开引用

    Node pred = node.prev;       // 前继节点
    while (pred.waitStatus > 0)  // 以当前节点为起点，向前寻找所有被取消调度的节点，一起出队
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    node.waitStatus = Node.CANCELLED;  // 设置当前节点状态

    if (node == tail && compareAndSetTail(node, pred)) {  // 设置新尾节点
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        // pred 表示前继节点，设置状态为 SIGNAL
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            // 后继节点
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);  // 前继节点和后继节点接起来
        } else {
            // 表示 node 前面没有节点了，所以取消 node 后需要唤醒后继节点
            unparkSuccessor(node);  // 唤醒当前节点的后继节点
        }

        node.next = node; // help GC
    }
}
```

如果一个线程的中断标志为 true，那么再调用`LockSupport.park()`将不会使线程等待，具体原因可见 **[park()/unpark()](./interrupt-LockSupport-park-unpark.html#font-color1fa774parkunparkfont)**，而`Thread.interrupted()`就变成了关键，它可以清除中断标志

这里来一个**小结：**

- 同步队列中只有首节点成功获取同步资源
- 除首节点的其它节点均处于等待状态，没有无时无刻自旋尝试获取同步状态
- 只有首节点的后继节点在首节点释放资源后唤醒自己才能尝试获取同步状态，否则处于等待
- 如果线程等待过程中被中断是不响应的，当自己获取同步状态后再进行自我中断`selfInterrupt()`补上

下面给出独占式同步状态获取的流程，也是调用`acquire(int arg)`方法的流程：

![5](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230330/0153091680112389BQE9EW5.svg)

#### <font color=#9933FF>释放同步状态</font>

终于把最麻烦的获取部分整完了，剩下的内容就轻松些许了～～

通过上面对 AQS 结构的介绍，可以知道释放同步状态其实就是调用`release(int arg)`方法

```java
// 
public final boolean release(int arg) {
    if (tryRelease(arg)) {      // 自己重写的方法
        Node h = head;          // 记录首节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 唤醒后继节点
        return true;
    }
    return false;
}
```

函数流程如下：

- `tryRelease()`尝试去释放同步状态，如果成功直接返回。这个方法需要被重写
- `unparkSuccessor()`唤醒后继节点

下面直接上这些函数的源码，具体细节看注释：

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)  // 将当前节点状态置 0，允许失败
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;  // 后继节点
    if (s == null || s.waitStatus > 0) {  // 表示后续节点不需要被唤醒 -> null 或者取消了调度
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)  // 从后向前找，因为 prev 链保证了强一致性
            if (t.waitStatus <= 0)  // 表示节点有效
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);  // 唤醒
}
```

### <font color=#1FA774>共享式同步状态获取与释放</font>

共享式和独占式最大的区别就在于：**共享式允许多个线程获取一个同步状态**。最典型的例子就是对一个文件进行读操作，因为不会改变共享资源，所以可以允许多个线程同时获取一个同步状态

![6](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230330/02433216801154121Ta7Jk6.svg)

#### <font color=#9933FF>获取同步状态</font>

获取同步状态是调用`acquireShared(int arg)`方法

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)    // 自己重写的方法
        doAcquireShared(arg);         // 将线程加入同步队列，直到成功获取同步状态为止
}
```

函数流程如下：

- `tryAcquireShared()`的返回值为 int，当 >= 0 时表示成功获取到同步状态，具体值表示剩余同步状态的数量，即还可以获取的线程数量。若成功直接返回
- `doAcquireShared()`将线程加入同步队列，直到成功获取同步状态为止

下面直接上这些函数的源码，具体细节看注释：

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);      // 创建当前线程对应的节点 (共享模式)
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();     // 当前节点的前继节点
            if (p == head) {                       // 前继节点是首节点
                int r = tryAcquireShared(arg);     // 尝试获取同步状态
                if (r >= 0) {                      // >= 0 表示成功获取
                    setHeadAndPropagate(node, r);  // 将 head 指向自己，如果还有剩余资源唤醒后面线程
                    p.next = null; // help GC
                    if (interrupted)               // 如果等待过程中被中断过，此处补上
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 后面和独占式相同
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
// propagate 表示剩余资源
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node); // 将 node 设置为首节点
    // 如果还有甚于资源继续唤醒后继节点
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();       // 唤醒后续节点
    }
}
```

#### <font color=#9933FF>释放同步状态</font>

释放同步状态是调用`doReleaseShared()`方法

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {    // 自己重写的方法，尝试释放同步状态
        doReleaseShared();          // 唤醒后续节点
        return true;
    }
    return false;
}
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))      // 如果设置失败会直接 continue 然后继续循环
                    continue;            // loop to recheck cases
                unparkSuccessor(h);      // 唤醒后续节点
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))  // 如果设置失败会直接 continue 然后继续循环
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

### <font color=#1FA774>参考文章</font>

- **Java 并发编程的艺术**

- **[AQS 详解](https://javaguide.cn/java/concurrent/aqs.html)**

- **[Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)**

- **[AQS深入理解 doReleaseShared源码分析 JDK8](https://blog.csdn.net/anlian523/article/details/106319538)**

- **[从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)**
