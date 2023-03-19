[toc]

# Java 集合框架

### <font color=#1FA774>整体框架</font>

Java 集合大致可以分为两大体系，一个是 Collection，另一个是 Map

- Collection ：主要由 List、Set、Queue 接口组成，List 代表有序、重复的集合；其中 Set 代表无序、不可重复的集合；Java 5 又增加了 Queue 体系集合，代表一种队列集合实现
- Map：则代表具有映射关系的键值对集合

`java.util.Collection`和`java.util.Map`下的接口和继承关系简单结构图：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220602/1658301654160310TDTfe78.svg" alt="8" style="zoom:70%;" />

其中，Java 集合框架中主要封装的是典型的数据结构和算法，如动态数组、双向链表、队列、栈、Set、Map 等

将集合框架挖掘处理，可以分为以下几个部分

- **数据结构：**`List`列表、`Queue`队列、`Deque`双端队列、`Set`集合、`Map`映射
- **比较器：**`Comparator`比较器、`Comparable`排序接口
- **算法：**`Collections`常用算法类、`Arrays`静态数组的排序、查找算法
- **迭代器：**`Iterator`通用迭代器、`ListIterator`针对`List`特化的迭代器

### <font color=#1FA774>有序列表「List」</font>

`List`集合的特点：**存取有序**，可以**存储重复的元素**，可以**使用下标操作元素**

`List`主要实现的类：`ArrayList`、`LinkedList`、`Vector`、`Stack`

#### <font color=#9933FF>ArrayList</font>

`ArrayList`是一个动态数组结构，支持随机存取，尾部插入删除方便，内部插入删除效率低 (因为要移动数组元素)；如果内部数组容量不足则自动扩容，因此当数组很大时，效率较低

#### <font color=#9933FF>LinkedList</font>

`LinkedList`是一个双向链表结构，在任意位置插入删除都很方便，但是不支持随机取值，每次都只能从一端开始遍历，直到找到查询的对象，然后返回

不过，它不像`ArrayList`那样需要进行内存拷贝，因此相对来说效率较高，但是因为存在额外的前驱和后继节点指针，因此占用的内存比`ArrayList`多一些

#### <font color=#9933FF>Vector</font>

`Vector`也是一个动态数组结构，是一个元老级别的类，早在 jdk1.1 就引入该类，之后在 jdk1.2 里引进`ArrayList`，`ArrayList`大部分的方法和`Vector`比较相似，但两者是不同的，`Vector`是允许同步访问的，`Vector`中的操作是线程安全的，但是效率低，而`ArrayList`所有的操作都是异步的，执行效率高，但不安全！

关于`Vector`，现在用的很少了，因为里面的`get`、`set`、`add`等方法都加了`synchronized`，所以，执行效率会比较低，如果需要在多线程中使用，可以采用下面语句创建`ArrayList`对象

```java
List<Object> list = Collections.synchronizedList(new ArrayList<>());
```

也可以考虑使用复制容器`java.util.concurrent.CopyOnWriteArrayList`进行操作，例如：

```java
final CopyOnWriteArrayList<Object> cowList = new CopyOnWriteArrayList<>();
```

#### <font color=#9933FF>Stack</font>

`Stack`是`Vector`的一个子类，本质也是一个动态数组结构，不同的是，它的数据结构是先进后出，取名叫栈！

关于`Stack`，现在用的也很少，因为有个`ArrayDeque`双端队列，可以替代`Stack`所有的功能，并且执行效率比它高！

### <font color=#1FA774>集「Set」</font>

`Set`集合的特点：**元素不重复**，**存取无序**，**无下标**

`Set`主要实现类：`HashSet`、`LinkedHashSet`、`TreeSet`

#### <font color=#9933FF>HashSet</font>

`HashSet`底层是基于`HashMap`的`key`实现的，元素不可重复，特性同`HashMap`

#### <font color=#9933FF>LinkedHashSet</font>

`LinkedHashSet`底层也是基于`LinkedHashMap`的`key`实现的，一样元素不可重复，特性同`LinkedHashMap`

#### <font color=#9933FF>TreeSet</font>

同样的，`TreeSet`也是基于`TreeMap`的`key`实现的，同样元素不可重复，特性同`TreeMap`

**`Set`集合的实现，基本都是基于`Map`中的键做文章，使用`Map`中键不能重复、无序的特性；所以，我们只需要重点关注`Map`的实现即可！**

### <font color=#1FA774>队列「Queue」</font>

`Queue`是一个队列集合，队列通常是指「先进先出 (FIFO)」的容器。新元素插入`offer`到队列的尾部，访问元素`poll`操作会返回队列头部的元素。通常，队列不允许随机访问队列中的元素

