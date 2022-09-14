# 「equals」「hashCode」

Java 中所有对象都继承`Object`对象，而`Object`类中有两个方法：`equals()`和`hashCode()`，它们是用来判断两个对象是否相等的

本篇文章就刨根问底式的好好总结一下这个知识点！！

### <font color=#1FA774>内存地址</font>

在正式介绍`equals()`和`hashCode()`之前，我们先来聊聊「对象的内存地址」

根据 Java 内存区域的划分可知，「对象」都是存储在「堆」上的，那如何正确获取对象内存地址呢？

首先添加依赖：

```xml
<dependency> 
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.10</version>
</dependency>
```

然后使用`addressOf()`方法：

```java
Object obj = new Object();
System.out.println("----------- GC 前 -----------");
System.out.println("The memory address is " + VM.current().addressOf(obj));
System.gc();
System.out.println("----------- GC 后 -----------");
System.out.println("The memory address is " + VM.current().addressOf(obj));

// ----------- GC 前 -----------
// The memory address is 26304775200
// ----------- GC 后 -----------
// The memory address is 25803930568
```

**<font color='red'>注意</font>**：大多数 JVM 实现中的内存地址会随着 GC 时移动对象而发生变化

#### <font color=#9933FF>identityHashCode()</font>

关于`identityHashCode()`和`VM.current().addressOf()`的区别，前者是**对象初始地址**的哈希值，后者是对象在内存中的实际地址

```java
Object obj = new Object();
System.out.println("Memory address: " + VM.current().addressOf(obj));
System.out.println("identityHashCode: " + System.identityHashCode(obj));
System.out.println("hashCode: " + obj.hashCode());
System.out.println("toString: " + obj.toString());

// Memory address: 26304775288
// identityHashCode: 2106620844
// hashCode: 2106620844
// toString: java.lang.Object@7d907bac
```

关于`identityHashCode()`和`hashCode()`的区别，这里引用 jdk 中的一句话

- **<font color='red'>Returns the same hash code for the given object as would be returned by the default method hashCode(), whether or not the given object's class overrides hashCode(). The hash code for the null reference is zero.</font>**

- **解释：**无论`hashCode()`有没有被重写，`identityHashCode()`返回的始终是默认的 hashCode 值。换句话说，如果`hashCode()`没有被重写，那么这两个方法返回的值是相同的；否则就是不同的
- 值得注意的是，默认的`toString()`方法返回的是 hashCode 的十六进制表达 -> 「7d907bac : 2106620844」

为什么说`identityHashCode()`是**对象初始地址**的哈希值呢？

- 因为对象的实际地址随着 GC 会发生改变，如果根据实际地址计算哈希值，那么对象每一次移动都会导致哈希值不同

```java
Object obj = new Object();
System.out.println("----------- GC 前 -----------");
System.out.println("The memory address is " + VM.current().addressOf(obj));
System.out.println("identityHashCode: " + System.identityHashCode(obj));
System.gc();
System.out.println("----------- GC 后 -----------");
System.out.println("The memory address is " + VM.current().addressOf(obj));
System.out.println("identityHashCode: " + System.identityHashCode(obj));

// ----------- GC 前 -----------
// The memory address is 26304775352
// identityHashCode: 2106620844
// ----------- GC 后 -----------
// The memory address is 25866845360
// identityHashCode: 2106620844
```

我们发现，GC 前后 the memory address 发生了变化，但是 identityHashCode 没有改变

**原因：**对象一旦调用了计算哈希的函数，那么哈希值就会被存储在 object header 中，下次直接从 object header 中取即可

### <font color=#1FA774>equals()</font>

如果我们自定义的类没有重写`equals()`，那么它就是直接使用`Object`类的`equals()`，如下所示：

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

这样是直接比较两个对象的**<font color='red'>内存地址</font>**，所以在这种情况下，`equals()`和`==`是等价的！！

```java
MyClass c1 = new MyClass();
MyClass c2 = new MyClass();
// 下面两种比较方法是等价的
System.out.println(c1.equals(c2));   // false
System.out.println(c1 == c2);        // false
```

下面的问题面试经常被问，根据上面的分析那到底怎么回答才更佳呢？

**<font color='red'>「equals() 方法」与「== 运算符」有什么区别呢？</font>**

- 默认情况下也就是从父类`Object`继承而来的「`equals()`方法」与「==」是完全等价的，比较的都是对象的**内存地址**
- 但我们可以重写`equals()`方法，使其按照我们需要的方式进行比较，如`String`类重写了`equals()`方法，使其比较的是字符的序列，而不再是内存地址

### <font color=#1FA774>hashCode()</font>

`hashCode()`在「内存地址」部分已经提到过，如果没有重写，它和`identityHashCode()`返回的值没啥区别

重点在于「重写」，著名问题：**<font color='red'>为什么重写 equals() 的同时还得重写 hashCode() ？</font>**

- equals - 保证比较对象是否绝对相等
- hashCode - 保证在最快的时间内判断两个对象是否相等，可能有误差值「哈希冲突」

一个保证可靠性，一个保证性能：

