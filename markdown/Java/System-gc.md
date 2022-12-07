# System.gc()

Runs the garbage collector.
Calling the gc method suggests that the Java Virtual Machine expend effort toward recycling unused objects in order to make the memory they currently occupy available for quick reuse. When control returns from the method call, the Java Virtual Machine has made a best effort to reclaim space from all discarded objects.
The call System.gc() is effectively equivalent to the call: Runtime.getRuntime().gc()

上面是 JDK8 对这个方法的说明，大概意思就是**调用该方法会触发垃圾收集行为**，下面强调几个点：

- `System.gc()`和`Runtime.getRuntime().gc()`等价
- `System.gc()`会显示触发 Full GC，同时对年轻代和老年代进行回收，尝试释放被丢弃对象的内存空间
- `System.gc()`附带一个免责声明，不能保证对垃圾收集器的调用 (不能确保立即生效)

**注意：**可以通过`System.gc()`调用来决定 Java 虚拟机 GC 的行为；但一般情况下，垃圾回收应该是自动进行的！！

对于不同垃圾收集器，`System.gc()`的表现可能略有不同，测试代码如下所示：

```java
// -Xms1024m -Xmx1024m -XX:+PrintGCDetails
public static void main(String[] args) {
    byte[] buffer = new byte[10 * 1024 * 1024];  // 10MB
    System.gc();
}
```

本人电脑中 HotSpot 虚拟机使用的垃圾收集器是 UseParallelGC，如下所示：

```bash
➜  ~ java -XX:+PrintCommandLineFlags -version
-XX:InitialHeapSize=536870912 -XX:MaxHeapSize=8589934592 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
openjdk version "1.8.0_352"
OpenJDK Runtime Environment (Zulu 8.66.0.15-CA-macos-aarch64) (build 1.8.0_352-b08)
OpenJDK 64-Bit Server VM (Zulu 8.66.0.15-CA-macos-aarch64) (build 25.352-b08, mixed mode)
```

使用 ParallelGC 输出的结果如下所示：

```java
[GC (System.gc()) [PSYoungGen: 20725K->10816K(305664K)] 20725K->10824K(1005056K), 0.0033120 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 10816K->0K(305664K)] [ParOldGen: 8K->10598K(699392K)] 10824K->10598K(1005056K), [Metaspace: 3230K->3230K(1056768K)], 0.0029934 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 305664K, used 13107K [0x00000007aab00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 262144K, 5% used [0x00000007aab00000,0x00000007ab7cce68,0x00000007bab00000)
  from space 43520K, 0% used [0x00000007bab00000,0x00000007bab00000,0x00000007bd580000)
  to   space 43520K, 0% used [0x00000007bd580000,0x00000007bd580000,0x00000007c0000000)
 ParOldGen       total 699392K, used 10598K [0x0000000780000000, 0x00000007aab00000, 0x00000007aab00000)
  object space 699392K, 1% used [0x0000000780000000,0x0000000780a59a70,0x00000007aab00000)
 Metaspace       used 3259K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 347K, capacity 388K, committed 512K, reserved 1048576K
```

可以看到一共进行了两次 GC，分别是 Young GC 和 Full GC。原因如下：

- **<font color='red'>arallel Scavenge (-XX:+UseParallelGC) 框架下，默认是在要触发 Full GC 前先执行一次 Young GC，并且两次 GC 之间能让应用程序稍微运行一小下，以期降低 Full GC 的暂停时间</font>**
- 因为 Young GC 会尽量清理了年轻代的死对象，减少了 Full GC 的工作量