`Queue`主要实现类：`ArrayDeque`、`LinkedList`、`PriorityQueue`

#### <font color=#9933FF>ArrayDeque</font>

`ArrayQueue`是一个基于数组实现的双端队列，可以想象，在队列中存在两个指针，一个指向头部，一个指向尾部，因此它具有「FIFO 队列」及「栈」的方法特性

#### <font color=#9933FF>LinkedList</font>

`LinkedList`是`List`接口的实现类，也是`Deque`的实现类，底层是一种双向链表的数据结构

`LinkedList`同时实现了`stack`、`Queue`、`PriorityQueue`的所有功能

在上面也有所介绍，`LinkedList`可以根据索引来获取元素，增加或删除元素的效率较高，如果查找的话需要遍历整合集合，效率较低

#### <font color=#9933FF>PriorityQueue</font>

`PriorityQueue`也是一个队列的实现类，此实现类中存储的元素排列并不是按照元素添加的顺序进行排列，而是内部会按元素的大小顺序进行排列，是一种能够自动排序的队列

### <font color=#1FA774>映射表「Map」</font>

`Map`是一个双列集合，其中保存的是键值对，键要求保持唯一性，值可以重复

`Map`主要实现类：`HashMap`、`LinkedHashMap`、`TreeMap`、`IdentityHashMap`、`WeakHashMap`、`Hashtable`、`Properties`

#### <font color=#9933FF>HashMap</font>

关于`HashMap`，相信大家都不陌生，继承自`AbstractMap`，`key`不可重复，因为使用的是哈希表存储元素，所以输入的数据与输出的数据，顺序基本不一致

另外，`HashMap`最多只允许一条记录的`key`为`null`

#### <font color=#9933FF>LinkedHashMap</font>

`HashMap`的子类，内部使用链表数据结构来记录插入的顺序，使得输入的记录顺序和输出的记录顺序是相同的

**`LinkedHashMap`与`HashMap`最大的不同处在于，`LinkedHashMap`输入的记录和输出的记录顺序是相同的！**

#### <font color=#9933FF>TreeMap</font>

能够把它保存的记录根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当用`Iterator`遍历时，得到的记录是排过序的

如需使用排序的映射，建议使用`TreeMap`，TreeMap实际使用的比较少！

#### <font color=#9933FF>IdentityHashMap</font>

继承自`AbstractMap`，与`HashMap`有些不同，在获取元素的时候，通过`==`代替`equals ()`来进行判断，**比较的是内存地址**

#### <font color=#9933FF>WeakHashMap</font>

`WeakHashMap`继承自`AbstractMap`，被称为缓存`Map`，向`WeakHashMap`中添加元素，再次通过键调用方法获取元素方法时，不一定获取到元素值，因为`WeakHashMap`中的`Entry`可能随时被 GC 回收

#### <font color=#9933FF>Hashtable</font>

`Hashtable`是一个元老级的类，键值不能为空，与`HashMap`不同的是，方法都加了`synchronized`同步锁，是线程安全的，但是效率上，没有HashMap快！

同时，HashMap 是 HashTable 的轻量级实现，他们都完成了Map 接口，区别在于 HashMap 允许K和V为空，而HashTable不允许K和V为空，由于非线程安全，效率上可能高于 Hashtable。

如果需要在多线程环境下使用HashMap，可以使用如下的同步器来实现或者使用并发工具包中的`ConcurrentHashMap`类

```java
Map<String, Object> map = Collections.synchronizedMap(new HashMap<>());

ConcurrentHashMap<String, Object> concurrentHashMap = new ConcurrentHashMap<>();
```

#### <font color=#9933FF>Properties</font>

`Properties`继承自`Hashtable`，`Properties`新增了`load()`和`store()`方法，可以直接导入或者将映射写入文件，另外，`Properties`的键和值都是`String`类型

### <font color=#1FA774>比较器</font>

`Comparable`和`Comparator`接口都是用来比较大小的，一般在`TreeSet`、`TreeMap`接口中使用的比较多，主要用于解决排序问题

#### <font color=#9933FF>Comparable</font>

对实现该接口的每个类的对象进行排序

```java
package java.lang;
import java.util.*;
public interface Comparable<T> {
    public int compareTo(T o);
}
```

若一个类实现了`Comparable`接口，实现`Comparable`接口的类的对象的`List`列表 (或数组) 可以通过`Collections.sort` (或`Arrays.sort`) 进行排序

此外，实现`Comparable`接口的类的对象 可以用作「有序映射」(如`TreeMap`) 中的键或「有序集合」(如`TreeSet`) 中的元素，而不需要指定比较器

例子：