- 判断两个对象是否相等，首先比较两个对象的 hashCode 是否相等。如果不相等，一定不同；但就算相等，也不一定相同「不同对象可能存在相同的 hashCode，概率极低」
- hashCode 相等的前提下，继续用 equals 进一步判断是否相等

用 String 举例，摘出了关键代码：

```java
public boolean equals(Object anObject) {
    // 先比较内存地址是否相等
    // 注意：这里的「内存地址」不等同于「hashCode」，而是和「identityHashCode」等价
    if (this == anObject) {
        return true;
    }
    return (anObject instanceof String aString)
        && (!COMPACT_STRINGS || this.coder == aString.coder)
        && StringLatin1.equals(value, aString.value);    // 核心比较部分，见下一个函数
}
// 核心比较部分
public static boolean equals(byte[] value, byte[] other) {
    // 比较每个字符是否相等
    if (value.length == other.length) {
        for (int i = 0; i < value.length; i++) {
            if (value[i] != other[i]) {
                return false;
            }
        }
        return true;
    }
    return false;
}
```

所以很明显了，如果所有的对象都直接用 equals 比较，显然很浪费时间。hashCode 只用比较两个整数是否相等，所以很快。先比较一波 hashCode，相等后再用 equals 比较

#### <font color=#9933FF>hashCode 在 Set Map 中的应用</font>

Java 中的`Set`和`Map`在比较`key`时，也是采用这种思想，先比较 hashCode，再比较 equals

```java
if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
```

设想一种场景，当我们使用自定义的对象作为`Map`的`key`时，而正好只重写`equals()`，没有重写`hashCode()`，会发生什么情况？？

```java
Map<PhoneNumber, String> m = new HashMap<>();
// 向 Map 中加入了一个 k v 对
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
// 通过 k 获得 v
m.get(new PhoneNumber(707, 867, 5309));  // 返回 null
```

最后和我们设想的结果可能不一致，其实原因也很简单，`put`时的 key 对象和`get`时的 key 对象，虽然它们的字段相同，但确是两个不同的对象，hashCode 显然不同，所以导致最后结果为 null

如果想要结果和预期一样，就必须重写`hashCode()`，让内存地址不同，但字段相同的对象有一样的 hashCode 值

最后，附上阿里巴巴 Java 开发手册上的一段话：

> 关于 hashCode 和 equals 的处理，遵循如下规则：
>
> - 只要覆写 equals，就必须覆写 hashCode
> - 因为 Set 存储的是不重复的对象，依据 hashCode 和 equals 进行判断，所以 Set 存储的对象必须覆写这两种方法
> - 如果自定义对象作为 Map 的键，那么必须覆写 hashCode 和 equals
>
> 说明：String 因为覆写了 hashCode 和 equals 方法，所以可以愉快地将 String 对象作为 key 来使用

#### <font color=#9933FF>hashCode 生成方式</font>

前文说默认的 hashCode 是**对象初始地址**的哈希值，其实不太严谨！！

不同的JVM对hashcode值的生成方式不同。Open JDK 中提供了 6 中生成 hash 值的方法

- 0：随机数生成器 (A randomly generated number)
- 1：通过对象内存地址的函数生成 (A function of memory address of the object)
- 2：硬编码1 (用于敏感度测试) (A hardcoded 1 (used for sensitivity testing)
- 3：通过序列 (A sequence)
- 4：对象的内存地址，强制转换为int。 (The memory address of the object, cast to int)
- 5：线程状态与xorshift结合 (Thread state combined with xorshift)

其中在 OpenJDK 6、7 中使用的是随机数生成器的 (第 0 种) 方式，OpenJDK 8、9 则采用第 5 种作为默认的生成方式

所以，单纯从 OpenJDK 的实现来说，其实 hashcode 的生成与对象内存地址没有什么关系。而 Object 类中 hashCode 方法上的注释，很有可能是早期版本中使用到了第 4 种方式

#### <font color=#9933FF>hashCode 的约定</font>

最后，我们来看一下对 hashCode 方法的约定和说明

- 当一个对象`equals`方法所使用的字段不变时，多次调用`hashCode`方法的值应保持不变
- 如果两个对象`equals(Object o)`方法是相等的，则`hashCode`方法值必须相等
- 如果两个对象`equals(Object o)`方法是不相等，则`hashCode`方法值不要求相等，但在这种情况下尽量确保`hashCode`不同，以提升性能

### <font color=#1FA774>参考文章</font>

- Effective Java 3
- 阿里巴巴 Java 开发手册
- [Java 正确获取对象内存地址的方式](https://www.awaimai.com/2981.html)
- [Java中如何获取对象（引用）地址？](https://blog.csdn.net/isscollege/article/details/78398968)
- [GC时对象地址变了，hashCode如何保持不变？](https://heapdump.cn/article/2576724)
- [为什么重写equals必须重写hashCode](https://segmentfault.com/a/1190000024478811)
- [面试官：为什么重写equals时必须重写hashCode方法？](https://www.modb.pro/db/49230)
- [重写equal()时为什么也得重写hashCode()之深度解读equal方法与hashCode方法渊源](https://blog.csdn.net/javazejian/article/details/51348320)
