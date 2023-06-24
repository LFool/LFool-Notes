# Java 线程池

### <font color=#1FA774>线程的创建</font>

在正式介绍线程池之前，先来聊聊创建线程有哪些方法！！

**方法一：**继承`Thread`类，重写`run()`方法

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("执行任务");
    }

    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        myThread.start();
    }
}
```

**方法二：**实现`Runnable`接口

如果研究过源码会发现方法一和方法二其实算一种，因为`Thread`实现了`Runnable`接口，方法一中重写的`run()`方法就是`Runnable`接口中抽象方法

```java
public static void main(String[] args) {
    new Thread(() -> {
        System.out.println("执行任务");
    }).start();
}
```

**方法三：**实现`Callable`接口，并结合`FutureTask`

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    FutureTask<String> task = new FutureTask<>(() -> {
        System.out.println("执行任务");
        return "返回点什么好呢！";
    });
    new Thread(task).start();
    System.out.println(task.get());  // 返回值
}
```

**方法四：**使用接下来要介绍的线程池！

### <font color=#1FA774>线程池引入</font>

大家肯定都多多少少听过「xxx池」，如：线程池、数据库连接池、HTTP 连接池等，它们都是一种基于**池化思想**来管理资源概念

如果没有线程池，那么我们在开发多线程程序时就需要随时随地的创建/销毁线程，而这样带来的开销也是极大的。所以线程池可以给我们带来的好处：

- **降低资源的消耗：**通过重复利用已创建的线程降低线程创建和销毁造成的开销
- **提高响应速度：**当任务到达时可以不需要等待创建线程而直接开始执行任务
- **提高线程的可管理性：**线程是稀缺资源，如果无限制地创建线程，不仅会消耗系统资源，还会降低系统稳定性，使用线程池可以进行统一分配、调优和监控

### <font color=#1FA774>Executor 框架介绍</font>

在传统的方式中，每创建一个线程时就绑定一个任务，也就是`run()`方法，当调用`start()`方法时就会执行任务，如下面代码所示：

```java
new Thread(() -> {
    System.out.println("任务");
}).start();
```

这样的局限性在于一个线程只能执行一个任务，当任务执行完后线程的使命也就结束了，接着就应该被销毁，这也是为什么需要频繁创建/销毁线程的原因！！

而在 Executor 框架中将**任务**和**任务的执行**解耦，也就意味着一个线程可以执行多个任务，我们只需要管理好一定数量的线程就可以实现复用，也避免了频繁的创建和销毁。这个框架分为三个部分：

- **任务 (Runnable/Callable)：**实现`Runnable`或`Callable`接口，更具体的是`run()`或`call()`方法
- **任务的执行 (Executor)：**执行实现`Runnable`或`Callable`接口的任务，主要包含两个接口、两个实现类，具体见下图
- **异步计算的结果 (Future)：**对于有返回值的任务，可以通过`Future`接口以及它的实现类`FutureTask`来接收。若没有返回值接收结果为`null`

![1](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230405/0553111680645191zEfhZk1.svg)

**<font color='red'>注意：</font>**线程池实现类`ThreadPoolExecutor`是`Executor`框架最核心的类，所以后文如果没有明确说明均为`ThreadPoolExecutor`

### <font color=#1FA774>线程池处理流程</font>

线程池的核心在于当一个任务提交后，如何去执行它呢？？

在介绍处理流程之前，先来看看四个概念：核心线程池、线程池、工作队列、饱和策略

- 核心线程池与线程池是包含的关系，在初始化线程池是会指定这两者的大小`corePoolSize, maximumPoolSize`
- 工作队列是一个 FIFO 的阻塞队列，用锁保证线程安全，它主要负责在核心线程池满了后存放无法立即分配线程的任务，在初始化线程池是会指定具体使用的阻塞队列，包括大小
- 饱和策略是在核心线程池、工作队列、线程池均满了后采取的策略

当一个线程池被创建后，默认情况下是没有线程的，只是初始化了一些参数，如上面介绍的。当一个任务被提交到线程池后，主要的流程为：