```java
/**
 * @Description: 实体类 Person 实现 Comparable 接口
 * @Author: LFool
 * @Date 2022/6/2 17:25
 **/
public class Person implements Comparable<Person> {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public int compareTo(Person o) {
        // 按照从小到大的顺序排列
        return this.age - o.age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "age=" + age +
                ", name='" + name + '\'' +
                '}';
    }
}
```

测试：

```java
public static void main(String[] args) {
    List<Person> list = new ArrayList<>();
    list.add(new Person("zs", 17));
    list.add(new Person("ls", 16));
    list.add(new Person("ww", 18));
    System.out.println(list.toString());
    Collections.sort(list);
    System.out.println(list.toString());
}
```

输出：

```
[Person{age=17, name='zs'}, Person{age=16, name='ls'}, Person{age=18, name='ww'}]
[Person{age=16, name='ls'}, Person{age=17, name='zs'}, Person{age=18, name='ww'}]
```

#### <font color=#9933FF>Comparator</font>

对实现该接口的每个类的对象进行排序

```java
public interface Comparator<T> {
    int compare(T o1, T o2);
    // ......
}
```

如果我们的这个类`Person`无法修改或者没有实现`Comparable`接口，我们又要对其进行排序，`Comparator`就可以派上用场了

将类`Person`实现的`Comparable`接口去掉

```java
/**
 * @Description: 实体类 Person
 * @Author: LFool
 * @Date 2022/6/2 17:25
 **/
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public int getAge() {
        return age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "age=" + age +
                ", name='" + name + '\'' +
                '}';
    }
}
```

测试：

```java
public static void main(String[] args) {
    List<Person> list = new ArrayList<>();
    list.add(new Person("zs", 17));
    list.add(new Person("ls", 16));
    list.add(new Person("ww", 18));
    System.out.println(list.toString());
    // 实现比较器 Comparator
    Collections.sort(list, new Comparator<Person>() {
        @Override
        public int compare(Person o1, Person o2) {
            return o1.getAge() - o2.getAge();
        }
    });
    System.out.println(list.toString());
}
```

上述也可使用 lambda 表示式

```java
Collections.sort(list, (o1, o2) -> o1.getAge() - o2.getAge());
// or
Collections.sort(list, Comparator.comparingInt(Person::getAge));
```

### <font color=#1FA774>常用工具类</font>

#### <font color=#9933FF>Collections 类</font>

`java.util.Collections`工具类为集合框架提供了很多有用的方法，这些方法都是静态的，在编程中可以直接调用

整个`Collections`工具类源码差不多有 4000 行，这里只针对一些典型的方法进行阐述

**<font color='red'>addAll</font>**

向指定的集合`c`中加入特定的一些元素`elements`

```java
public static <T> boolean addAll(Collection<? super T> c, T... elements);
```

**<font color='red'>binarySearch</font>**

利用二分法在指定的集合中查找元素

```java
// 集合元素 T 实现 Comparable 接口的方式，进行查询
public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key);
// 元素以外部实现 Comparator 接口的方式，进行查询
public static <T> int binarySearch(List<? extends T> list, T key, Comparator<? super T> c);
```

**<font color='red'>sort</font>**

```java
// 集合元素 T 实现 Comparable 接口的方式，进行排序
public static <T extends Comparable<? super T>> void sort(List<T> list);
// 元素以外部实现 Comparator 接口的方式，进行排序
public static <T> void sort(List<T> list, Comparator<? super T> c);
```

**<font color='red'>shuffle</font>**

混排，随机打乱原来的顺序，它打乱在一个`List`中可能有的任何排列的踪迹

```java
// 方法一
public static void shuffle(List<?> list);
// 方法二：指定随机数访问
public static void shuffle(List<?> list, Random rnd);
```

**<font color='red'>reverse</font>**

集合排列反转

```java
// 直接反转集合的元素
public static void reverse(List<?> list);
// 返回可以使集合反转的比较器 Comparator
public static <T> Comparator<T> reverseOrder();
// 集合的反转的反转，如果 cmp 不为 null，返回 cmp 的反转的比较器
// 如果 cmp 为 null，效果等同于第二个方法
public static <T> Comparator<T> reverseOrder(Comparator<T> cmp);
```

**<font color='red'>synchronized 系列</font>**

确保所封装的集合线程安全 (强同步)

```java
// 同步 Collection 接口下的实现类
public static <T> Collection<T> synchronizedCollection(Collection<T> c);
// 同步 SortedSet 接口下的实现类
public static <T> SortedSet<T> synchronizedSortedSet(SortedSet<T> s);
// 同步 List 接口下的实现类
public static <T> List<T> synchronizedList(List<T> list);
// 同步 Map 接口下的实现类
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m);
// 同步 SortedMap 接口下的实现类
public static <K,V> SortedMap<K,V> synchronizedSortedMap(SortedMap<K,V> m);
```

