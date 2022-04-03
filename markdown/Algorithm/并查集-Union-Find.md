[toc]

# 并查集（Union-Find）

[990. 等式方程的可满足性](https://leetcode-cn.com/problems/satisfiability-of-equality-equations/)

[130. 被围绕的区域](https://leetcode-cn.com/problems/surrounded-regions/)



顾名思义，并查集是来解决图的连通性问题

- Union -- 连接两个节点

- Find  -- 查找所属的连通分量

所以，并查集主要就是实现以下接口：

```java
class UF {
    /* 将 p 和 q 连接 */
    public void union(int p, int q);
    /* 判断 p 和 q 是否连通 */
    public boolean connected(int p, int q);
    /* 返回图中有多少个连通分量 */
    public int count();
    
    /* 返回当前节点的根节点 */
    private int find(int x);
}
```

### <font color=#1FA774>存储数据结构</font>

**<font color='red'>如何表示节点与节点之间的连通性关系呢？？</font>**

- 如果`p`和`q`连通，则它们有相同的根节点

用数组`parent[]`来表示这种关系

- 如果自己就是根节点，那么`parent[i] = i`，即自己指向自己

- 如果自己不是根节点，则`parent[i] = root id`

```java
private int count;
private int[] parent;
// 构造函数
public UF (int n) {
    this.count = n;
    parent = new int[n];
    for (int i = 0; i < n; i++) {
        // 最初，每个节点均是独立的
        parent[i] = i;
    }
}
```

### <font color=#1FA774>Union 方法</font>

**<font color='red'>介绍了存储的数据结构，那如何把两个节点连接起来呢？？</font>**

很简单，只需将其中任一一个节点的根节点指向另一个节点的根节点即可

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220402/2156381648907798FMZz6W1234.jpg" alt="1234" style="zoom:33%;" />

```java
// 伪代码
public void union(int p, int q) {
    // 找到 p 的根节点 rootP
    // 找到 q 的根节点 rootQ
    // 如果已经在同一个连通分中，跳过
    // parent[rootP] = rootQ
    // 或 parent[rootQ] = rootP
}
```

现在的问题就变成了如何快速找到某一个节点的根节点！！

刚刚介绍数据结构的时候，强调了根节点的特点，即自己指向自己

```java
private int find(int x) {
    while (x != parent[x]) {
        x = parent[x];
    }
    return x;
}
```

如何，是不是很简单，哈哈哈哈哈哈

### <font color=#1FA774>connected() && count()</font>

这两个方法的实现很简单

```java
public boolean connected(int p, int q) {
    int rootP = find(p);
    int rootQ = find(q);
    return rootP == rootQ;
}
```

`count()`需要维护一个全局变量，来记录图的连通分量的数量

另外，我们需要明确的是：只有在调用`union()`方法时，才可能改变连通分量的数量

```java
public void union(int p, int q) {
    int rootP = find(p);
    int rootQ = find(q);
    if (rootP == rootQ) return;
    parent[rootP] = rootQ;
    // 连通分量 -1
    count--;
}
public int count() {
    return this.count;
}
```

### <font color=#1FA774>瓶颈分析</font>

行文至此，已经把并查集的所有接口实现。但这远远不够，因为此时的代码还不完美，时间复杂度可能会很高

分析上述实现的方法，`find()`是决定并查集时间复杂度的重要因素。抛开`find()`因素，其他方法的时间复杂度均可视为`O(1)`。所以如果要优化算法的时间复杂度，需要用`find()`入手



对于有 n 个节点 1 个连通分量的并查集来说，最坏的时间复杂度为 `O(n)`，最好的时间复杂度为`O(1)`

- 最坏情况：全部只有左孩子

- 最好情况：n - 1 叉树，即根节点有 n - 1 个孩子

### <font color=#1FA774>优化角度 1：平衡性优化</font>

**<font color='red'>思路：当我们每次连接两个节点的时候，不希望出现头重脚轻的情况，而希望到达一种平衡的状态</font>**

使用额外的一个数组`size[]`记录每个连通分量中的节点数，每次均把节点数少的分量接到节点数多的分量上，如图

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220402/2156181648907778aOyUAs123.jpg" alt="123" style="zoom:33%;" />

**<font color='red'>注意：只有每个连通分量的根节点的 size[] 才可以代表该连通分量中的节点数</font>**

```java
private int count;
private int[] parent;
private int[] size;
// 构造函数
public UF (int n) {
    this.count = n;
    parent = new int[n];
    size = new int[n];
    for (int i = 0; i < n; i++) {
        parent[i] = i;
        // 最初，每个连通分量均为 1
        size[i] = 1;
    }
}
public void union(int p, int q) {
    int rootP = find(p);
    int rootQ = find(q);
    if (rootP == rootQ) return;
    /******** 修改部分 ********/
    if (size[rootP] < size[rootQ]) {
        parent[rootP] = rootQ;
        size[rootQ] += size[rootP]
    } else {
        parent[rootQ] = rootP;
        size[rootP] += size[rootQ]
    }
    /********** end **********/
    count--;
}
```

### <font color=#1FA774>优化角度 2：路径压缩</font>

**<font color='red'>思路：使树高始终保持为常数</font>**

```java
private int find(int x) {
    while (parent[x] != x) {
        // 进行路径压缩
        parent[x] = parent[parent[x]];
        x = parent[x];
    }
    return x;
}
```

**<font color='red'>注意：</font>**

- 当使用了路径压缩优化后，平衡性优化可不使用
- 路径压缩优化比平衡性优化更为常用
- 但是可在某些题目中借鉴平衡性优化的思想，如 [128. 最长连续序列](https://leetcode-cn.com/problems/longest-consecutive-sequence/)

### <font color=#1FA774>完整模版</font>

```java
class UF {
    private int count;
    private int[] parent;
    private int[] size;
    public UF(int n) {
        this.count = n;
        parent = new int[n];
        size = new int[n];
        for (int i = 0; i < n; i++) {
            parent[i] = i;
            size[i] = 1;
        }
    }
    public void union(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ) return ;
        // 平衡性优化
        if (size[rootP] < size[rootQ]) {
            parent[rootP] = rootQ;
            size[rootQ] += size[rootP];
        } else {
            parent[rootQ] = rootP;
            size[rootP] += size[rootQ];
        }
    }
    public boolean conneted(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        return rootP == rootQ;
    }
    public int count() {
        return this.count;
    }
    private int find(int x) {
        while (x != parent[x]) {
            // 路径压缩
            parent[x] = parent[parent[x]];
            x = parent[x];
        }
        return x;
    }
}
```

### <font color=#1FA774>实战题目</font>

#### [128. 最长连续序列](https://leetcode-cn.com/problems/longest-consecutive-sequence/)

**<font color='red'>注意：size 别写反了！！！！🩸的教训</font>**

**亮点**

- 利用`Map`进行了一个「下标」和「值」的对应
- 利用`Map`进行重复元素的排除
- 利用`Map`可快速判断当前并查集中已有元素
- 将`num[i]`和`num[i] - 1 && num[i] + 1`相连

```java
class Solution {
    public int longestConsecutive(int[] nums) {

        Map<Integer, Integer> map = new HashMap<>();
        UF uf = new UF(nums.length);

        for (int i = 0; i < nums.length; i++) {
            // 存在重复元素，跳过
            if (map.containsKey(nums[i])) continue;

            if (map.containsKey(nums[i] - 1)) {
                uf.union(i, map.get(nums[i] - 1));
            }
            if (map.containsKey(nums[i] + 1)) {
                uf.union(i, map.get(nums[i] + 1));
            }
            map.put(nums[i], i);        
        }
        return uf.getMaxConnectSize();
    }
}
class UF {
    private int[] parent;
    private int[] size;

    public UF(int n) {
        parent = new int[n];
        size = new int[n];
        for (int i = 0; i < n; i++) {
            parent[i] = i;
            size[i] = 1;
        }
    }
    public void union(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ) return;
        parent[rootP] = rootQ;
        // 注意 别写反了
        size[rootQ] += size[rootP];
    }
    // get root id
    private int find(int x) {
        while (x != parent[x]) {
            parent[x] = parent[parent[x]];
            x = parent[x];
        }
        return x;
    }
    public int getMaxConnectSize() {
        int maxSize = 0;
        for (int i = 0; i < parent.length; i++) {
            if (i == parent[i]) {
                maxSize = Math.max(maxSize, size[i]);
            }
        }
        return maxSize;
    }
}
```