- 判断核心线程池中的工作线程数和 corePoolSize 的关系。若工作线程数 < corePoolSize，那么直接新建一个工作线程；否则尝试将任务加入任务队列中
- 判断任务队列中的任务数和队列大小的关系。如果任务队列中的任务数 < 队列大小，那么直接将任务加入到任务队列中；否则尝试在线程池 (非核心线程池) 中创建线程
- 判断工作线程数和 maximumPoolSize 的关系。若工作线程数 < maximumPoolSize，那么直接在线程池中创建；否则直接交给饱和策略

为了整个流程更加清楚，配一张流程图和执行示意图：

![2](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230405/0652521680648772zxvgOA2.svg)

### <font color=#1FA774>线程池状态</font>

**<font color='red'>注意：</font>**这里说的是线程池的状态，而不是线程的状态

线程池的运行状态并不是用户显示的设置，而是伴随着线程池的运行由内部来维护。线程池的内部由一个整型原子变量来维护两个值：运行状态 (runState) & 工作线程数量 (workerCount)

下面是`ThreadPoolExecutor`的部分源码：

```java
// ctl 包含了运行状态和工作线程数量两个信息，其中高 3 位表示运行状态，低 29 为表示工作线程数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;               // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;          // 00011111111111111111111111111111 (二进制)

private static final int RUNNING    = -1 << COUNT_BITS;               // 11100000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;               // 00000000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;               // 00100000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;               // 01000000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;               // 01100000000000000000000000000000

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }    // 取高 3 位
private static int workerCountOf(int c)  { return c & CAPACITY; }     // 取低 29 位
private static int ctlOf(int rs, int wc) { return rs | wc; }          // 合并两个变量
```

根据上面代码可以看出线程池一共分为 5 种运行状态：

- **RUNNING：**能接收新提交的任务，也可以处理任务队列中的任务
- **SHUTDOWN：**关闭状态，不再接收新提交的任务，但能处理任务队列中已保持的任务
- **STOP：**不能接收新提交的任务，也不能处理任务队列中的任务，会中断正在处理的任务
- **TIDYING：**所有任务已经终止，workerCount = 0
- **TERMINATED：**在`terminated()`方法执行后进入该状态

状态之间的转换如下图：

![3](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230405/0728491680650929h71XK53.svg)

**<font color='red'>技巧：</font>**由于状态是递增的，在源码中可以看到大量用`< 0`判断线程池是否处于运行，用`>= 0`判断线程池是否处于半死不活的状态

### <font color=#1FA774>阻塞队列</font>

阻塞队列用来存储没有分配线程的任务，类似于是一个缓冲区，也可以看作是生产者-消费者模型，任务的创建者作为生产者向队列中添加任务，任务的执行者作为消费者从队列中取出任务

使用不同的队列可以实现不一样的任务存取策略，下面介绍一些常见的阻塞队列：

- **ArrayBlockingQueue：**一个由数组实现的有界阻塞队列，在初始化时必须规定大小，遵循 FIFO。支持公平锁和非公平锁
- **LinkedBlockingDeque：**一个由链表实现的有界阻塞队列，默认长度为`Integer.MAX_VALUE`，使用默认长度有容量危险
- **PriorityBlockingQueue：**一个支持优先级的无界阻塞队列，会自动扩容，默认按照自然序排列，也可以自定义比较方式
- **DelayQueue：**一个支持延时获取元素的无界阻塞队列，使用`PriorityQueue`实现，在创建元素时可以指定多久才能从队列中获取当前元素
- **SynchronousQueue：**一个不存储元素的阻塞队列，每一个`put`操作必须等待一个`take`操作，否则不能继续添加元素
- **LinkedTransferQueue：**一个由链表实现的无界阻塞队列，相比于其它队列，可以尝试直接把生产者生产的任务交给消费者，而不需要入队
- **LinkedBlockingDeque：**一个由链表实现的双向阻塞队列，队列的头尾都可以添加或删除元素，多线程并发时可以最多将锁的竞争降低一半

### <font color=#1FA774>饱和策略</font>

