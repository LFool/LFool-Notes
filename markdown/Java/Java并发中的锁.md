# Java 并发中的「锁」

### <font color=#1FA774>自旋锁 VS 自适应自旋锁</font>

如果上下文切换频繁，那么开销也会变大。有时候可能同步代码块比较简单，执行时间小于将线程挂起再唤醒的时间，所以这个时候选择阻塞线程等同步代码块执行完成后再唤醒线程将得不偿失！！

我们可以让当前线程自旋，也就是 CAS 里面的失败不断重试的过程，直到成功获得锁为止，流程图如下：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230323/0237021679510222MhN41M1.svg" alt="1" style="zoom:88%;" />

**<font color='red'>缺点：</font>**虽然自旋可以减少上下文切换开销，但毕竟自旋过程始终需要占用 CPU 资源。如果锁占用时间很长，导致自旋时间过长就会白白浪费很多 CPU 资源，但对于锁占用时间短的场景效果会非常好

针对于自旋锁的局限，提出了**<font color='red'>自适应自旋锁</font>**。它不会傻瓜式无脑的永远自旋等待锁的释放，而是自旋一定次数后 (默认值 10 次)，如果依旧没有成功获得锁，那么就直接挂起该线程，后续过程和非自旋锁一致

### <font color=#1FA774>Mark Word</font>

