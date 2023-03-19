# 手撸 LRU

[146. LRU 缓存](https://leetcode-cn.com/problems/lru-cache/)



**LRU：Least Recently Used (最近最少使用优先)**

这是一种最近最少使用优先的淘汰算法，顾名思义优先把最近最少使用的任务淘汰掉 (感觉说了一个废话。。。)

下面直接举一个详细的例子说明：

> 缓存区大小：3
>
> 任务池大小：5
>
> 时刻一：启动任务 1，此时任务 1 加入缓存区
>
> - 缓存区内容：<kbd>任务 1</kbd>
>
> 时刻二：启动任务 2，此时任务 2 加入缓存区
>
> - 缓存区内容：<kbd>任务 2</kbd> -> <kbd>任务 1</kbd>
>
> 时刻三：调用任务 1
>
> - 缓存区内容：<kbd>任务 1</kbd> -> <kbd>任务 2</kbd>
>
> 时刻四：启动任务 3，此时任务 3 加入缓存区
>
> - 缓存区内容：<kbd>任务 3</kbd> -> <kbd>任务 1</kbd> -> <kbd>任务 2</kbd>
>
> 时刻五：启动任务 4，此时缓存区已满，任务 4 加入缓存区之前需要**淘汰最近最少使用的任务**，即任务 2
>
> - 缓存区内容：<kbd>任务 4</kbd> -> <kbd>任务 3</kbd> -> <kbd>任务 1</kbd>
>
> 时刻六：调用任务 1
>
> - 缓存区内容：<kbd>任务 1</kbd> -> <kbd>任务 4</kbd> -> <kbd>任务 3</kbd>
>
> 时刻七：启动任务 5，此时缓存区已满，任务 5 加入缓存区之前需要**淘汰最近最少使用的任务**，即任务 3
>
> - 缓存区内容：<kbd>任务 5</kbd> -> <kbd>任务 1</kbd> -> <kbd>任务 4</kbd>

相信上面的流程已经非常情况了，那么如何实现这种效果呢？？

而且，「加入新任务、调用任务、淘汰任务」都需要时间复杂度为 O(1)

**<font color='red'>解决方案：双向链表 ➕ Hash 表</font>**

- 双向链表：由于有`prev`和`next`节点，所以如果定位到某一个节点后，可快速删除该节点
- Hash 表：存储 key 和 Node 的映射，通过 key 可快速获得 Node

**我们规定：**双向链表的头节点方向为最近最少使用的节点，而每次新加入节点都插入到尾节点

OK，下面开始构造「双向链表」的数据结构

```java
// 双向链表节点的数据结构
public class Node {
    // 存储 key，value (均视为 int 类型)
    public int key, value;
    // 前节点 && 下一节点
    public Node prev, next;
    
    public Node(int key, int value) {
        this.key = key;
        this.value = value;
    }
}

// 双向链表的数据结构
public class DoubleList {
    // 头尾虚拟节点 (不存储实际内容，仅仅表示头尾节点)
    private Node head, tail;
    // 双向链表大小
    private int size;
    
    public DoubleList() {
        this.head = new Node(0, 0);
        this.tail = new Node(0, 0);
        // 头尾相连形成一个空双向链表
        head.next = tail;
        tail.prev = head;
    }
    
    // 删除节点 x
    public void remove(Node x) {
        x.prev.next = x.next;
        x.next.prev = x.prev;
        size--;
    }
    
    // 添加节点 x 到表尾
    public void addLast(Node x) {
        x.next = tail;
        x.prev = tail.prev;
        tail.prev.next = x;
        tail.prev = x;
        size++;
    }
    
    // 删除头节点后的第一个节点
    public Node removeFirst() {
        // 如果链表为 null
        if (head.next == tail) return null;
        Node first = head.next;
        // 调用 remove()
        remove(first);
        return first;
    }
    
    // 返回链表大小
    public int size() {
        return this.size;
    }
}
```

好了，基础的数据结构构造好了后，我们开始今天的**核心部分**

```java
public class LRUCache {
    private Map<Integer, Node> map;
    private DoubleList cache;
    private int capacity;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        map = new HashMap<>();
        cache = new DoubleList();
    }
    
    // 调用 key 任务
    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        makeRecently(key);
        return map.get(key).value;
    }
    
    // 添加 (key, value) 任务
    // 如果存在：删除后，添加 (key, value) 到表尾
    // 如果不存在：判断 capacity 是否够用，如果不够，调用 deleteLeastRecently()
    public void put(int key, int value) {
        if (map.containsKey(key)) {
            deleteKey(key);
            addRecently(key, value);
            return ;
        }
        if (cache.size() == capacity) deleteLeastRecently();
        addRecently(key, value);
    }
    
    // 使 cache 中的任务成为「最近使用」
    // 即：移动到表尾
    private void makeRecently(int key) {
        Node x = map.get(key);
        cache.remove(x);
        cache.addLast(x);
    }
    
    // 添加 (key, value) 为「最近使用」
    private void addRecently(int key, int value) {
        Node x = new Node(key, value);
        map.put(key, x);
        cache.addLast(x);
    }
    
    // 删除 key 的任务
    private void deleteKey(int key) {
        Node deleteNode = map.get(key);
        cache.remove(deleteNode);
        map.remove(key);
    }
    
    // 删除 cache 中「最近最少使用任务」
    private void deleteLeastRecently() {
        Node deleteNode = cache.removeFirst();
        map.remove(deleteNode.key);
    }
}
```



上面都是纯手动实现的 API，下面给出借助`LinkedHashMap`实现的代码

```java
class LRUCache {

    private int capacity;
    private LinkedHashMap<Integer, Integer> cache;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        cache = new LinkedHashMap<>();
    }

    public int get(int key) {
        if (!cache.containsKey(key)) {
            return -1;
        }
        makeRecently(key);
        return cache.get(key);
    }

    public void put(int key, int value) {
        if (cache.containsKey(key)) {
            cache.put(key, value);
            makeRecently(key);
            return ;
        }
        if (cache.size() >= capacity) {
            int leastUsed = cache.keySet().iterator().next();
            cache.remove(leastUsed);
        }
        cache.put(key, value);
    }

    private void makeRecently(int key) {
        int val = cache.get(key);
        cache.remove(key);
        cache.put(key, val);
    }
}
```

