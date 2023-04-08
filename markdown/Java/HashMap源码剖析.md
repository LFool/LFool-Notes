# HashMap 源码剖析

### <font color=#1FA774>前置知识</font>

**注意：**该部分如果没有特殊提 JDK 版本，那么就都是基于 JDK8 中的 HashMap！！

#### <font color=#9933FF>扰动函数</font>

HashMap 本质上是一种哈希表，在不考虑哈希冲突的情况下，增删改查操作的时间复杂度均为 O(1)。哈希表的本质是一个数组，每次操作都是根据下标，才能达到时间复杂度均为 O(1) 的效果

哈希表的底层原理是将对象的**哈希值**通过一个**哈希函数**映射成一个下标地址，然后将该元素存入数组中对应的下标中。但在实际中，可能存在多个对象的哈希值映射成下标后相等的情况，最简单的做法就是拉链法

上述说的情况被称之为**哈希冲突**，哈希冲突堆积越严重就会越影响增删改查的效率，所以最理想的情况就是元素均匀分布在哈希表中。下图中的第一种情况显然要比第二种情况查找效率高

![1](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230318/0027241679070444vX8W7w1.svg)

我们知道 Java 中每个对象都有一个自己的 HashCode，可否直接用 HashCode % n 作为数组的下标，其中 n 为数组的长度

原则上是可以的，但是 HashMap 并没有直接使用对象的 HashCode，而是进行了一次扰动，被称之为扰动函数：

```java
static final int hash(Object key) {
    int h;
    // 原 HashCode 和右移 16 位的 HashCode 异或，结合了 HashCode 的高 16 位和低 16 位
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

扰动函数的目的为了防止一些实现较差的`hashCode()`方法导致元素没有较为均匀的分布在哈希表中，增加了哈希冲突的概率，降低了数据存取的效率

#### <font color=#9933FF>初始容量</font>

这里首先需要明确两个概念：数组长度和元素个数。数组长度表示哈希表的长度，即上图中绿色的部分，长度为 6；元素的个数表示哈希表中存放的元素个数，即上图中紫色的部分

所以可能出现元素个数大于数组长度的情况！！这一部分说的容量概念就是指数组的长度

HashMap 中有一个默认的初始容量，大小为 16：

```java
// The default initial capacity - MUST be a power of two.
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

上面的注释提到容量大小必须是 $2^n$，HashMap 有一个可以自定义容量大小的构造函数：

```java
public HashMap(int initialCapacity, float loadFactor) {
    // 省略部分代码 ...
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
// 返回大于 cap 且最小的 2 的次方
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

可以看到如果传入的不是一个 $2^n$，那么就会通过`tableSizeFor()`方法获得一个大于目标数且最小的 $2^n$ 的一个数作为初始容量

这里可以引出一个很经典的问题：**<font color='red'>HashMap 的初始容量为什么是 $2^n$，以及扩容为什么是 2 倍的形式？</font>**

我们平时将一个数映射到 10 以内的话，都是直接取模，如：`n % 10`，这里我们也可以用相同的方式：`hash % cap`，但 HashMap 并没有使用这种方法，而是用位运算`hash & (cap - 1)`

这两种方式只有在 $cap = 2^n$ 的时候才相等，不信的话自己可以尝试几个例子。HashMap 之所有使用位运算是因为`&`比`%`的效率高很多

这里还有第二个原因，不得不提到一个阈值的概念`threshold = capacity * loadFactor`，loadFactor 是下面要介绍的负载因子。当 HashMap 中**元素个数** > threshold 后就需要扩容：

```java
if (++size > threshold)
	resize();
// ...
if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&  // 新容量是原容量的两倍
     oldCap >= DEFAULT_INITIAL_CAPACITY)
