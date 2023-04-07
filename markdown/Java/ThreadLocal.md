# ThreadLocal

### <font color=#1FA774>ThreadLocal 引入</font>

在单线程应用程序中可能会维持一个全局的数据库连接，在程序启动时初始化这个连接对象，从而避免在调用每个方法时都要传递一个 Connection 对象

但是如果在多线程中没有同步的情况下使用全局共享变量就会存在线程安全问题，而 ThreadLocal 可以保证每个线程拥有属于自己的连接对象，相互独立

```java
public class ThreadLocalTest {
    // 全局共享变量
    private static final ThreadLocal<String> threadLocal = new ThreadLocal<>();
    public static void main(String[] args) {
        new Thread(() -> {
            threadLocal.set("abc");
            System.out.println("线程一：" + threadLocal.get());
        }).start();
        new Thread(() -> {
            threadLocal.set("def");
            System.out.println("线程二：" + threadLocal.get());
        }).start();
    }
}
// result
线程一：abc
线程二：def
```

每个线程都有一片属于自己的独立内存空间，它是一个`Map`，底层是一个`Entry[]`数组，而每个`Entry`中包含了`key & value`，其中`key`就是`ThreadLocal`对象，而`value`就是设置的值

可能描述的比较模糊，直接来一个图：

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230407/1951481680868308vTTr2l1.svg)

当调用`threadLocal.get()`获取线程私有值时，首先会获得线程私有的`ThreadLocalMap`对象，然后以`threadLocal`为 key 获取到对应的`Entry`对象，最终获取对应的 value 值

### <font color=#1FA774>ThreadLocal 数据结构</font>

上一部分已经稍微介绍了一点 ThreadLocal 底层结构，这一部分从源码的角度详细介绍一波！！

首先每个线程都有一个自己的`ThreadLocalMap`对象，它是`ThreadLocal`的静态内部类：