如上文所讲，当核心线程池、任务队列、线程池均满了时，新提交的任务会直接交给饱和策略处理，一共有四种：

- **ThreadPoolExecutor.AbortPolicy：**丢弃任务并抛出异常，默认的饱和策略，可以及时通过异常发现问题，适用于一些关键性业务
- **ThreadPoolExecutor.DiscardPolicy：**丢弃任务但不抛出异常，无法及时发现问题，适用于一些无关紧要的业务
- **ThreadPoolExecutor.DiscardOldestPolicy：**丢弃队列最前面的任务，然后重新提交被拒绝的任务，实际场景中需要权衡老任务是否可以被丢弃
- **ThreadPoolExecutor.CallerRunsPolicy：**由提交任务的线程去执行该任务，这种策略没有放弃任何一个任务

### <font color=#1FA774>源码分析</font>

**<font color='red'>前提：</font>**线程池实现类`ThreadPoolExecutor`是`Executor`框架最核心的类，下面主要分析`ThreadPoolExecutor`类的源码

#### <font color=#9933FF>初始化</font>

`ThreadPoolExecutor`的构造函数一共四个，主要就是初始化一些参数，这里介绍其中最全的一个：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;                     // 核心线程池线程数量
    this.maximumPoolSize = maximumPoolSize;               // 线程池最大线程数
    this.workQueue = workQueue;                           // 任务队列
    this.keepAliveTime = unit.toNanos(keepAliveTime);     // 当线程数大于核心线程数时，多余空闲线程存活的最长时间
    this.threadFactory = threadFactory;                   // 线程工厂，用来创建线程，一般默认即可
    this.handler = handler;                               // 饱和策略
}
```

#### <font color=#9933FF>execute()</font>

当创建好线程池后就会调用`execute()`提交任务，源码的流程和上面介绍的 **[线程池处理流程](./Java线程池.html#线程池处理流程)** 完全吻合！！下面是对`execute()`的源码分析：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    
    int c = ctl.get();  // c 包含了线程池运行状态和工作线程数量
    // 如果「工作线程数量 < 核心线程池大小」，直接创建新的工作线程，优先让核心线程池线程饱和
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))  // 创建新的工作线程，true 表示在核心线程池中创建
            return;
        c = ctl.get();
    }
    // 到这里说明工作线程数量 >= 核心线程池大小，优先把任务添加到任务队列中
    // 线程池为运行状态，然后把任务添加到任务队列中
    if (isRunning(c) && workQueue.offer(command)) {
        // 成功加入任务队列
        int recheck = ctl.get();  // 重新检查状态
        // 线程池为非运行状态，然后把刚刚添加的任务从队列中删除
        if (! isRunning(recheck) && remove(command))
            reject(command);  // 交给饱和策略处理
        else if (workerCountOf(recheck) == 0)
            // 工作线程数量为 0，可能是因为 corePoolSize = 0，所以需要创建一个空线程，该线程算在线程池中，而不算在核心线程池，从参数 false 可以看出
            addWorker(null, false);
    }
    // 到这里说明工作线程数量 >= 核心线程池大小，且任务队列已满，尝试在线程池中创建线程
    else if (!addWorker(command, false))
        // 该任务无法被执行，交给饱和策略处理
        reject(command);
}
```

#### <font color=#9933FF>addWorker()</font>

从`execute()`的源码中可以看到调用了多次`addWorker()`方法，它是添加一个工作线程 (包括创建和启动线程两个步骤)。下面是对`addWorker()`的源码分析：