**更多对象头的内容可见 [对象内存布局](./对象的创建.html#对象内存布局)**

Java 中的每一个对象都可以作为锁。对象头中的 Mark Word 记录了对象运行时数据，32 bit 计算机的 Mark Word 如下图所示：

![1](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221113/2208461668348526wCfnAt9.svg)

64 bit 计算机的 Mark Word 如下图所示：(本文主要针对 64 bit)

![11](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230324/1458171679641097hzGGdR11.svg)

### <font color=#1FA774>无锁</font>

本部分要介绍的无锁就对应「未锁定」的状态，标志位 01，偏向模式 0

既然是无锁，那怎么保证多线程情况下共享变量的一致性呢？？很显然需要用到 **[CAS](./CAS.html)**，即：每次访问都乐观的认为存在线程竞争锁的概率很小，如果更新时出现冲突则重试

它有优势，适用于并发量不大，读操作较多的场景；但它也有劣势，单核 CPU 或并发量很大的系统会导致自旋重试的时间过长，得不偿失

### <font color=#1FA774>偏向锁</font>

有研究发现大多情况下锁不仅不存在多线程竞争，而且总是由同一个线程多次获得。在这种场景下，如果一个线程频繁进入同步块和退出同步块都使用 CAS 操作来加锁和解锁，势必会降低性能

所以就出现了偏向锁，对应「可偏向」的状态，标志位 01，偏向模式 1，**它适用于只有一个线程访问同步代码块的场景**

偏向锁假定将来只有第一个访问同步块的线程会使用锁，不会有其它任何线程来申请锁。所以只需要在 Mark Word 中使用 CAS 记录偏向线程 ID，如果记录成功，则偏向锁获取成功，以后该线程只需要判断 Mark Word 中的偏向线程 ID 即可零成本获得锁；如果记录失败，表示有其它线程竞争，膨胀为轻量级锁

偏向锁使用了等到竞争出现才释放锁的机制，所以当线程 A 获得了偏向锁，就算线程 A 退出了同步块也不会释放偏向锁，Mark Word 中的偏向线程 ID 依旧指向线程 A。当线程 B 尝试竞争偏向锁时，持有偏向锁的线程 A 才会释放锁，而且锁的撤销必须等待 **[全局安全点](./HotSpot的算法细节实现.html#安全点)**，此时处于能保障一致性的快照中。根据持有偏向锁的线程 A 的状态有两种情况：

- 线程 A 死亡或者线程 A 已经退出同步代码块，此时会直接撤销偏向锁，变成无锁状态，以供竞争锁的线程 B 获得锁
- 线程 A 还处于同步代码块中，撤销偏向锁后升级为轻量级锁。**注意：当前线程依旧是偏向锁，其它线程再获得时才变成轻量级锁**

可以看出偏向锁就是完全假定大多数情况下锁不仅不存在多线程竞争，而且总是由同一个线程多次获得。如果存在同一时间段内两个线程竞争锁，就会直接膨胀为轻量级锁

这样做的好处是以后同一个线程多次进入退出同步代码块时只需要判断 Mark Word 中的偏向线程 ID，不需要使用 CAS 操作来加锁解锁。前者只需要一次 CAS 原子指令，后者需要依赖多次 CAS 原子指令

#### <font color=#9933FF>小测试</font>

**<font color='red'>注意：</font>**在虚拟机启动时对偏向锁有延迟，所以要么休眠几秒钟再创建对象 (一定要在创建对象之前休眠)，要么添加参数`-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0`

```java
public class TestLock {
    // 共享变量 obj
    public static Object obj;
    public static void main(String[] args) {
        
        obj = new Object();
        // 加锁之前，输出对象头中的内容
        System.out.println("before lock");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        // 访问同步代码块
        sync();
        // 加锁之后，输出对象头中的内容
        System.out.println("after lock");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
    private static void sync() {
        synchronized (obj) {
            System.out.println("同步代码块执行中 ...");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
    }
}

// -------------- 输出 --------------
before lock
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)                                  // 可偏向的状态
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

同步代码块执行中 ...
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000150009005 (biased: 0x0000000000540024; epoch: 0; age: 0)     // 偏向锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

after lock
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000150009005 (biased: 0x0000000000540024; epoch: 0; age: 0)     // 偏向锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

**<font color='red'>解释：</font>**在加锁前 Mark Word 标志位 01，偏向模式 1，表示一种可偏向的状态；在执行同步代码块的过程中 Mark Word 标志位 01，偏向模式 1，表示偏向锁；在执行完同步代码块后 Mark Word 标志位 01，偏向模式 1，表示线程不会主动释放偏向锁，而是等到有其它线程竞争时才会释放

下面来模拟一波偏向锁升级为轻量级锁的过程！！

```java
public class TestLock {
    public static Object obj;
    public static void main(String[] args) throws InterruptedException {
        obj = new Object();
        System.out.println("before lock");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());

        sync("主线程");

        Thread.sleep(1000);

        // 子线程启动，此时主线程依旧持有锁，所以会升级为轻量级锁
        Thread thread = new Thread(() -> {
            try {
                sync("子线程");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        thread.start();
        thread.join();

        System.out.println("after lock");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
    private static void sync(String msg) throws InterruptedException {
        synchronized (obj) {
            System.out.println(msg + "：同步代码块执行中 ...");
            // 此时没有线程在竞争锁
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
            Thread.sleep(5000);  // 休眠 5s
            // 此时有线程在竞争锁
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
    }


// -------------- 输出 --------------
before lock
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)                                  // 可偏向的状态
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

// 主线程执行同步代码块
主线程：同步代码块执行中 ...
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000014400e005 (biased: 0x0000000000510038; epoch: 0; age: 0)     // 偏向锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000014400e005 (biased: 0x0000000000510038; epoch: 0; age: 0)     // 偏向锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

// 子线程执行同步代码块
子线程：同步代码块执行中 ...
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000016ee8a900 (thin lock: 0x000000016ee8a900)                   // 轻量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000016ee8a900 (thin lock: 0x000000016ee8a900)                   // 轻量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

after lock
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)                            // 无锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

**<font color='red'>解释：</font>**主线程获得的是偏向锁，同步代码块还未执行完时子线程来竞争锁，此时锁升级。主线程持有的依旧保持为偏向锁，但等主线程退出同步块后子线程获得的是轻量级锁。后续其它线程再次访问这个同步块时获得的都是轻量级锁，因为锁无法降级 (偏向锁 -> 轻量级锁 -> 重量级锁)

对象的 HashCode 会延迟到调用`hashCode()`方法时，当一个对象调用了`hashCode()`方法，那么该对象不能再成为偏向锁，只能成为轻量级锁或重量级锁。因为如果变成偏向锁，Mark Word 中原来的 HashCode 字段将会丢失。轻量级锁或重量级锁会先拷贝一份 Mark Word 到线程栈帧的锁记录中，当解锁时会恢复原 Mark Word

```java
public class TestLock {
    public static Object obj;
    public static void main(String[] args) throws InterruptedException {
        obj = new Object();
        obj.hashCode();
        sync();
    }
    private static void sync() throws InterruptedException {
        synchronized (obj) {
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
    }
}

// -------------- 输出 --------------
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000016f1769e0 (thin lock: 0x000000016f1769e0)                   // 轻量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

**<font color='red'>注意：</font>**调用的`hashCode()`方法必须未重写，否则就算调用对象头中也不会记录 HashCode。如果重写了`hashCode()`方法，那么可以调用`System.identityHashCode(obj)`

### <font color=#1FA774>轻量级锁</font>

偏向锁无法忍受两个及以上的线程竞争锁，会直接升级为轻量级锁；轻量级锁无法忍受三个及以上的线程竞争锁，会直接升级为重量级锁

若线程 A 正在执行同步块，此时线程 B 竞争锁，那么线程 B 会自旋等待。这样是为了避免上下文切换开销大于自旋等待的开销

轻量级锁对应「轻量级锁定」的状态，标志位 00，**它适用于存在竞争，但竞争极小的场景**

当线程准备进入同步块时，如果同步对象锁的状态为无锁，JVM 会首先在线程的栈帧中创建一个锁记录 (Lock Record)，用于存储锁对象目前的 Mark Word，然后拷贝对象头中的 Mark Word 到锁记录中

拷贝成功后，JVM 使用 CAS 操作尝试将对象头的 Mark Word 更新为指向 Lock Record 的指针，并将 Lock Record 中的 owner 指针指向对象的 Mark Word -> 双向奔赴

如果更新成功，直接修改锁对象 Mark Word 标志位为 00；否则判断锁对象 Mark Word 是否指向当前线程的栈帧，如果是，就直接进入同步块，否则表示存在多个线程竞争锁

若一个线程正在执行同步块，一个线程正在自旋等待，此时又来了一个线程，那么轻量级锁升级为重量级锁

#### <font color=#9933FF>小测试</font>

下面来模拟一波轻量级锁升级为重量级锁的过程！！

```java
public class TestLock {
    public static Object obj;
    public static void main(String[] args) throws InterruptedException {
        obj = new Object();
        obj.hashCode(); // 直接轻量级锁，跳过可偏向锁
        System.out.println("before lock");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        sync("主线程");
        Thread.sleep(100);
        // 子线程启动，竞争者 1
        Thread thread1 = new Thread(() -> {
            try {
                sync("子线程 1");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        thread1.start();
        // 子线程启动，竞争者 2
        Thread thread2 = new Thread(() -> {
            try {
                sync("子线程 2");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        thread2.start();
        thread1.join();
        thread2.join();

        System.out.println("after lock");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
    private static void sync(String msg) throws InterruptedException {
        synchronized (obj) {
            System.out.println(msg + "：同步代码块执行中 ...");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
            Thread.sleep(1000);
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
    }
}

// -------------- 输出 --------------
before lock
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000001540e19d01 (hash: 0x1540e19d; age: 0)                            // 无锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

主线程：同步代码块执行中 ...
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000016dc8a9c0 (thin lock: 0x000000016dc8a9c0)                       // 轻量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000016dc8a9c0 (thin lock: 0x000000016dc8a9c0)                       // 轻量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

子线程 1：同步代码块执行中 ...
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000142054be2 (fat lock: 0x0000000142054be2)                       // 重量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000142054be2 (fat lock: 0x0000000142054be2)                       // 重量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

子线程 2：同步代码块执行中 ...
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000142054be2 (fat lock: 0x0000000142054be2)                       // 重量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000142054be2 (fat lock: 0x0000000142054be2)                       // 重量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

after lock
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000142054be2 (fat lock: 0x0000000142054be2)                       // 重量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

**<font color='red'>解释：</font>**主线程正在访问同步代码块，此时子线程 1 和子线程 2 也开始竞争锁，所以需要升级成重量级锁。最后当三个线程都退出同步代码块后，锁的状态还保持为重量级锁，需要等待一会才可以被释放。当我们使用`sleep`暂停几秒钟后，重新将获得轻量级锁，因为导致线程进入 SafePoint，使得 Lock 被降级

```java
// 部分代码
thread2.start();
thread1.join();
thread2.join();

Thread.sleep(5000);   // 添加内容

System.out.println("after lock");
System.out.println(ClassLayout.parseInstance(obj).toPrintable());

sync("主线程");        // 添加内容

// -------------- 部分输出 --------------
after lock
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000001540e19d01 (hash: 0x1540e19d; age: 0)                            // 无锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

主线程：同步代码块执行中 ...
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000016d3da9c0 (thin lock: 0x000000016d3da9c0)                       // 轻量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000000016d3da9c0 (thin lock: 0x000000016d3da9c0)                       // 轻量级锁
  8   4        (object header: class)    0x00000f68
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

### <font color=#1FA774>重量级锁</font>

如果是重量级锁，JVM 会将竞争锁的线程全部挂起，等重量级锁被释放后会唤起被挂起的线程

原来 synchronized 被称为重量级锁，但在 JDK6 中为了减少获得锁和释放锁带来的性能开销引入了偏向锁和轻量级锁

下面对比一下三种锁：可偏向锁、轻量级锁、重量级锁

|    锁    |                             优点                             |                             缺点                             |                适用场景                |
| :------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :------------------------------------: |
| 可偏向锁 | 加锁解锁不需要额外的消耗<br />和执行非同步方法相比仅存在几纳秒的差距 | 如果线程间存在锁竞争<br />会带来额外的锁撤销的消耗 (可偏向锁 -> 轻量级锁) | 适用于只有一个线程访问同步代码块的场景 |
| 轻量级锁 |    竞争的线程不会阻塞，而是自旋等待，提高了程序的响应速度    |            如果长时间得不到锁<br />会自旋消耗 CPU            | 追求响应时间<br />同步代码块执行速度快 |
| 重量级锁 |      线程竞争不使用自旋，而是直接挂起等待，不会消耗 CPU      |                     线程阻塞，响应速度慢                     | 追求吞吐量<br />同步代码块执行速度较长 |

### <font color=#1FA774>参考文章</font>

- **[看完这篇恍然大悟，理解Java中的偏向锁，轻量级锁，重量级锁](https://blog.csdn.net/DBC_121/article/details/105453101)**
- **[浅谈偏向锁、轻量级锁、重量级锁](https://juejin.cn/post/6844903550586191885#comment)**
- **[不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)**
- **[Java锁与线程的那些事](https://tech.youzan.com/javasuo-yu-xian-cheng-de-na-xie-shi/)**
- **[深入理解Java的对象头mark word](https://blog.csdn.net/qq_36434742/article/details/106854061)**
- **[Synchronized 升级到重量级锁之后就下不来了？你错了！](https://juejin.cn/post/6936067917255540767#comment)**