```java
public class Thread implements Runnable {
    // 省略其它代码 ...
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

接着看一下`ThreadLocalMap`的结构：

```java
static class ThreadLocalMap {
    // Entry 类
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
		// key 继承 WeakReference 类，是一个弱引用
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    // Entry 数组
    private Entry[] table;
}
```

可以看出`ThreadLocalMap`和`HashMap`有些许的相似，**关于 HashMap 详细介绍可见 [HashMap 源码剖析](./HashMap源码剖析.html)**

但也有一些值得关注的点：弱引用 -> **<font color='red'>指一些非必须的对象，但它比软引用强度更弱，被弱引用关联的对象只能生存到下一次垃圾收集发生为止</font>**。**关于四种引用的详细介绍可见[「强/软/弱/虚」引用](./强软弱虚引用.html)**

如果`ThreadLocal`对象只有`Entry`对它的一个弱引用，那么当 JVM 进行 GC 时会将`ThreadLocal`回收，如下图所示：

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230407/2022521680870172CB5bco2.svg)

但此时`value`对象还存在，就出现了内存泄露。不过不要紧，因为后续会检测到 Entry 不为 null 但 key 为 null 的对象，然后将其清理掉，后续会介绍

回到上一部分的代码，思考一下`threadLocal`只有弱引用吗？？？显然不是，它还存在一个强引用！！！所以`threadLocal`对象并不会被 GC 清理

**<font color='red'>注意：</font>**`ThreadLocal`必须有且仅有弱引用时才会在 GC 时被清理

### <font color=#1FA774>计算 ThreadLocal 对象的 HashCode</font>

后文的源码分析会涉及到计算`ThreadLocal`对象的 HashCode，所以这里先来介绍一波～～

在`ThreadLocal`类中有一个`threadLocalHashCode`变量记录着对象的 HashCode，主要从这里入手：

```java
// threadLocalHashCode 为 final，调用 nextHashCode() 计算一次后就不会再改变
private final int threadLocalHashCode = nextHashCode();           // 调用 nextHashCode() 获得 HashCode
private static AtomicInteger nextHashCode = new AtomicInteger();  // 一个原子类整型变量
private static final int HASH_INCREMENT = 0x61c88647;             // 每次增加的步长 (十六进制)
private static int nextHashCode() {                               // 在上一个 HashCode 的基础上增加 HASH_INCREMENT
    return nextHashCode.getAndAdd(HASH_INCREMENT);                // 通过 unsafe 保证原子性
}
```

可以看到`ThreadLocal`对象计算 HashCode 的方式有些特别，不像传统的通过重写`hashCode()`方法，而是设置一个步长，当前对象 HashCode = Prev-HashCode + step

可以写一个代码验证一下：

```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
    ThreadLocal<String> threadLocal1 = new ThreadLocal<>();
    ThreadLocal<String> threadLocal2 = new ThreadLocal<>();
    // 通过反射获取对象的 threadLocalHashCode 值
    Class<?> aClass = ThreadLocal.class;
    Field field = aClass.getDeclaredField("threadLocalHashCode");
    field.setAccessible(true);  // 设置访问权限
    int o1 = (int) field.get(threadLocal1);
    int o2 = (int) field.get(threadLocal2);
    System.out.println(o1);
    System.out.println(o2);
    System.out.println(o2 - o1);
}
// result
-1401181199
239350328
1640531527
```

从输出可以看出两个`ThreadLocal`对象的 HashCode 差值刚好是 0x61c88647 的倍数 (0x61c88647 的十进制为 1640531527)

至于为什么设置增长步长为 0x61c88647，是因为这样可以使计算得到索引分布的更均匀，减少哈希冲突

### <font color=#1FA774>ThreadLocal.set()</font>

下面开始在源码的世界畅游！！当直接调用`set()`方法时如下：

```java
public void set(T value) {
    Thread t = Thread.currentThread();  // 当前线程
    ThreadLocalMap map = getMap(t);     // 根据当前线程 t 获取对应的 ThreadLocalMap，属于线程私有
    if (map != null) {                  // 表示 map 已经初始化
        map.set(this, value);           // 尝试将 value 插入 map 中
    } else {                            // 表示 map 未初始化
        createMap(t, value);            // 初始化 map
    }
}
// 根据线程 t 获取对应的 ThreadLocalMap
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;  // 直接返回即可
}
```

在分析`ThreadLocalMap`结构时可以发现`Thread`类并没有对它初始化，而是直接设置了一个 null 值。真正的初始化是在第一次调用`set()`方法时通过`createMap()`方法初始化：

```java
void createMap(Thread t, T firstValue) {
    // this 表示当前 ThreadLocal 对象
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
// key 为 ThreadLocal 对象；value 为传入的值
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];  // INITIAL_CAPACITY = 16
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);  // 根据 HashCode 计算索引
    table[i] = new Entry(firstKey, firstValue);  // 在对应索引处根据 key 和 value 创建 Entry 对象
    size = 1;  // size 表示 map 中放入元素的个数
    setThreshold(INITIAL_CAPACITY);  // 设置阈值
}
// 2/3 倍 Entry 数组大小
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

