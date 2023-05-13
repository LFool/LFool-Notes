# interrupt()中断 && LockSupport.park()/unpark()

### <font color=#1FA774>中断状态</font>

Thread 对象 native 实现中有一个标识位属性代表线程的**中断状态**，可以认为它是一个 Boolean 类型的变量，初始值为 false

通过调用一个线程的`isInterrupted()`方法可以判断该线程是否处于中断状态

通过调用一个线程的`interrupt()`方法将该线程中断，该操作只是将该线程的中断标志置为 true，至于线程以何种动作处理该中断就要看线程自己

- 如果线程因`sleep(), wait(), join()`方法处于等待状态，那么线程会定时检查中断标志是否为 true，如果为 true 则会在调用方法处抛出 InterruptedException 异常并清除中断标志重新设置为 false。抛出异常是为了唤醒线程
- 如果线程正在运行、竞争锁，那么是不可中断的，直接忽略

```java
public static void main(String[] args) throws InterruptedException {
    Thread mainThread = Thread.currentThread();
    Thread sleepThread = new Thread(() -> {
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }, "SleepThread");
    Thread waitThread = new Thread(() -> {
        try {
            Thread.currentThread().wait();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }, "WaitThread");
    Thread joinThread = new Thread(() -> {
        try {
            mainThread.join();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }, "JoinThread");
    sleepThread.start();
    waitThread.start();
    joinThread.start();
    System.out.println("中断前标志位 ...");
    System.out.println("Sleep Thread isInterrupted: " + sleepThread.isInterrupted());
    System.out.println("Wait Thread isInterrupted: " + waitThread.isInterrupted());
    System.out.println("Join Thread isInterrupted: " + joinThread.isInterrupted());
    // 中断
    sleepThread.interrupt();
    waitThread.interrupt();
    joinThread.interrupt();
    Thread.sleep(2000);
    System.out.println("中断后标志位 ...");
    System.out.println("Sleep Thread isInterrupted: " + sleepThread.isInterrupted());
    System.out.println("Wait Thread isInterrupted: " + waitThread.isInterrupted());
    System.out.println("Join Thread isInterrupted: " + joinThread.isInterrupted());
}
// result
中断前标志位 ...
Sleep Thread isInterrupted: false
Wait Thread isInterrupted: false
Join Thread isInterrupted: false
中断后标志位 ...
Sleep Thread isInterrupted: false
Wait Thread isInterrupted: false
Join Thread isInterrupted: false
```

### <font color=#1FA774>等待状态</font>

当我们调用`Object.wait(), Object.join(), LockSupport.park()`方法时，会使运行状态的线程变为等待状态

当我们调用`Object.notify(), Object.notifyAll(), LockSupport.unpark()`方法时，会使等待状态的线程变为运行状态

```java
public static void main(String[] args) {
    Thread consume = new Thread(() -> {
        for (int i = 0; i < 10; i++) {
            LockSupport.park();  // 等待
            System.out.println("消费者线程: " + i);
        }
    });
    Thread product = new Thread(() -> {
        for (int i = 0; i < 10; i++) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            LockSupport.unpark(consume);  // 唤醒
        }
    });
    consume.start();
    product.start();
}
// result (每隔 1s 输出 1 次)
消费者线程: 0
消费者线程: 1
消费者线程: 2
消费者线程: 3
消费者线程: 4
消费者线程: 5
消费者线程: 6
消费者线程: 7
消费者线程: 8
消费者线程: 9
```

### <font color=#1FA774>park()/unpark()</font>

前面说过，中断可以唤醒处于等待状态的线程，而调用`LockSupport.park()`正好可以使线程处于等待状态，那么是否可以使用`interrupt()`和`LockSupport.park()`实现无限次的唤醒和等待呢？

```java
public static void main(String[] args) {
    Thread consume = new Thread(() -> {
        for (int i = 0; i < 10; i++) {
            LockSupport.park();  // 等待
            System.out.println("消费者线程: " + i);
        }
    });
    Thread product = new Thread(() -> {
        consume.interrupt();     // 中断
    });
    consume.start();
    product.start();
}
// result
消费者线程: 0
消费者线程: 1
消费者线程: 2
消费者线程: 3
消费者线程: 4
消费者线程: 5
消费者线程: 6
消费者线程: 7
消费者线程: 8
消费者线程: 9
```

