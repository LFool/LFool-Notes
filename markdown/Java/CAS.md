# CAS

### <font color=#1FA774>悲观锁 VS 乐观锁</font>

在多线程并发编程的环境下，共享变量的更新是一门艺术，很容易就造成数据不一致

根据 JMM 可知，每个线程都有自己的工作内存且相互独立，保存着主内存的副本。每次线程需要读写数据时，需要去主内存拷贝一份到自己的工作内存中

假设主内存有一个共享变量`x = 1`

- 线程 A 在时刻一从主内存中拷贝了一份到自己的工作内存，时刻二将工作内存中的`x + 1`，时刻三将工作内存中的`x`写回主内存中

- 线程 B 在时刻一从主内存中拷贝了一份到自己的工作内存，时刻二将工作内存中的`x + 1`，时刻三将工作内存中的`x`写回主内存中

此时主内存中变量`x`的值为 2，但是两个线程都对它 +1，正确的结果应该是 3 才对，这就是并发问题

目前对共享变量的更新采用两种模式：

- **悲观锁：**悲观的认为在更新数据时大概率会有其它线程去争夺共享资源，所以在获取资源时就将该资源锁住，当自己用完了才释放该资源，其它线程才可以重新争夺该资源。synchronized 就是 Java 中悲观锁典型的实现，有对象锁和类锁，但是其它没有争抢到资源的线程会被阻塞，当资源可再次被争夺时才会从阻塞态转运行态，涉及到线程切换，存在上下文切换开销，效率比较慢
- **乐观锁：**乐观的认为在更新数据时有其它线程去争夺共享资源的概率非常小，所以更新数据时不需要加锁，但是在正式更新数据前会去检查数据是否被其它线程修改过，如果没有修改就直接更新，否则就执行不同操作 (报错或自动重试)。CAS 就是一种乐观锁典型的实现

悲观锁适合写操作多的场景，先加锁可以保证写操作时数据正确

乐观锁适合读操作多的场景，不加锁可以提高读操作的性能

### <font color=#1FA774>CAS 是什么</font>

**<font color='red'>CAS (Compare And Swap)：</font>**比较并替换，它有三个核心参数：**共享变量的内存地址**、**预期原值**、**新值**

每次去指定内存地址更新共享变量值之前先判断是否和预期原值一致，如果一致就直接更新，否则就不做任何处理。所以往往 CAS 都配合失败重试使用

**注意：**就 CAS 更新这一操作来说，它是一个原子性操作，不会造成数据不一致性

### <font color=#1FA774>CAS 缺点</font>

**<font color='red'>ABA 问题</font>**

预期原值：A；新值：C

如果共享变量的值被其它线程修改为 B，那么此时 CAS 就会更新失败，然后不断重试。若在重试阶段有其它线程将共享变量的值改回了 A，那么就会被 CAS 更新成功

共享变量的值经历了 A -> B -> A 的过程，虽然最后的值回到了 A，但却还是被修改过。所以为了解决这种问题，加入了版本的概念：1A -> 2B -> 3A

现在判断是否和预期原值相等就必须判断值和版本是否均相等才可以被更新

**<font color='red'>可能会消耗较高的 CPU</font>**

虽然 CAS 没有使用锁，线程从阻塞机制变成了非阻塞机制，但是在线程竞争比较大的时候，如果 CAS 不能更新成功，就会一直重试，消耗 CPU 资源

**<font color='red'>不能保证代码块的原子性</font>**

CAS 只能保证一个共享变量更新的原子性，对于代码块的原子性依旧不能保证，这个时候就需要用到 synchronized 了 (万能！！)

### <font color=#1FA774>CAS 优点</font>

- 可以保证变量操作的原子性
- 并发量不高的情况下，CAS 比 synchronized 效率高
- 资源占用时间较短的情况下，CAS 比 synchronized 效率高

### <font color=#1FA774>Demo</font>

```java
public class CASTest {
    // 共享变量
    private volatile int a;
    // unsafe 有 CAS 方法
    private static final Unsafe unsafe = reflectGetUnsafe();
    public static void main(String[] args) {
        CASTest casTest = new CASTest();
        // 线程 1
        new Thread(() -> {
            for (int i = 1; i < 5; i++) {
                // 修改共享变量
                casTest.increment(i);
                System.out.print(casTest.a + " ");
            }
        }).start();
        // 线程 2
        new Thread(() -> {
            // 刚刚开始会卡在 i = 5 处不断重试，直到共享变量的值等于 4
            for (int i = 5; i < 10; i++) {
                // 修改共享变量
                casTest.increment(i);
                System.out.print(casTest.a + " ");
            }
        }).start();
    }

    public int increment(int x) {
        // 如果失败的话，就不断重试
        while (true) {
            try {
                // 获取共享变量的偏移量
                long fieldOffset = unsafe.objectFieldOffset(CASTest.class.getDeclaredField("a"));
                // 预期原值：x - 1；新值：x
                if (unsafe.compareAndSwapInt(this, fieldOffset, x - 1, x)) break;
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
            }
        }
    }
    // 利用反射获取 Unsafe 对象
    // 不能直接通过 getUnsafe() 方法获取，因为只有被启动类加载的类才可以成功调用该方法
    // 原因：为了安全，因为 Unsafe 直接操作内存，使用不当会引发安全问题
    public static Unsafe reflectGetUnsafe() {
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            return (Unsafe) field.get(null);
        } catch (Exception e) {
            return null;
        }
    }
}
// 输出
// 1 2 3 4 5 6 7 8 9 
```

### <font color=#1FA774>参考文章</font>

- **[CAS 操作](https://javaguide.cn/java/basis/unsafe.html#cas-操作)**

- **[并发编程的基石——CAS机制](https://www.cnblogs.com/54chensongxia/p/12160085.html)**

- **[不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)**