回到`set()`中，如果`map`已经被初始化，那么将尝试将 (key, value) 插入 map 中：

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);  // 应被插入的索引下标
    
    // 处理哈希冲突，向后找，直到遇到 null
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {  // nextIndex() 是一个循环增加，到了 len 会从 0 开始
        ThreadLocal<?> k = e.get();
        // 遇到相同的 key，直接覆盖 value
        if (k == key) {
            e.value = value;
            return;
        }
        // 遇到 key = null，表示 tab[i] 失效
        if (k == null) {
            // 这个函数主要完成了两件事：
            // 1. 向前向后寻找清理失效 entry 的边界，以 entry = null 时结束
            //    如果向前没有找到，那么就向后找第二个失效的 entry，开始清理，第一个失效的 entry 需要用来存放插入新值
            // 2. 向后找是否有 key 相同的 entry，如果有则交换，然后覆盖 value，如果没有则直接覆盖第一个失效的 entry，以 entry = null 时结束
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 表示向后找的过程中没有遇到失效的 tab[i]，此时 tab[i] = null
    tab[i] = new Entry(key, value);  // 直接赋值
    int sz = ++size;  // 更新元素的个数
    // 判断是否需要重新根据哈希移动位置：没有失效的元素可以清理 && 元素的个数 >= 阈值
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();  // 注意：rehash() 只是重新根据 hash 移动了元素位置，并没有扩容
}
```

在向后寻找处理哈希冲突时，如果遇到了失效元素会调用`replaceStaleEntry()`方法，它的逻辑比较复杂，单拎出来介绍：

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    int slotToExpunge = staleSlot;
    // 从 i - 1 开始向前寻找，直到遇到 null。如果寻找过程中遇到 key = null 的元素，更新 slotToExpunge
    // 所以 [slotToExpunge, staleSlot] 维护了前一个 entry = null 之后的存在 key = null 的最大区间 
    for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len))
        if (e.get() == null)  // key = null 时更新 slotToExpunge
            slotToExpunge = i;

    // 从 i + 1 开始向后寻找，直到遇到 null
    for (int i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 向后寻找的过程中遇到了 key 相等的情况
        if (k == key) {
            e.value = value;  // 覆盖 value

            // tab[i] 表示 key 相同的 entry；tab[staleSlot] 表示 key = null 的 entry
            // 交换两者
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // staleSlot 一直为出现哈希冲突后向后寻找的第一个 key = null 的下标
            // slotToExpunge = staleSlot 表示向前寻找时没有找到 key = null 的 entry
            // 上面将 i 和 staleSlot 交换了位置，所以此时 i 是向后寻找出现的第一个 key = null 的下标
            // 将 slotToExpunge  设置为 i，后面从 slotToExpunge 开始清理失效的 entry (key = null)
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            // expungeStaleEntry() 向后线性清理失效 entry
            // cleanSomeSlots() 启发式清理失效 entry
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }
        // 执行到此处表示没有遇到 key 相同的 entry
        // 如果向前没有找到失效的 entry，那么 slotToExpunge 记录着 staleSlot 后第一个失效的 entry 下标
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }
    // staleSlot 是理论插入索引后的第一个失效索引；slotToExpunge 是理论插入索引后的第二个失效索引
    // 第一个失效索引用来覆盖插入的新值，所以从第二个失效索引开始清理 entry
    tab[staleSlot].value = null;  // 设置为 null，有助于 GC
    tab[staleSlot] = new Entry(key, value);
    // 从 slotToExpung 开始清理 entry
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

该部分逻辑有点复杂，但我们只需要抓住几个判断的重点：

- 理论索引处是否存在冲突？
- 向前遍历是否有失效元素？
- 向后遍历是否有失效元素？注意有两次向后遍历，第一次从理论索引处向后遍历，第二次从第一个失效元素处向后遍历
- 向后遍历是否存在 key 相同的元素？
- 向后遍历是否存在第二个失效元素？这决定了向前遍历无失效元素时，slotToExpunge 是否会在向后遍历时获得赋值机会，如果没有赋值那么 slotToExpunge = staleSlot，不会触发清理失效元素

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230408/0416091680898569aLHQGK3.svg)

到此为止，介绍过的部分完成了两件事情：

- 安顿好了要插入的元素，存在三种情况：在 null 处创建；在失效元素处创建；在 key 相等元素处创建。前两者都需要重新`new Entry()`，而第三者只需要覆盖 value
- 确定了是否需要从 slotToExpunge 处开始清理失效元素

所以下面开始介绍如何清理失效元素，从上面可以看出主要是`cleanSomeSlots(expungeStaleEntry(slotToExpunge), len)`来清理，下面就先看内层方法`expungeStaleEntry()`：

```java
// 线性清理。参数 staleSlot = slotToExpunge
// 返回从 staleSlot 开始第一个 null 位置的下标 i
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 将 tab[staleSlot] 和对应的 value 置为 null，方便 GC。key 不需要置 null，因为它是一个弱应用
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;  // 更新元素个数

    // Rehash until we encounter null
    Entry e;
    int i;
    // tab[i] = null 时退出循环
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();  // 获取 e 中的 key
        if (k == null) {     // 元素失效
            e.value = null;  // 置为 null
            tab[i] = null;   // 置为 null
            size--;          // 更新元素个数
        } else {             // 元素没有失效
            int h = k.threadLocalHashCode & (len - 1);  // 计算元素索引 (下标)
            if (h != i) {    // 表示该元素并没有在对的位置，因为哈希冲突的存在
                // 下面是移动该元素，为了让它离正确的位置更近，h 是它正确的位置
                tab[i] = null; // GC
                // 从 tab[h] 开始向后找第一个 null 位置，最坏的情况是回到 i
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    // tab[i] = null
    return i;
}
```

下面来看看清理元素的外层方法`cleanSomeSlots()`：

```java
// 启发式清理。从 i + 1 开始
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;  // 记录是否清理了元素
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);  // i + 1
        Entry e = tab[i];
        if (e != null && e.get() == null) {  // 清理失效元素
            n = len;                         // 一旦探测到有失效元素，设置 n 为 table 长度
            removed = true;
            i = expungeStaleEntry(i);        // 从 i 开始线性清理
        }
    } while ( (n >>>= 1) != 0);              // 结束条件。如果没有探测到失效元素，n 每次减半；如果探测到了失效元素，n 重新等于 table 长度
    return removed;
}
```

### <font color=#1FA774>ThreadLocalMap 扩容机制</font>

虽迟但到，有 Map 怎么会没有扩容机制呢？？！！这不来了吗！！在`ThreadLocalMap`的`set()`方法的最后两行，有一个关键性判断：

```java
private void set(ThreadLocal<?> key, Object value) {
    // 省略其它代码 ...
    
    // 判断是否需要重新根据哈希移动位置：没有失效的元素可以清理 && 元素的个数 >= 阈值
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();  // 注意：rehash() 只是重新根据 hash 移动了元素位置，并没有扩容
}
```

先来看看`rehash()`方法：

```java
private void rehash() {
    expungeStaleEntries();  // 先清理一波失效元素，看看能不能腾出一点空位
    
    // 清理完后如果元素个数 >= 3/4 倍的阈值，就需要真正的扩容
    if (size >= threshold - threshold / 4)
        resize();
}
// 从 0 开始整体清理一波失效元素，底层还是调用 expungeStaleEntry()
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)  // 失效元素
            expungeStaleEntry(j);          // 该方法见上面
    }
}
```

再来看看`resize()`方法：

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;  // 新容量为原容量的两倍
    Entry[] newTab = new Entry[newLen];  // 新 Entry 数组
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();  // 获取 key
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);  // 计算新索引下标
                //  从 h 开始找到第一个不为 null 的位置
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;  // 放入元素
                count++;
            }
        }
    }
    setThreshold(newLen);  // 设置新阈值，2/3 * newLen
    size = count;
    table = newTab;
}
```

### <font color=#1FA774>ThreadLocal.get()</font>

`get`相比于`set`就简单多了，直接看吧～～

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);  // this 是调用 get() 方法的 ThreadLocal 对象
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;  // 获取 value
            return result;          // 返回
        }
    }
    // map = null || e = null，设置初始值并返回，具体见下方
    return setInitialValue();
}
private T setInitialValue() {
    T value = initialValue();  // initialValue() 直接返回 null
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    // 下面同 set()
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
    if (this instanceof TerminatingThreadLocal) {
        TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
    }
    return value;
}
```

可以看出，如果`ThreadLocalMap`有`ThreadLocal`对象为 key 的 value 值，那么就直接返回；如果没有，会向`ThreadLocalMap`中插入一个`<ThreadLocal, null>`的键值对，并返回 null

### <font color=#1FA774>参考文章</font>

- **[面试官：小伙子，听说你看过ThreadLocal源码？](https://juejin.cn/post/6844904151567040519)**