newThr = oldThr << 1; // double threshold         // 新阈值是原阈值的两倍
```

既然扩容了，那么有些元素肯定需要重新计算哈希映射，然后移动到新的位置，如下图所示：

![2](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230318/1748321679132912bGMEON2.svg)

可以看到如果扩容后容量大小依旧保持 $2^n$，那么将会很容易计算元素的新位置。如果扰动计算后的 hash 的高一位为 0，那么扩容后位置将不变；如果为 1，那么扩容后的位置将是原位置➕扩容前的容量

#### <font color=#9933FF>负载因子</font>

负载因子控制数组存放元素的疏密程度，通过负载因子和容量大小可以计算得到一个阈值，即：`threshold = capacity * loadFactor`，当 HashMap 中**元素个数** > threshold 后就需要扩容

所以当负载因子越接近于 1，那么数组中存放的元素就越多，也就越密集；当负载因子越接近于 0，那么数组中存放的元素就越少，也就越稀疏

负载因子越大就会导致查询元素效率低；越小就会导致空间利用率低，存放元素分散。官方给出的默认值为 0.75

通过默认的容量大小 16 和默认的负载因子 0.75，就可以计算出阈值为 12，表示当元素个数大于 12 后就需要进行扩容

#### <font color=#9933FF>数组扩容</font>

关于数组扩容，在介绍上面三个概念的时候都提过，所以本部分就不再赘述。简单的从源码角度分析一下如何确定元素扩容后的位置：

```java
// e 表示当前要处理的元素
// oldCap 表示扩容前容量
// loHead 和 loTail 表示位置没变的的链表头尾节点
// hiHead 和 hiTail 表示位置变了的的链表头尾节点
if ((e.hash & oldCap) == 0) {
    // 位置没变
    if (loTail == null)
        loHead = e;
    else
        loTail.next = e;
    loTail = e;
}
else {
    // 位置变了
    if (hiTail == null)
        hiHead = e;
    else
        hiTail.next = e;
    hiTail = e;
}
```

### <font color=#1FA774>底层数据结构</font>

在 JDK8 之前 HashMap 底层是数组➕链表相结合的形式

在 JDK8 及之后 HashMap 底层数据结构有了变化。当链表长度大于阈值 (默认为 8) 时，首先会调用`treeifyBin()`方法来判断是否需要转换成红黑树。只有当数组长度 $\ge 64$ 时，才会执行`treeify()`方法真正的转换成红黑树，否则知识简单的扩容

```java
// TREEIFY_THRESHOLD 为 8
if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
	treeifyBin(tab, hash);

