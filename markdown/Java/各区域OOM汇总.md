# 各区域 OOM 汇总

运行时数据区一共有五个部分组成，如下图所示：

![90](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221120/1341531668922913hhaD9x90.svg)

除了程序计数器外，其它区域都有发生 **OutOfMemoryError (OOM)** 异常的可能，本篇文章通过实验来验证异常实际发生的代码场景！！

**在实验的过程中，会用到一些虚拟机参数，参数具体含义可见 [运行时数据区常用参数汇总](./运行时数据区常用参数汇总.html)**

### <font color=#1FA774>Java 堆溢出</font>

**关于 Java 堆的详细内容可见 [Java 堆](./运行时数据区域.html#堆)**

Java 堆用于存储对象实例，只要不断地创建对象，并且保证 GC Roots 到对象之间有可达路径来避免垃圾回收机制清除这些对象，那么随着对象数量的增加，总容量触及最大堆的容量限制后就会产生 OOM

下面代码限制 Java 堆的大小为 10MB，不可扩展，通过参数`-XX:+HeapDumpOnOutOfMemoryError`可以让虚拟机在出现内存溢出异常的是会后 Dump 出当前的内存堆转储快照以便进行事后分析

```java
// -Xms10m -Xmx10m -XX:+HeapDumpOnOutOfMemoryError
public class HeapOOM {
    static class OOMObject {}
    public static void main(String[] args) throws InterruptedException {
        List<OOMObject> list = new ArrayList<>();
        while (true) {
            list.add(new OOMObject());
        }
    }
}
```

运行结果：

```java
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid3283.hprof ...
Heap dump file created [21460433 bytes in 0.046 secs]
```

Java 堆内存的 OOM 异常是实际应用中最常见的内存溢出异常情况。出现 Java 堆内存溢出时，异常堆栈信息`java.lang.OutOfMemoryError`会跟随进一步提示`Java heap space`

要解决 OOM 异常或 Heap Space 的异常，一般的手段是首先通过内存映像分析工具 (如 Eclipse Memory Analyzer) 对 dump 出来的堆转储快照进行分析，重点是**<font color='red'>确认内存中的对象是否是必要的</font>**，也就是要先分清楚到底是出现了**<font color='red'>内存泄漏 (Memory Leak)</font>** 还是**<font color='red'>内存溢出 (Memory Overflow)</font>**

- **内存泄漏：**本应该被回收的对象，由于错误的引用链导致未进行回收
- **内存溢出：**为对象分配内存时，发现内存不够，导致无法完成内存分配

有两种可能导致**内存溢出**。其一：由于内存泄漏导致内存溢出；其二：未发生内存泄漏，单纯的是因为内存不够导致内存溢出

如果是内存泄漏，可进一步通过工具查看泄漏对象到 GC Roots 的引用链。于是就能找到泄漏对象是通过怎样的路径与 GC Roots 相关联并导致垃圾收集器无法自动回收它们的。掌握了泄漏对象的类型信息，以及 GC Roots 引用链的信息，就可以比较准确地定位出泄漏代码的位置

如果不存在内存泄漏，换句话说就是内存中的对象确实都还必须存活着，那就应当检查虚拟机的堆参数 (`-Xmx`与`-Xms`)，与机器物理内存对比看是否还可以调大，从代码上检查是否存在某些对象生命周期过长、持有状态时间过长的情况，尝试减少程序运行期的内存消耗

利用 JProfiler 分析上面 Dump 出来的内存堆转储快照：

![1](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221120/1530131668929413t5f4x71.svg)

可以很明显的看到是`OOMObject`对象占用了大量的内存，此时可以从该对象入手，分析到底是内存泄漏导致的内存溢出，还是内存不足导致的内存溢出！！

### <font color=#1FA774>虚拟机栈溢出</font>

**关于虚拟机栈的详细内容可见 [虚拟机栈](./虚拟机栈.html)**

关于虚拟机栈和本地方法栈，在《Java 虚拟机规范》中描述了两种异常：

- 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出 **StackOverflowError (SOF)** 异常
- 如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出 **OutOfMemoryError (OOM)** 异常

《Java 虚拟机规范》明确允许 Java 虚拟机实现自行选择是否支持栈的动态扩展，而 HotSpot 虚拟机的选择时不支持扩展，所以除非在**创建线程**申请内存时就因无法获得足够内存而出现 OOM 异常，否则在线程**运行时**是不会因为扩展而导致内存溢出的，只会因为栈容量无法容纳新的栈帧而导致 SOF 异常

#### <font color=#9933FF>实验一：测试 SOF 异常</font>

实验环境：

- 单线程
- 使用`-Xss`参数减少栈内存容量，固定为`640k`

```java
// -Xss640k
public class JavaVMStackSOF {
    private int stackLength = 1;
    public void stackLeak() {
        stackLength++;
        stackLeak();
    }
    public static void main(String[] args) {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length: " + oom.stackLength);
            throw e;
        }
    }
}
```

运行结果：

```java
stack length: 5163
Exception in thread "main" java.lang.StackOverflowError
	at com.lfool.myself.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12)
	at com.lfool.myself.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12)
	at com.lfool.myself.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12)
    ......
```

#### <font color=#9933FF>实验二：测试 OOM 异常</font>

```java
// -Xss2m -Xms10m -Xmx10m -XX:+HeapDumpOnOutOfMemoryError
public class JavaVMStackOOM {
    private void dontStop() {
        while (true) { }
    }
    public void stackLeakbyThread() {
        while (true) {
            Thread thread = new Thread(this::dontStop);
            thread.start();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakbyThread();
    }
}
```

### <font color=#1FA774>方法区溢出</font>

**关于方法区的详细内容可见 [方法区](./方法区.html)**

这里就不展开论述了！！等我有更深刻的理解后，再补！！