**关于这个内容的详细解释可见 [Major GC和Full GC的区别是什么？触发条件呢？ - RednaxelaFX的回答 - 知乎](https://www.zhihu.com/question/41922036/answer/93079526)**

如果我们使用参数`-XX:+UseSerialGC`，那么就只进了一次 Full GC，输出结果如下所示：

```java
[Full GC (System.gc()) [Tenured: 0K->10599K(699072K), 0.0023243 secs] 21424K->10599K(1013632K), [Metaspace: 3250K->3250K(1056768K)], 0.0023414 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 314560K, used 13981K [0x0000000780000000, 0x0000000795550000, 0x0000000795550000)
  eden space 279616K,   5% used [0x0000000780000000, 0x0000000780da74c8, 0x0000000791110000)
  from space 34944K,   0% used [0x0000000791110000, 0x0000000791110000, 0x0000000793330000)
  to   space 34944K,   0% used [0x0000000793330000, 0x0000000793330000, 0x0000000795550000)
 tenured generation   total 699072K, used 10599K [0x0000000795550000, 0x00000007c0000000, 0x00000007c0000000)
   the space 699072K,   1% used [0x0000000795550000, 0x0000000795fa9e38, 0x0000000795faa000, 0x00000007c0000000)
 Metaspace       used 3261K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 347K, capacity 388K, committed 512K, reserved 1048576K
```

无论是使用 ParallelGC 还是使用 SerialGC，都会发现一个很奇怪的地方，慢慢道来...(注意：我们的分析基于`ParallelGC`)

根据虚拟机启动参数`-Xms1024m -Xmx1024m `可以知道为 Java 堆分配了固定的 1024m 的内存，利用`jstat`命令查看内存分配情况：

```bash
➜  ~ jps            
22214 
43398 Main
44281 Jps
44268 Launcher
44269 Test01
➜  ~ jstat -gc 44269
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
43520.0 43520.0  0.0    0.0   262144.0 25968.7   699392.0     0.0     4480.0 800.1  384.0   77.0       0    0.000   0      0.000    0.000
```

新生代和老年代的默认比例是 1 : 2，所以新生代和老年代的内存大小分别约为 298.5m : 683m (注意：新生代只算一个 Survivor 区的大小)

新生代中 Eden、s0、s1 的默认比例是 8 : 1 : 1，所以它们的内存大小分别约为 256m : 42.5m : 42.5m

上面具体把每个区的大小计算出来只是为了有个大小概念，有些误差没关系，只需要明确对于 10MB 大小的`buffer`数组存到任何区域都是绰绰有余的

首先`buffer`会在 Eden 区分配内存，当执行为一次 Young GC 后，`buffer`会被移动到 s0 区，但是当执行完 Full GC，新生代中的对象全部移动到了老年代

此时问题就来了，新生代晋升到老年代只有四种情况：

- 大对象直接在老年代分配：如果一个对象的大小超过了参数`-XX:PretenureSizeThreshold`指定的值 (默认为 0)，那么就直接在老年代为对象分配内存
- 长期存活的对象将进入老年代：如果一个对象的 GC 年龄超过了参数`-XX:MaxTenuringThreshold`指定的值 (默认为 15)，那么在下一次 GC 时就会被移动到老年代
- 动态对象年龄判定：如果 Survivor 中低于或等于某年龄的所有对象大小的总和大于 Survivor 空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无需等到上一情况中年龄的要求
- 空间分配担保：如果 Young GC 时，Survivor 不能存下所有年轻代中存活的对象，那么就需要老年代空间担保，将 Survivor 存不下的对象直接晋升到老年代中

这个例子好像不满足上述四种中的任何一种情况，那为什么会在 Full GC 时将新生代中的对象全部移动到老年代呢？

**<font color='red'>HotSpot 的 Full GC 实现中，默认年轻代中所有活的对象都要晋升到老年代，实在晋升不了才会留在年轻代</font>**

**注意：**执行完 Full GC 后老年代内存空间的使用**<font color='red'>不减反增</font>**也是非常正常的行为。假如执行 Full GC 的时候，老年代中的对象几乎没有死掉的，而年轻代又有活对象晋升来，那么 Full GC 结束后老年代内存空间的使用自然就上升了

**关于这个内容的详细解释可见 [JVM full GC的奇怪现象，求解惑？ - RednaxelaFX的回答 - 知乎](https://www.zhihu.com/question/48780091/answer/113063216)**