现在就可以看出很明显的问题，我们只中断了一次，为什么会输出 10 个呢？？不应该只输出 1 个就继续等待了吗？

我们先来仔细介绍一下`park()/unpark()`通信机制，它们俩之间是通过「许可 (permit)」来通信，是一个整型变量，它的数量是不允许叠加的，只有 0 和 1 两种状态，0 表示没有许可，1 表示有许可

调用`unpark()`会将许可 +1，如果已经是 1 就不变；调用 `park()` 会将许可 -1，如果已经是 0 就不变，但是会等待新的许可来才能被唤醒

它们俩有一个独特的性质：**<font color='red'>不需要管调用的先后顺序，可以先调用`unpark()`获得一个许可，后面再调用`park()`使用掉这个许可</font>**

**<font color='red'>注意：</font>**如果先调用`unpart()`，必须保证线程已经启动，也就是执行了`start()`方法，否则没有效果。详情可见 JDK 注释：

- This operation is not guaranteed to have any effect at all if the given thread has not been started.

`park()`是调用`UNSAFE.park(false, 0L)`方法，而`UNSAFE.park(false, 0L)`是一个本地方法，在 **[os_posix.cpp](https://hg.openjdk.org/jdk/jdk/file/1871c5d07caf/src/hotspot/os/posix/os_posix.cpp#l1955)** 中。代码太长，看不懂？？？直接上简化版的伪代码：

```cpp
park() {
    if (permit > 0) {
        permit = 0;
        return;
    }

    if (中断状态 == true) {
        return;
    }

    阻塞当前线程;  // 将来会从这里被唤醒

    if (permit > 0) {
        permit = 0;
    }
}
```

当 permit = 0 时表示线程处于等待状态，当 permit = 1 时表示线程处于唤醒状态，所以`park()/unpark()`就是 permit 处于 0 和 1 之间的变化

可以看出只要「permit = 1」或者「中断状态 = true」，那么执行`park()`就不能够阻塞线程，这也解释了为什么上面只调用了一次中断，后续就不会因`park()`而等待

同理`unpark()`简化版的伪代码如下：

```cpp
unpark(Thread thread) {
    if (permit < 1) {
        permit = 1;
        if (thread处于阻塞状态)
            唤醒线程thread;
    }
}
```

### <font color=#1FA774>interrupt() && unpark()</font>

简化版的伪代码如下：

```cpp
interrupt() {
    if (中断状态 == false) {
        中断状态 = true;
    }
    unpark(this);    // 注意这是 Thread 的成员方法，所以可以通过 this 获得 Thread 对象
}
```

`interrupt()`会设置中断状态为 true，同时调用一次`unpark()`，将 permit 设置为 1

### <font color=#1FA774>interrupt() && sleep()</font>

简化版的伪代码如下：

```cpp
sleep(){ //这里忽略了休眠时间，假设时间是大于 0 即可
    if (中断状态 == true) {
        中断状态 = false;
        throw new InterruptedException();
    }
    
    线程开始休眠;

    if (中断状态 == true) {
        中断状态 = false;
        throw new InterruptedException();
    }
}
```

`sleep()`会去检测中断状态，如果中断状态为 true，就消耗掉它 (置为 false) 并抛出异常

**注意：**`wait(), join()`同`sleep()`

### <font color=#1FA774>参考文章</font>

- **[interrupt()中断对LockSupport.park()的影响](https://blog.csdn.net/anlian523/article/details/106752414)**

- **[Java的LockSupport.park()实现分析](https://blog.csdn.net/hengyunabc/article/details/28126139)**

- **[Java之Interrupt解析](https://juejin.cn/post/6844904094474174478)**

- **[java并发编程---Thread.interrupted方法对LockSupport.park()的影响](https://blog.csdn.net/e891377/article/details/104707412)**
