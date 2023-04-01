# ConcurrentHashMap

### <font color=#1FA774>为什么要使用 ConcurrentHashMap</font>

**原因一：线程安全**

- 常用的 **[HashMap](./HashMap源码剖析.html)** 集合不能保证线程安全，在多线程的情况下会引起**死循环**或者**数据丢失**的情况

- 导致死循环的主要原因是 JDK7 时使用头插法添加元素，在扩容时可能会移动元素，而头插法的顺序和原顺序是相反的，会导致链表成环，详细分析可见 **[疫苗：JAVA HASHMAP的死循环](https://coolshell.cn/articles/9606.html)**

- 在 JDK8 时改用尾插法添加元素解决了死循环问题，但在多线程环境下可能还会存在数据丢失的问题。总之 HashMap 不能保证线程安全，所以尽量在多线程下不使用它

**原因二：执行效率**

- 线程安全的 HashTable 存在效率低的问题，因为它是将所有方法都加 synchronized 保证线程安全
- 假设线程 A 使用`get`获取元素，那么其它线程既不能使用`put`添加元素，也不能使用`get`获取元素，即任何时候都只能执行该对象的一个方法，很大程度上降低了执行效率

基于上述两方面的原因 ConcurrentHashMap 应运而生，它的核心就是**分段锁**，即一个对象拥有多把锁，每一把锁只锁住对象的一部分数据，相比于 HashTable 的所有线程竞争一把锁大大提高了效率

举个简单的例子，假设一个 ConcurrentHashMap 对象的数据有 3 部分 A、B、C，那么就会对应三把锁 LockA、LockB、LockC。若线程 1 只会操作 A 部分的数据，那么它只用获取 LockA 即可，完全不会妨碍其它线程竞争 B、C 部分的数据

由于 JDK7 和 JDK8 中的 ConcurrentHashMap 实现上有区别，所以下面的内容分开介绍！！

### <font color=#1FA774>JDK7 中的 ConcurrentHashMap</font>

#### <font color=#9933FF>存储结构</font>

ConcurrentHashMap 是由 Segment 数组和 HashEntry 数组组成。Segment 是一种可重入锁，在 ConcurrentHashMap 扮演锁的角色；HashEntry 则是用于存储键值对数据。具体结构如下图所示

一个 ConcurrentHashMap 包含一个 Segment 数组，Segment 的结构和 HashMap 类似，是数组➕链表的形式。每个 Segment 都守护一个 HashEntry 数组，若修改它，必须先获取对应 Segment 锁

**<font color='red'>注意：</font>**Segment 数组一旦初始化就不能改变其大小，默认大小 16，可认为默认大小下最多支持 16 个线程并发；HashEntry 数组支持扩容



![1](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230401/0130471680283847leYTtY1.svg)



#### <font color=#9933FF>初始化</font>

初始化过程就是通过 initialCapacity (初始容量大小，HashEntry 数组长度，默认 16)，loadFactor (负载因子，默认 0.75)，concurrencyLevel (并发级别，Segment 数组长度，默认 16) 三个参数来初始化 Segment 数组、HashEntry 数组、段偏移量 segmentShift、段掩码 segmentMask

ConcurrentHashMap 有四个构造函数：(最主要的是第四个)

```java
// 无参构造函数，一切均为默认值
public ConcurrentHashMap() {
    // this(16, 0.75, 16)
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}
// 指定初始容量，其它默认
public ConcurrentHashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}
// 指定初始容量和负载因子，其它默认
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, DEFAULT_CONCURRENCY_LEVEL);
}
// 一切均指定
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // MAX_SEGMENTS 最大并发级别，大小为 1 << 16
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;  // (1 << sshift) = 2^n，即：sshift = n
    int ssize = 1;   // Segment 数组实际长度，必须大于等于 concurrencyLevel 且为最小的 2^n
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift;  // 段偏移量，用于定位参与哈希运算的位数
    this.segmentMask = ssize - 1;     // 段掩码，ssize - 1 = 2^n - 1 = 11...11 (n 个 1)
    if (initialCapacity > MAXIMUM_CAPACITY)  // MAXIMUM_CAPACITY 最大容量，大小为 1 << 30
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;  // 平均每个段负责的元素个数
    // 等价于 (initialCapacity + ssize - 1) / ssize，即向上取整
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;  // MIN_SEGMENT_TABLE_CAPACITY 每个段负责的最小元素个数，大小为 2
    // cap 必须 2^n，所以寻找大于等于 c 且为最小的 2^n
    while (cap < c)
        cap <<= 1;
    // 创建 Segment 数组，设置 segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),  // (int)(cap * loadFactor) 表示阈值，元素超过就要扩容
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // 写第一个元素，保证原子性和有序性
    this.segments = ss;
}
```

根据上面的源码总结一下初始化的流程：

- 参数校验：方法参数是否符合要求
- 校验并发级别：判断是否大于所允许的最大并发级别 1 << 16
- 寻找并发级别：寻找 Segment 数组的实际长度，必须大于等于 concurrencyLevel 且为最小的 2^n
- 计算段偏移量和段掩码；计算平均每个段负责的元素个数 c，需要向上取整，然后寻找寻找大于等于 c 且为最小的 2^n 作为实际的 HashEntry 长度
- 创建 Segment 数组，设置 segments[0]，默认大小为 2，扩容阈值 = 2 * 0.75 = 1.5，当插入第二个元素时才会进行扩容

#### <font color=#9933FF>插入</font>

下面开始分析插入操作的源码，也就是`put(key, value)`方法以及它调用的一些方法：

```java
// 插入元素 <key, value>
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);  // 对 HashCode 进行了扰动计算，避免元素分布不均匀
    // 默认情况下，segmentShift = 28，segmentMask = 15
    // 将 hash 高 4 位 & 1111，计算该 key 存入的 Segment 下标
    int j = (hash >>> segmentShift) & segmentMask;
    // 获取 Segment 数组中的第 j 个元素，通过 UNSAFE + 偏移量 获取
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        // s 为 null，初始化 Segment[j]
        s = ensureSegment(j);  // 具体见下方
    return s.put(key, hash, value, false);  // 具体见下方
}
// 通过给定下标 k 利用 CAS 创建 Segment[k]
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // Segment[k] 偏移量
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype (原型)，获取基本参数
        int cap = proto.table.length;    // HashEntry 容量
        float lf = proto.loadFactor;     // 负载因子
        int threshold = (int)(cap * lf); // 阈值
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // 再次检查
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) { // 自旋检查
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))  // 确保更新的原子性
                    break;
            }
        }
    }
    return seg;
}
// 前面还属于数据的处理流程，下面开始真正的插入操作
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // tryLock() 非阻塞获取 ReentrantLock 独占锁，如果获取不到，再使用 scanAndLockForPut() 获取，具体见下方
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;  // (n - 1) & hash 利用位运算计算下标
        HashEntry<K,V> first = entryAt(tab, index);  // CAS 获取下标的值
        for (HashEntry<K,V> e = first;;) {    // 从第一个元素开始遍历链表
            if (e != null) {
                K k;
                // 检查 key 是否存在，如果存在直接替换
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;  // 下一个元素
            }
            else {  // key 不存在的情况，需要插到链表中 (头插法)
                // 头插法：node.next = first
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;  // 统计元素个数
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)  // 判断是否需要扩容
                    rehash(node);  // 扩容
                else
                    setEntryAt(tab, index, node);  // 不需要扩容，直接利用 CAS 更新 tab[index] = node
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();  // 释放锁
    }
    return oldValue;
}
// 不断自旋 tryLock() 获取锁，当自旋次数大于 MAX_SCAN_RETRIES 时，使用 lock() 阻塞获取锁
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);  // 根据 hash 获取第一个元素
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {  // MAX_SCAN_RETRIES = Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1
            lock();  // 自旋达到指定次数后，阻塞获取锁
            break;
        }
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {  // 有其它线程修改了 first，更新
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
// 扩容到原来的两倍，元素位置要么不变，要么变为 i + oldSize，参数中的 node 会使用头插法插入到指定位置
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;  // 新容量
    threshold = (int)(newCapacity * loadFactor);  // 新阈值
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];  // 新 tab
    int sizeMask = newCapacity - 1;
    // 遍历 HashEntry 数组
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];  // 下标 i 处的首节点
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;  // 新下标
            if (next == null)   //  Single node on list (头插法)
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                // 遍历链表，结束后 lastRun 后面的元素的新下标都是相同的，即：lastIdx，直接移过去即可
                for (HashEntry<K,V> last = next; last != null; last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;  // 将 lastRun 及后面的一起移动到新位置
                // 遍历剩余元素，即 lastRun 之前的，移动到新位置，头插法
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    // 头插法插入新的节点 node
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

插入操作的内容有点多，现在来总结一下整个流程：

首先是一个初始化判断，判断对应的 segment 是否初始化，若没有则调用`ensureSegment()`初始化，它底层使用 CAS 进行自旋检查并更新，确保操作的原子性和可见性

在进行真正的插入操作前需要使用`tryLock()`非阻塞的获取 segment 锁，它是`ReentrantLock()`独占锁，若失败则调用`scanAndLockForPut()`自旋调用`tryLock()`，若自旋次数超过指定次数，则改用`lock()`阻塞获取锁

根据 hash 计算得到 HashEntry 数组对应的下标，然后开始遍历。如果插入元素 key 存在则直接覆盖，否则需要头插法插入链表中，在插入前判断是否需要扩容

每次扩容都是原容量的 2 倍，然后开始遍历式移动元素到新 HashEntry 数组中，元素的新下标要么保持不变，要么变为原下标➕原容量，这都是依赖于 HashEntry 数组大小是 $2^n$

#### <font color=#9933FF>查找</font>

相比于插入操作，查找操作就简单很多，只需要两个步骤：

- 计算对应的 segment 数组下标和 HashEntry 数组下标
- 从 HashEntry 指定位置开始从头遍历寻找 key 相同的 value

**<font color='red'>注意：</font>**查找操作不涉及数据更新修改操作，所以不需要用锁来保证线程安全，直接裸奔即可

下面开始分析查找操作的源码，也就是`get(key)`方法：

```java

public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;  // 获取对应的 segment 下标
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        // 从头开始向后找
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            // 判断是否相等：地址相等 or (hash 相等 && 值相等)
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

### <font color=#1FA774>JDK8 中的 ConcurrentHashMap</font>

#### <font color=#9933FF>存储结构</font>

JDK8 中的 ConcurrentHashMap 相比于 JDK7 来说变化比较大，不再是之前的 **Segment 数组 + HashEntry 数组 + 链表**，而是 **Node 数组 + 链表/红黑树**。当冲突链表达到一定长度时，链表会转换成红黑树。具体结构如下图所示

![2](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230401/1650541680339054r71iQ52.svg)

这里要介绍一个特殊的节点类型：

```java
// hash 值为 -1，存储着 nextTable 的引用
// 只有在 table 发生扩容时，ForwardingNode 才会发挥作用，作为一个占位符放在原 table 中表示当前节点为 null 或者已经被移动
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
}
```

#### <font color=#9933FF>初始化</font>

ConcurrentHashMap 有五个构造函数：(最主要的是第五个)

```java
// 无参构造函数，一切均为默认值
public ConcurrentHashMap() {
}
// 指定初始容量，其它默认
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?  // MAXIMUM_CAPACITY = 1 << 30
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));  // tableSizeFor(c) 返回大于等于 c 且最小的 2^n
    this.sizeCtl = cap;
}
// 通过集合初始化
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;  // DEFAULT_CAPACITY = 16
    putAll(m);
}
// 指定初始容量和负载因子，其它默认
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}
// 一切均指定
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

**<font color='red'>注意：</font>**可以看出在构造函数中并没有初始化数组相关的对象，仅仅只是计算了一些参数，实际的初始化在第一次插入的时候，这里提前介绍一下

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 自旋
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl < 0 表示有线程正在初始化或 resize table，所以直接让出 CPU 使用权
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  // 使用 CAS 将 sizeCtl 置为 -1，表示即将要开始初始化。SIZECTL 表示 sizeCtl 的偏移量
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  // n 是初始化容量
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);  // 计算下次扩容的阈值，n - (n >>> 2) = 0.75n
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

从源码中可以看出 table 的初始化是通过 CAS➕自旋 完成的。sizeCtl 决定着当前的初始状态：

- -1 表示有线程正在初始化
- -N 表示有 M-1 个线程正在进行扩容，其中 M 表示 -N 的低 16 位数值。**关于 -N 的解释可见 [ConcurrentHashMap的sizeCtl含义纠正](https://blog.csdn.net/Unknownfuture/article/details/105350537)**
-  如果 table 没有初始化，表示初始化大小
-  如果 table 已经初始化，表示判断扩容阈值，默认是 table 大小的 0.75 倍

**用到的并发技巧：**

- **<font color='red'>volatile 变量 (sizeCtl)：</font>**它是一个标记位，用 -1 表示已经有线程正在初始化，其它线程无法再进行初始化，而且 volatile 保证了该变量在线程间立即可见
- **<font color='red'>CAS 操作：</font>**CAS 保证设置 sizeCtl 标记的原子性，确保了只有一个线程能设置成功

#### <font color=#9933FF>插入</font>

```java
// 当调用 put(k, v) 时，首先会调用下面这个方法
public V put(K key, V value) {
    return putVal(key, value, false);   // putVal() 方法见下方
}
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());  // 进行了一次扰动计算 h ^ (h >>> 16)
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {   // 自旋
        Node<K,V> f; int n, i, fh;      // f 是目标位置元素、fh 目标位置元素的 hash 值
        if (tab == null || (n = tab.length) == 0)  // 进行初始化 (CAS + 自旋)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {  // (n - 1) & hash 计算下标；tabAt() 使用 CAS 获取对应下标元素
            // table[i] 为 null，直接 CAS 放入
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED) // MOVED = -1，表示正在扩容，当前线程帮助扩容，以加快速度
            tab = helpTransfer(tab, f);  // 帮助扩容方法
        else {
            // 把新节点按照链表或者红黑树的方式插入到合适的位置，这个过程采用同步内置锁实现并发
            V oldVal = null;
            synchronized (f) {  // 获取锁
                if (tabAt(tab, i) == f) { // 再次判断，防止被其它线程修改
                    // MOVED     = -1;         // hash for forwarding nodes (迁移节点)
                    // TREEBIN   = -2;         // hash for roots of trees   (树节点)
                    // RESERVED  = -3;         // hash for transient reservations
                    if (fh >= 0) {  // 表示 f 是链表结构，遍历链表。若找到 key，直接覆盖；否则在链表尾部插入新节点
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 找到了 key 相等的元素，直接覆盖
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;  // 存在相同的 key，记录下来
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 遍历到了链表尾部，直接插入
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {  // 表示 f 是 TreeBin 结构，在树结构上遍历元素，更新或增加节点
                        Node<K,V> p;
                        binCount = 2;
                        // 调用红黑树插入节点的方法
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;  // 存在相同的 key，记录下来
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)  // TREEIFY_THRESHOLD = 8
                treeifyBin(tab, i);  // 链表转红黑树
                if (oldVal != null)  // 表示没有插入新的节点，而是直接覆盖，所以此处直接 return
                    return oldVal;
                break;
            }
        }
    }
    // 表示插入成功
    addCount(1L, binCount);
    return null;
}
```

**用到的并发技巧：**

- **<font color='red'>减少锁粒度：</font>**将 Node 数组中的每个元素都单独作为一把锁，大大减小了锁竞争
- **<font color='red'>使用 Unsafe 提供的方法：</font>**使用 Unsafe 提供的原子性方法获取数组中的元素，保证了数据最新
- **<font color='red'>使用 synchronized 关键字：</font>**保证覆盖 value 或插入新节点时的原子性和可见性

#### <font color=#9933FF>帮助扩容</font>

```java
// 当前线程插入元素时发现正在扩容，就会转而帮助扩容，提高扩容速度
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 如果 tab 不为空，且头节点是 fwd，表示正在扩容移动
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length) << RESIZE_STAMP_SHIFT;
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if (sc == rs + MAX_RESIZERS || sc == rs + 1 || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {  // sc + 1 表示增加一个线程来协助扩容
                transfer(tab, nextTab);  // 迁移数据
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

#### <font color=#9933FF>addCount()</font>

在`putVal()`方法的最后，如果插入成功，那么会调用`addCount()`将计数➕1，这里面还会涉及是否需要扩容

```java
// 更改计数值，并检查元素数量是否到达阈值
private final void addCount(long x, int check) {
    // as 计数桶
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {  // CAS 尝试更新 baseCount，baseCount 表示所有元素数量
        // 进入该 if 内有两种可能
        // 1. counterCells 已经被初始化
        // 2. CAS 更新失败，存在竞争
        CounterCell a; long v; int m;
        boolean uncontended = true;  // 标志是否存在竞争，true 表示没有，false 表示有
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||  // 计数桶为 null，直接进入 fullAddCount()
            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 进入该 if 内表示要么计数桶还没有初始化，要么 CAS 更新计数桶失败
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();  // 总元素数量 = baseCount + 各个计数桶中的数量
    }
    if (check >= 0) {    // 检查是否扩容
        Node<K,V>[] tab, nt; int n, sc;
        // sizeCtl 是扩容的阈值
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 进入该循环中表示需要扩容
            int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT;  // rs 高 16 位表示数组长度，低 16 位表示扩容线程数量
            if (sc < 0) {  // 表示已经有线程在进行扩容，只需要帮助扩容即可
                if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
                    (nt = nextTable) == null || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))  // sc + 1 表示增加一个线程来协助扩容
                    transfer(tab, nt);
            }
            // 表示还没有线程在进行扩容，需要重置 sc 使其满足低 16 位的数值 - 1 表示正在扩容的线程数量
            else if (U.compareAndSwapInt(this, SIZECTL, sc, rs + 2)) // 一开始低 16 位的数值为 0，+2 后变成 2，-1 后变成 1，正好是正在扩容的线程数量
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

首先有一个全局变量`baseCount`记录所有元素的数量，其次有一个计数桶数组。该方法给出了两种计数策略

- **竞争小：**当 counterCells 未初始化时，表示之前一直没有出现过竞争，直接将值累加到 baseCount 上即可
- **竞争大：**将计数累加到计数桶中，分而治之，减少竞争

从上面源码可以看出当计数桶没有初始化或者对计数桶更新失败时会调用`fullAddCount()`方法：

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        // 计数桶已经初始化，即数组初始化，但数组中的元素可能没有初始化
        if ((as = counterCells) != null && (n = as.length) > 0) {
            if ((a = as[(n - 1) & h]) == null) { // 随机选择一个计数桶，如果为 null 表示还没有被线程递增过
                if (cellsBusy == 0) {            // 如果不 busy
                    CounterCell r = new CounterCell(x); // 先初始化保存在临时变量 r 中
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {  // 设置 cellsBusy = 1
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            // 在锁下再检查一次
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;  // 将刚刚创建的 r 赋值给 rs[j]
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;  // 标记不 busy
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            // 计数桶不为 null，尝试递增
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            // collide = false，进入后 collide = true，然后重新来一次循环，但此处只会进入一次，也就表示最多只会循环两次
            else if (!collide)
                collide = true;
            // 到这里表示经历了两次循环，且两次都失败了，则需要对计数桶扩容，初始长度为 2
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {  // 设置 cellsBusy = 1
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1]; // 扩容两倍
                        // 将原计数桶中的数量赋值到新的里面
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;  // 赋值
                    }
                } finally {
                    cellsBusy = 0;  // 标记不 busy
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        // 开始初始化计数桶
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {  // CAS 设置 cellsBusy = 1，表示现在计数桶 busy 中 ...
            // 如果有其它线程同时初始化计数桶，由于 CAS 操作只能有一个线程进入这里！！
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {                   // 再次确认桶空
                    CounterCell[] rs = new CounterCell[2];  // 初始化一个长度为 2 的计数桶
                    rs[h & 1] = new CounterCell(x);         // h 是一个随机数，随机选择一个桶
                    counterCells = rs;                      // 将初始化好的计数桶赋值给 ConcurrentHashMap
                    init = true;
                }
            } finally {
                cellsBusy = 0;  // 标记不 busy
            }
            if (init)
                break;
        }
        // 若有线程同时来初始化计数桶，没有抢到 busy 资格的线程通过 CAS 更新 baseCount
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

如果发现计数桶中的元素没有初始化，或者已经初始化但在`addCount()`中 CAS 更新失败，则进入`fullAddCount`

- 若计数桶没有初始化，那么直接初始化一个大小为 2 的数组，随机选择一个元素作为本次递增的计数对象，并使用 CAS➕cellsBusy 更新
- 若计数桶已经初始化，但随机选择桶内的元素没有初始化，那就初始化该元素，并使用 CAS➕cellsBusy 更新
- 若计数桶已经初始化且选择桶内的元素也已经初始化，但 CAS 更新失败，那么再尝试一次 (两次机会)。如果最后依旧更新失败，扩容计数桶为原来的 2 倍

#### <font color=#9933FF>扩容</font>

**ConcurrentHashMap 使用 CAS 操作将扩容的并发性能实现最大化。在扩容过程中，可以安全的`get()`查询数据，如果有线程进行`put()`操作，还会协助扩容**

前文提到 sizeCtl 决定着当前的状态，如果 table 已经初始化，那么 sizeCtl > 0 就表示扩容阈值，当元素个数超过了 sizeCtl 就会触发扩容

在两个地方会引起扩容，第一个就是在上面介绍的`addCount()`方法中；第二个就是在`treeifyBin()`方法中，如果链表节点数 $\ge 8$ 但数组大小 $< 64$，则只会扩容而不会链表转红黑树

这两个地方扩容的步骤差不多，`addCount()`在上面已经分析过，这里重点分析`treeifyBin()`中调用的`tryPresize()`方法

```java
// 调用 tryPresize(n << 1)，n 为原数组大小
private final void tryPresize(int size) {
    // size 已经是原数组大小的两倍
    // c 是 大于等于 size 且最小的 2^n
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        // table 未初始化
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            // 进行扩容
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);  // 数据迁移
        }
    }
}
```

可以看出最主要的还是`transfer()`方法，它是最最最长也是最最最麻烦的，它主要就是进行数据迁移

```java
// tab 原数组，nextTab 新数组
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;  // stride 可以认为是步长，即每个线程负责的原 table 中的桶数量，最少 16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)  // NCPU 表示 CPU 核心数 
        stride = MIN_TRANSFER_STRIDE; // MIN_TRANSFER_STRIDE = 16
    if (nextTab == null) {            // nextTab 为 null 表示第一次扩容
        try {
            @SuppressWarnings("unchecked")
            // n 是原数组的长度，创建一个 2n 大小的 table
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME (内存溢出)
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;  // 赋值给全局变量 nextTable
        transferIndex = n;    // 表示当前线程需要进行数据迁移的桶区间 [transferIndex - stride, transferIndex - 1]  -> 从后往前
    }
    int nextn = nextTab.length; // 新容量
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);  // 当原 table 中某个桶的所有节点都迁移完后，用 fwd 占据该位置
    boolean advance = true;    // 标识一个桶的迁移工作是否完成，true 表示可以进行下一个桶的迁移
    boolean finishing = false; // 最后一个数据迁移的线程将该值置为 true，并进行本轮扩容的收尾工作
    for (int i = 0, bound = 0;;) {  // i 桶索引，bound 左边界
        Node<K,V> f; int fh;
        // 第一次自旋前的预处理，为了定位本轮处理的桶区间
        // 正常情况下，预处理完成后：[bound, i] = [transferIndex - stride, transferIndex - 1]
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 当前是处理最后一个 tranfer 任务的线程或者出现扩容冲突
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {  // 所有桶迁移均已完成
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);  // sizeCtl = 1.5n
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {  // 扩容线程 -1，表示当前线程已经完成自己的 transfer 任务
                // sc 是 sizeCtl - 1 之前的值，判断当前线程是否为本轮扩容中的最后一个线程，如果不是，则直接退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                // 最后一个线程需要重新检查一次原 table 中的所有桶，确保所有桶都被迁移到新 table 中
                // 此时会重新回到 while (advance) 循环中
                i = n; // recheck before commit
            }
        }
        // tab[i] 为 null，不需要迁移，直接放一个 fwd
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // tab[i] 已经为 fwd
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 开始迁移 tab[i]
            synchronized (f) {
                if (tabAt(tab, i) == f) {  // 再次检查 f 有没有被修改
                    Node<K,V> ln, hn;
                    if (fh >= 0) {  // 链表
                        int runBit = fh & n;  // 记录高位是 0 or 1
                        Node<K,V> lastRun = f;
                        // 遍历链表，结束后 lastRun 后面的元素的新下标都是相同的，即：lastIdx，直接移过去即可
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {  // 下标不变
                            ln = lastRun;
                            hn = null;
                        }
                        else {              // 下标改变
                            hn = lastRun;
                            ln = null;
                        }
                        // 遍历剩余元素，即 lastRun 之前的，移动到新位置，头插法
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)  // 下标不变
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else                // 下标改变
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);     // 下标不变
                        setTabAt(nextTab, i + n, hn); // 下标改变
                        setTabAt(tab, i, fwd);        // 原 table 中用 fwd 占位
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {  // 红黑树
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // UNTREEIFY_THRESHOLD = 6，判断是否需要红黑树转链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

#### <font color=#9933FF>查找</font>

相比于上面的`put()`而言，`get()`的流程简单太多！！

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());  // 进行了一次扰动计算 h ^ (h >>> 16)
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {  // hash 相同
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // MOVED     = -1;         // hash for forwarding nodes (迁移节点)
        // TREEBIN   = -2;         // hash for roots of trees   (树节点)
        // RESERVED  = -3;         // hash for transient reservations
        else if (eh < 0)  // 判断是否是特殊节点
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {  // 遍历链表
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

### <font color=#1FA774>参考文章</font>

- **Java 并发编程的艺术**
- **[ConcurrentHashMap源码&底层数据结构分析](https://javaguide.cn/java/collection/concurrent-hash-map-source-code.html)**
- **[Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析](https://www.javadoop.com/post/hashmap#Java7%20ConcurrentHashMap)**
- **[ConcurrentHashMap扩容迁移等方法的源码分析](https://blog.csdn.net/wwj17647590781/article/details/118151008)**
- **[ConcurrentHashMap是如何实现线程安全的](https://blog.csdn.net/qq_41737716/article/details/90549847)**
- **[ConcurrentHashMap的sizeCtl含义纠正](https://blog.csdn.net/Unknownfuture/article/details/105350537)**
- **[深入浅出ConcurrentHashMap1.8](https://www.jianshu.com/p/c0642afe03e0)**