#### <font color=#9933FF>Arrays 类</font>

`java.util.Arrays`工具类也为集合框架提供了很多有用的方法，这些方法都是静态的，在编程中可以直接调用

整个`Arrays`工具类源码有 3000 多行，这里只针对一些典型的方法进行阐述

**<font color='red'>asList</font>**

将一个数组转变成一个`List`，准确来说是`ArrayList`

```java
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}
```

**注意：**这个`List`是定长的，企图添加或者删除数据都会报错`java.lang.UnsupportedOperationException`

**<font color='red'>sort</font>**

对数组进行排序，适合「byte,char,double,float,int,long,short」等基本类型，还有 Object 类型

```java
// 基本数据类型，例子 int 类型数组
public static void sort(int[] a);
// Object 类型数组
// 如果使用 Comparable 进行排序，Object 需要实现 Comparable
// 如果使用 Comparator 进行排序，可以使用外部比较方法实现
public static void sort(Object[] a);
```

**<font color='red'>binarySearch</font>**

通过二分查找法对已排序的数组进行查找

如果数组没有经过`Arrays.sort`排序，那么检索结果未知

适合「byte,char,double,float,int,long,short」等基本类型，还有 Object 类型和泛型

```java
// 基本数据类型，例子 int 类型数组，key 为要查询的参数
public static int binarySearch(int[] a, int key);
// Object 类型数组，key 为要查询的参数
// 如果使用 Comparable 进行排序，Object 需要实现 Comparable
// 如果使用 Comparator 进行排序，可以使用外部比较方法实现
public static int binarySearch(Object[] a, Object key);
// 泛型类型数组，key 为要查询的参数
// 需要传入 Comparator
public static <T> int binarySearch(T[] a, T key, Comparator<? super T> c);
```

**<font color='red'>copyOf</font>**

数组拷贝，底层采用`System.arrayCopy` (native 方法) 实现

适合「byte,char,double,float,int,long,short」等基本类型，还有泛型数组

```java
// 基本数据类型，例子 int 类型数组，newLength 新数组长度
public static int[] copyOf(int[] original, int newLength);
// T 为泛型数组，newLength 新数组长度
public static <T> T[] copyOf(T[] original, int newLength);
```

**<font color='red'>copyOfRange</font>**

数组拷贝，指定一定的范围，底层采用`System.arrayCopy` (native 方法) 实现

适合「byte,char,double,float,int,long,short」等基本类型，还有泛型数组

```java
// 基本数据类型，例子 int 类型数组，from：开始位置，to：结束位置
public static int[] copyOfRange(int[] original, int from, int to);
// T 为泛型数组，from：开始位置，to：结束位置
public static <T> T[] copyOfRange(T[] original, int from, int to);
```

**<font color='red'>equals && deepEquals</font>**

equals：判断两个数组的每一个对应的元素是否相等 (对于两个数组的元素`a`和`a2`有`a == null ? a2 == null : a.equals(a2)`)

```java
// 基本数据类型，例子 int 类型数组，a 为原数组，a2 为目标数组
public static boolean equals(int[] a, int[] a2);
// Object 数组，a 为原数组，a2 为目标数组
public static boolean equals(Object[] a, Object[] a2);
```

deepEquals：主要针对一个数组中的元素还是数组的情况 (多维数组比较)

```java
// Object 数组，a1 为原数组，a2 为目标数组
public static boolean deepEquals(Object[] a1, Object[] a2);
```

**<font color='red'>toString && deepToString</font>**

toString：将数组转换成字符串，中间用逗号隔开

```java
// 基本数据类型，例子 int 类型数组，a 为数组
public static String toString(int[] a);
// Object 数组，a 为数组
public static String toString(Object[] a);
```

deepToString：当数组中又包含数组，就不能单纯的利用`Arrays.toString()`了，使用此方法将数组转换成字符串

```java
// Object 数组，a 为数组
public static String deepToString(Object[] a);
```

### <font color=#1FA774>迭代器</font>

JCF 的迭代器 (Iterator) 为我们提供了遍历容器中元素的方法。只有容器本身清楚容器里元素的组织方式，因此迭代器只能通过容器本身得到。每个容器都会通过内部类的形式实现自己的迭代器

```java
List<String> list = new ArrayList<>();
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    System.out.println(it.next());
}
```

JDK 1.5 引入了增强的`for`循环，简化了迭代容器时的写法

```java
List<String> list = new ArrayList<>();
for (String s : list) {
    System.out.println(s);
}
```