final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // MIN_TREEIFY_CAPACITY 为 64
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            // 树化
            hd.treeify(tab);
    }
}
```

### <font color=#1FA774>源码分析</font>

#### <font color=#9933FF>构造函数</font>

HashMap 的构造函数只负责一件事情，初始化负载因子 loadFactor 和初始容量大小 initialCapacity。它提供了三个构造函数：

```java
// 无参构造函数，使用默认的 loadFactor (0.75) 和 initialCapacity (16)
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
// 指定初始容量大小，使用默认的 loadFactor (0.75)
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
// 指定初始容量大小和负载因子
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    // 通过 tableSizeFor() 方法获得一个大于目标数且最小的 2^n 的一个数作为初始容量
    this.threshold = tableSizeFor(initialCapacity);
}
```

**注意：**构造函数中并没有对数组进行初始化，HashMap 将对数组的初始化延迟到了调用`put()`方法时，在`resize()`中初始化数组，扩容也在`resize()`中

#### <font color=#9933FF>插入</font>

插入是 HashMap 使用非常频繁的操作，下面是插入的流程图：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230318/2218011679149081DUntyf3.svg" alt="3" style="zoom:80%;" />

接着结合源码分析一波上述流程：

```java
// 当调用 put(k, v) 时，首先会调用下面这个方法
public V put(K key, V value) {
    // hash(key) 是上面提到的扰动函数；putVal() 方法见下方
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table 是 HashMap 底层的数组结构
    // 如果 table 没有初始化或者 table 的长度为 0，直接执行 resize() 方法。HashMap 数组初始化延迟到此处
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // (n - 1) & hash 利用位运算计算下标
    // 如果该下标处为 null，表示该位置还没有存储元素，直接将当前要插入元素放在这里即可
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 下面是该下标处不为 null 的情况，后面可能是链表结构，也可能是红黑树结构
        Node<K,V> e; K k;
        // 判断该下标处的 key 值是否和要插入元素的 key 值相等。如果相等，直接覆盖 value 即可
        // 判断策略：先判断 hash 是否相等；再判断 key 地址或实际值是否相等 (这也是为什么重写 equals 方法的同时也要重写 hashCode 方法)
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 判断该元素是否为 TreeNode 类型。如果是 TreeNode 表示后面存储结构是红黑树；否则表示后面的存储结构是链表
        else if (p instanceof TreeNode)
            // 执行红黑树的插入操作
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 后面的存储结构是链表
        else {
            // 从头开始循环链表，寻找插入位置
            for (int binCount = 0; ; ++binCount) {
                // 已经到链表最后一个位置，表示链表中并没有 key 相同的元素
                if ((e = p.next) == null) {
                    // 直接接在链表的最后面
                    p.next = newNode(hash, key, value, null);
                    // 如果当前链表长度大于 8 就执行树化
                    // 注意：执行 treeifyBin() 方法并非一定会被树化，还需要满足数组长度大于 64，该方法具体见下方
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 判断 key 是否相等，判断策略同上
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // e 不等于 null 表示插入的 key 已经存在，直接覆盖 value 即可
        if (e != null) { // existing mapping for key
            // 覆盖 value
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 判断是否扩容
    if (++size > threshold)
        resize();   // 具体见下方
    afterNodeInsertion(evict);
    return null;
}
// 判断是否树化
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // MIN_TREEIFY_CAPACITY = 64
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            // 真正的树化
            hd.treeify(tab);
    }
}
// 扩容方法
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    // 原容量
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 原阈值
    int oldThr = threshold;
    // 新容量、新阈值
    int newCap, newThr = 0;
    // 原容量不为 0 表示 table 已经被初始化过
    if (oldCap > 0) {
        // 原容量大于最大容量，无法扩容，增大阈值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新容量为原容量的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 新阈值为原阈值的两倍
            newThr = oldThr << 1; // double threshold
    }
    // 原容量为 0，但原阈值不为 0 表示使用带参构造函数创建 HashMap 对象，此时的 oldThr 表示初始容量大小
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 原容量和原阈值均为 0 表示使用无参构造函数创建 HashMap 对象
    else {               // zero initial threshold signifies using defaults
        // 默认初始容量
        newCap = DEFAULT_INITIAL_CAPACITY;
        // 阈值 = 默认初始容量 * 负载因子
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 执行到此处，newCap 肯定不为 0，但 newThr 可能为 0
    // 如果 newThr 为 0，就用 newCap * loadFactor 获得新的阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 更新 threshold
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 初始化新 table
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // oldTab 不为 null 表示需要拆分元素，将 oldTab 中的元素分配到 newTab 中
    if (oldTab != null) {
        // 遍历 oldTab 数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // e 表示当前处理元素
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 表示该下标处只有一个元素，直接计算新的下标 e.hash & (newCap - 1)
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 底层结构为树
                else if (e instanceof TreeNode)
                    // 该方法具体见下方
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 底层结构为链表
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;   // 保持位置不变的链表
                    Node<K,V> hiHead = null, hiTail = null;   // 位置变化的链表
                    Node<K,V> next;
                    // 遍历链表
                    do {
                        next = e.next;
                        // 高位为 0 表示位置不变
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 高位为 1 表示位置改变
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;            // 位置不变
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;   // 位置改变 -> 新下标 = 原下标 + 原容量
                    }
                }
            }
        }
    }
    return newTab;
}
// 底层结构为树的拆分方法
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        // 高位为 0
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        // 高位为 1
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }
    // 位置保持不变
    if (loHead != null) {
        // UNTREEIFY_THRESHOLD = 6
        if (lc <= UNTREEIFY_THRESHOLD)
            // untreeify 将红黑树转换成链表
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    // 位置改变
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            // untreeify 将红黑树转换成链表
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

#### <font color=#9933FF>查找</font>

上面分析了插入操作，下面的查找操作就相对于简单很多。同样的，下面是查找的流程图：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20230319/0104251679159065JBcTnK4.svg" alt="4" style="zoom:80%;" />

接着结合源码分析一波上述流程：

```java
// 当调用 get(k) 时，首先会调用下面这个方法
public V get(Object key) {
    Node<K,V> e;
    // getNode() 方法见下方
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // table 是否初始化、table 长度是否大于 0、table 对应下标处是否为 null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 判断第一个元素
        // 判断策略：先判断 hash 是否相等；再判断 key 地址或实际值是否相等 (这也是为什么重写 equals 方法的同时也要重写 hashCode 方法)
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 从下一个元素开始继续判断
        if ((e = first.next) != null) {
            // 判断该元素是否为 TreeNode 类型
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 在链表中循环遍历查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```