# Java 8 Stream sorted()

介绍`Stream sorted()`三种排序的实现「自然排序」「比较器」「反向排序」

`sorted()`：它使用自然排序对流的元素进行排序。元素类必须实现`Comparable`接口

`sorted(Comparator<? super T> comparator)`：这里我们使用`lambda`表达式创建`Comparator`的实例。我们可以按升序和降序对流元素进行排序

以下代码行将按自然顺序对列表进行排序

```java
list.stream().sorted() 
```

为了反转自然排序`Comparator`提供了`reverseOrder()`方法。我们使用它如下

```java
list.stream().sorted(Comparator.reverseOrder()) 
```

以下代码行使用`Comparator`对列表进行排序

```java
list.stream().sorted(Comparator.comparing(Student::getAge)) 
```

为了反转顺序，`Comparator`提供了`reversed()`方法。我们使用这种方法如下

```java
list.stream().sorted(Comparator.comparing(Student::getAge).reversed()) 
```

### <font color=#1FA774>Stream sorted() with List</font>

```java
// 升序
// 前提：Person 类必须实现 Comparable 接口
List<Person> list = new ArrayList<>();
List<Person> sortedList = new ArrayList<>();
list.stream().sorted().forEach(sortedList::add);

// 降序
// 前提：Person 类必须实现 Comparable 接口
list.stream().sorted(Comparator.reverseOrder()).forEach(sortedList::add);

// 根据年龄升序
// Person 类不需要实现 Comparable 接口
list.stream().sorted(Comparator.comparing(Person::getAge)).forEach(sortedList::add);
// 根据年龄降序
list.stream().sorted(Comparator.comparing(Person::getAge).reversed()).forEach(sortedList::add);

// lambda 表达式实现 Comparator 接口
list.stream().sorted((o1, o2) -> {
    if (o1.getAge() == o2.getAge()) return o1.getName().compareTo(o2.getName());
    return o1.getAge() - o2.getAge();
}).forEach(sortedList::add);
```

### <font color=#1FA774>Stream sorted() with Set</font>

```java
Set<Person> set = new HashSet<>();
List<Person> sorted = set.stream().sorted().collect(Collectors.toList());
System.out.println(sorted);
// 和 List 几乎一模一样
```

需要注意的一点是，不能把排序后的结果重新输出到另一个`Set`中，因为`Set`无顺序，会破坏顺序

### <font color=#1FA774>Stream sorted() with Map</font>

`Map`同`Set`一样无顺序，如果把排序后的结果重新输出到另一个`Map`中中，可能会导致顺序被破坏

我们可以使用`LinkedHashMap`，这样会保证顺序

```java
// 前提：key 对应的类必须实现 Comparable 接口
Map<Integer, Person> map = new LinkedHashMap<>();
Map<Integer, Person> sortedMap = new LinkedHashMap<>();
map.entrySet().stream().sorted(Map.Entry.comparingByKey()).forEachOrdered(e -> sortedMap.put(e.getKey(), e.getValue()));

// 前提：value 对应的类必须实现 Comparable 接口
map.entrySet().stream().sorted(Map.Entry.comparingByValue()).forEachOrdered(e -> sortedMap.put(e.getKey(), e.getValue()));

// lambda 表达式实现 Comparator 接口
// value 对应的类不需要实现 Comparable 接口
map.entrySet().stream().sorted(((o1, o2) -> {
    if (o1.getValue().getAge() == o2.getValue().getAge()) return o2.getValue().getName().compareTo(o1.getValue().getName());
    return o1.getValue().getAge() - o2.getValue().getAge();
})).forEachOrdered(e -> sortedMap.put(e.getKey(), e.getValue()));
```