```java
// 返回 false 表示添加失败，true 表示添加成功 (创建并启动成功)
// core 为 true 表示在核心线程池中创建，为 false 表示在线程池中创建
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {  // 死循环
        int c = ctl.get();       // c 包含了线程池运行状态和工作线程数量
        int rs = runStateOf(c);  // 线程池运行状态

        // 这个判断较复杂，改写一波：if (rs >= SHUTDOWN && (rs != SHUTDOWN || firstTask != null || workQueue.isEmpty())
        // 要使这个判读不成立：要么线程池为运行状态；要么线程池为半死不活状态同时满足 (rs = SHUTDOWN && firstTask = null && !workQueue.isEmpty())
        // 总结：如果线程池在 SHUTDOWN 状态下，不会再接受新任务，在此前提下如果状态到了 STOP 或者传入任务不为 null 或者任务队列为 null 都不需要创建新的线程
        // 更形象一点：线程池已经停止或者正在停止且没有剩余任务需要执行就不会创建线程
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {  // 死循环
            int wc = workerCountOf(c);  // 工作线程数量
            if (wc >= CAPACITY ||       // CAPACITY = (1 << 29) - 1
                wc >= (core ? corePoolSize : maximumPoolSize))
                // 表示超过了最大限度，直接返回
                return false;
            if (compareAndIncrementWorkerCount(c))  // 表示可以创建工作线程，CAS 操作将 workerCount + 1
                break retry;  // 退出最外层死循环
            c = ctl.get();    // 重写读
            if (runStateOf(c) != rs)  // 表示状态被改变，回到最外层死循环开始重新循环
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;  // 工作线程是否启动
    boolean workerAdded = false;    // 工作线程是否添加
    Worker w = null;                // Worker 实现 Runnable 接口，表示一个工作线程，后文详细介绍该类
    try {
        w = new Worker(firstTask);  // 创建工作线程，从线程工厂中创建
        final Thread t = w.thread;  // 创建的线程
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;  // 可重入锁
            mainLock.lock();        // 获取锁
            try {

                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN || // 线程池处于运行状态
                    // firstTask = null 表示工作线程为 0，必须创建一个工作线程，否则无法执行任务
                    // SHUTDOWN 下可以执行任务队列中的任务，只是不能提交新的任务罢了
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // 检查线程是否已经启动
                        throw new IllegalThreadStateException();
                    workers.add(w);  // 添加到工作线程集合中，workers 是一个 HashSet
                    int s = workers.size();  // 工作线程数量
                    if (s > largestPoolSize) // largestPoolSize 记录历史峰值工作线程数。因为 maximumPoolSize 可以重新设置，所以峰值可能会增大
                        largestPoolSize = s;
                    workerAdded = true;      // 工作线程成功添加到线程集合中
                }
            } finally {
                mainLock.unlock();           // 释放锁
            }
            if (workerAdded) {
                t.start();                   // 启动线程
                workerStarted = true;        // 工作线程启动成功
            }
        }
    } finally {
        if (! workerStarted)     // 工作线程启动失败
            addWorkerFailed(w);  // 从 workers 删除线程，同时更新 workerCount
    }
    return workerStarted;
}
```

配上一张执行流程图：

<img src="https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230406/0231271680719487W8Ediz4.svg" alt="4" style="zoom:80%;" />

#### <font color=#9933FF>Worker</font>

从`addWorker()`的源码中可以看出线程池将任务`Runnable`封装成了一个`Worker`实例，下面来看看`Worker`的庐山真面目

```java
// 继承 AQS，实现 Runnable 接口
// Worker 被设计成一种不可重入锁，在线程销毁时用来判断线程是否在执行任务，后面详细介绍
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;
    Runnable firstTask;
    // 构造方法
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 从线程工厂中创建一个线程，传入的参数是 this，那么如果执行 thread.start() 将会执行 Worker 中的 run()
        this.thread = getThreadFactory().newThread(this);
    }
    public void run() {
    	runWorker(this);  // 线程复用的关键
    }
}
```

#### <font color=#9933FF>runWorker()</font>

`addWorker()`中会调用`t.start()`启动创建的线程，后续会执行`Worker`中的`run()`方法，而`run()`方法中调用`runWorker()`方法。下面来看看`runWorker()`的源码：

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;  // 任务
    w.firstTask = null;
    w.unlock(); // 由于 Worker 初始化时将 state 设置为 -1，为了禁止中断，这里需要解锁一次，将 state 设置为 0 允许中断
    boolean completedAbruptly = true;  // 记录线程是否因为用户异常退出，默认 true
    try {
        // getTask () 从任务队列中获取任务，由于使用阻塞队列，所以该循环可能会处于阻塞状态
        // firstTask 不为 null 或者可以从任务队列中获取新任务，表示有任务可以执行
        while (task != null || (task = getTask()) != null) {
            w.lock();  // 获取锁
            // 如果线程池正在停止 (由 RUNNING 或者 SHUTDOWN 向 STOP 变更)，那么需要确保当前线程处于中断
            // 否则，要确保当前线程不处于中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();  // 执行任务
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;         // 清空 task，否则 while 会死循环执行同一个 task
                w.completedTasks++;  // 记录当前线程完成的任务数
                w.unlock();          // 释放锁
            }
        }
        // 离开循环表示没有任务
        completedAbruptly = false;   // 表示线程是正常退出
    } finally {
        processWorkerExit(w, completedAbruptly);  // 线程退出
    }
}
```

#### <font color=#9933FF>getTask()</font>

在`runWorker()`中线程会不断尝试从任务队列中获取可执行任务，这就是通过调用`getTask()`实现，下面看看它的源码：

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {  // 死循环
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 线程池状态为 STOP 表示不能处理任务队列中的任务，会中断正在处理的任务；任务队列为 null 表示没有任务
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();  // -1
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;  // 是否允许超时
        
        // 条件一：wc > maximumPoolSize 表示通过 setMaximumPoolSize() 方法减少过线程池容量，现在工作线程过饱和，需要销毁一些线程
        // 条件二：timed && timedOut 表示线程命中了超时控制并且上一轮循环通过 poll() 拉取任务为 null，现在工作线程拿不到，需要销毁一些线程
        // 条件三：wc > 1 || workQueue.isEmpty() 表示工作线程 > 1 或者任务队列为空，说明满足上面两个条件之一的同时再满足本条件后可以销毁该线程
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :  // 超时拉取
                workQueue.take();                                      // 阻塞拉取
            if (r != null)    // 拿到了任务
                return r;
            timedOut = true;  // 超时拉取时没有拿到任务
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

#### <font color=#9933FF>processWorkerExit()</font>

在`runWorker()`中，如果线程获取不到任务时会执行`processWorkerExit()`，它主要负责终结当前工作线程，下面看看它的源码：

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 因为抛出用户异常导致线程终结，直接使工作线程数 -1 即可
    // 如果因为正常获取不到任务导致线程终结，工作线程数在 getTask() 中已经被减少过，此处不需要减少
    if (completedAbruptly)
        decrementWorkerCount();  // -1

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;  // 统计完成任务总数
        workers.remove(w);                       // 将工作线程从 workers 集合中去除，因为断开了引用链，GC 会回收它
    } finally {
        mainLock.unlock();
    }

    tryTerminate();  // 每次减少工作线程都会判断线程池状态是否应该变为 TERMINATED
    
    // 防止还有任务，但没工作线程的情况
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {  // RUNNING or SHUTDOWN
        if (!completedAbruptly) {
            // min 表示所需最少的工作线程数量
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;  // 保持最少有 1 个去处理任务队列中的任务
            if (workerCountOf(c) >= min)  // 实际工作线程数量 > 最少的工作线程数量
                return; // replacement not needed
        }
        // 执行到此处说明工作线程不够，需要添加一个新的空线程
        addWorker(null, false);
    }
}
```

到此为止，「向线程池提交任务 -> 创建工作线程 -> 拉取任务真正执行 -> 无任务时终结线程」这一整个流程就分析完了，给出一张完整的流程图：

![5](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230406/0401371680724897uF1DWF5.svg)

#### <font color=#9933FF>shutdown()</font>

为了补上前面挖的坑：**[Worker](./Java线程池.html#worker)** 类继承 AQS 实现不可重入锁可以用来判断线程是否在执行任务

任何时候都只允许一个线程获取该锁，且同一线程不可重入。`runWorker()`中可以看到线程执行任务时会上锁`w.lock()`，当我们可以可以获取该锁，说明线程空闲

调用`shutdown()`方法会使线程池处于 SHUTDOWN 状态，不再接收新提交的任务，但能处理任务队列中已保持的任务，所以该方法中需要中断空闲的线程，下面看看它的源码：

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();   // 中断工作线程
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 遍历集合
        for (Worker w : workers) {
            Thread t = w.thread;
            // w.tryLock() 尝试获取锁，如果成功获取表示该线程空闲
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();  // 中断线程
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

### <font color=#1FA774>Runnable & Callable & execute & submit 比较</font>

#### <font color=#9933FF>Runnable VS Callable</font>

Runnable 接口不会返回结果或抛出异常；Callable 可以返回结果和抛出异常

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

#### <font color=#9933FF>execute VS submit</font>

`execute()`方法用于提交不需要返回的任务，无法判断任务执行的成功与否

`submit()`方法用于提交需要返回值的任务，通过一个`Future`类型对象接收

```java
ExecutorService executorService = Executors.newCachedThreadPool();
Future<Integer> task = executorService.submit(() -> {
    System.out.println("test");
    return 123;
});
System.out.println(task.get());  // 获取线程返回值
```

### <font color=#1FA774>几种常见的内置线程池</font>

下面要介绍的常见内置线程池都是基于`ThreadPoolExecutor`或`ScheduledThreadPoolExecutor`创建的，只是提前设置好了一些参数让线程池具有某些特性，我们完全可以直接使用构造函数自己创建

#### <font color=#9933FF>FixedThreadPool</font>

FixedThreadPool 被称为可重用固定线程数的线程池

```java
// corePoolSize = maximumPoolSize = nThreads
// 任务队列：链表，有界阻塞队列，默认长度为 Integer.MAX_VALUE
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

不推荐使用的原因

- 核心线程池数量和线程池数量相等，而且任务队列默认长度为 Integer.MAX_VALUE
- 由于任务队列长度过大，如果任务量过大，会不断的加入到队列中导致 OOM

#### <font color=#9933FF>SingleThreadExecutor</font>

SingleThreadExecutor 是只有一个线程的线程池

```java
// corePoolSize = maximumPoolSize = 1
// 任务队列：链表，有界阻塞队列，默认长度为 Integer.MAX_VALUE
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

不推荐使用的原因

- 线程池允许的线程数量太少，如果任务量过大，会不断的加入到队列中导致 OOM

#### <font color=#9933FF>CachedThreadPool</font>

CachedThreadPool 是一个会根据需要创建新线程的线程池

```java
// corePoolSize = 0; maximumPoolSize = Integer.MAX_VALUE
// 任务队列：不存储元素的阻塞队列，每一个 put 操作必须等待一个 take 操作，否则不能继续添加元素
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
```

不推荐使用的原因

- 线程池允许的线程数量过多，如果任务量过大且添加任务速度大于执行速度，会不断的创建新线程，导致耗尽 CPU 和内存资源

#### <font color=#9933FF>ScheduledThreadPool</font>

ScheduledThreadPool 用来在给定的延迟后运行任务或者定期执行任务。这个在实际项目中基本不会被用到，也不推荐使用

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```

### <font color=#1FA774>实践出真知</font>

线程池面临的核心问题在于：**<font color='red'>如何配置线程池的参数</font>**，主要是 corePoolSize、maximumPoolSize、任务队列、饱和策略

**场景一：**由于没有预估好请求流量，导致最大核心线程数设置偏小，大量抛出`RejectedExecutionException`异常

**场景二：**由于任务队列设置过长，最大线程数失效，导致请求数量增加时大量任务堆积在队列中，任务执行时间过长

回到上面的问题：**如何配置线程池的参数**，这里给出一种解决方案：**动态调整参数**

### <font color=#1FA774>参考文章</font>

- **Java 并发编程的艺术**
- **[Java 线程池详解](https://javaguide.cn/java/concurrent/java-thread-pool-summary.html)**
- **[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)**
- **[硬核干货：4W字从源码上分析JUC线程池ThreadPoolExecutor的实现原理](https://www.throwx.cn/2020/08/23/java-concurrency-thread-pool-executor/)**
