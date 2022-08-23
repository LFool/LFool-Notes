# BFS 算法框架

[111. 二叉树的最小深度](https://leetcode-cn.com/problems/minimum-depth-of-binary-tree/)

[752. 打开转盘锁](https://leetcode-cn.com/problems/open-the-lock/)



前文讲了 **[回溯(DFS) 算法框架](./回溯(DFS).html)**，算法核心可以大概总结为：不到南墙不回头

如果说 DFS 是树的递归遍历，那么 BFS 对应的就是树的层序遍历，一层一层的往下遍历

### <font color=#1FA774>层序遍历</font>

如下图所示，树的层次遍历就是一层一层的往下遍历

![4](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220418/2020401650284440NwZ7OM4.svg)

具体代码如下所示：

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int size = q.size();
        List<Integer> list = new ArrayList<>();
        // 完整的一个循环即表示一层
        for (int i = 0; i < size; i++) {
            TreeNode cur = q.poll();
            list.add(cur.val);
            if (cur.left != null) q.offer(cur.left);
            if (cur.right != null) q.offer(cur.right);
        }
        res.add(list);
    }
    return res;
}
```

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16402516458648251645864825235pN5IJY.png" alt="image-20220226164025008" style="zoom:18%;" /> 注：层次遍历是可以携带层级数据，即可以知道每次处理的是第几层

利用上面的特点我们可以求一些最值问题，如：**[二叉树的最小深度](https://leetcode-cn.com/problems/minimum-depth-of-binary-tree/)**



<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220824/0005501661270750NbXWbT9.svg" alt="9" style="zoom:50%;" />

```java
public int minDepth(TreeNode root) {
    if (root == null) return 0;
    // 记录层次信息
    int depth = 0;
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int sz = q.size();
        // 更新 depth
        depth++;
        for (int i = 0; i < sz; i++) {
            TreeNode cur = q.poll();
            if (cur.left != null) q.offer(cur.left);
            if (cur.right != null) q.offer(cur.right);
            // 如果是叶子节点，即可以得知该节点的深度最小
            // 原因：层次遍历是从上到下，可以保证首次遇到的叶子节点的深度是最小的
            if (cur.left == null && cur.right == null) return depth;
        }
    }
    return depth;
}
```

### <font color=#1FA774>打开转盘锁</font>

这个题目，我们可以抽象成一棵树，如下图所示：

![5](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220418/2037021650285422cDCA4G5.svg)

对于每一组 4 位数字，都有 8 种变换

因为存在可能重复的 4 位数字，所以我们需要记录每组数字的访问情况

由于题目有一个「死亡数字」，所以我们遇到「死亡数字」，需要跳过

综上：我们可以很容易知道剪枝的情况，即：

- 访问「访问过的数字」时，跳过
- 访问「死亡数字」时，跳过

我们先来实现一些小功能：得到某一位数字加减 1 后的数字

```java
// +1
private String increaseOne(String s, int i) {
    char[] ch = s.toCharArray();
    if (ch[i] == '9') ch[i] = '0';
    else ch[i] += 1;
    return new String(ch);
}
// -1
private String decreaseOne(String s, int i) {
    char[] ch = s.toCharArray();
    if (ch[i] == '0') ch[i] = '9';
    else ch[i] -= 1;
    return new String(ch);
}
```

我们现在可以根据思路写出如下代码：

```java
public int openLock(String[] deadends, String target) {
    // 记录死亡数字，方便快速判断
    Set<String> deadendSet = new HashSet<>();
    for (String deadend : deadends) deadendSet.add(deadend);
    // 和层次遍历中的队列作用一样
    Queue<String> q = new LinkedList<>();
    // 记录访问情况
    Set<String> visited = new HashSet<>();
    int step = 0;
    // 添加起始数字
    q.offer("0000");
    // 标记为已访问
    visited.add("0000");
    while (!q.isEmpty()) {
        int sz = q.size();
        // 一层一层的遍历
        for (int i = 0; i < sz; i++) {
            String cur = q.poll();
            // 访问「死亡数字」时，跳过
            if (deadendSet.contains(cur)) continue;
            // 找到目标，结束
            if (cur.equals(target)) return step;
            // 存储孩子节点到队列中，分 4 次
            for (int j = 0; j < 4; j++) {
                // +1
                String pushOne = increaseOne(cur, j);
                // -1
                String minusOne = decreaseOne(cur, j);
                // 如果没有访问过
                if (!visited.contains(pushOne)) {
                    // 加入队列
                    q.offer(pushOne);
                    // 标记为已访问
                    visited.add(pushOne);
                }
                // 同上
                if (!visited.contains(minusOne)) {
                    q.offer(minusOne);
                    visited.add(minusOne);
                }
            }
        }
        step++;
    }
    return -1;
}
```

### <font color=#1FA774>打开转盘锁优化</font>

在讲优化之前，我们先看看上面代码的执行过程：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220418/2125201650288320KqYXP96.svg" alt="6" style="zoom:67%;" />

可以明显看到，未优化的版本从`start`开始一层一层的向下遍历，直到遇到`target`节点停止

**<font color='red'>优化思路：双向 BFS (双向奔赴 哈哈哈哈哈)</font>**

执行过程如下图所示：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220418/2157121650290232AzIJuv7.svg" alt="7" style="zoom:67%;" />

相比于单向的 BFS，双向 BFS 只遍历了半颗树就得到了结果

**<font color='red'>但是双向 BFS 有一个缺点：必须知道终点才可以用</font>**

代码如下：

```java
public int openLock(String[] deadends, String target) {
    Set<String> deadendSet = new HashSet<>();
    for (String deadend : deadends) deadendSet.add(deadend);
    // 存放 start 往下遍历的结果
    Set<String> q1 = new HashSet<>();
    // 存放 target 往上遍历的结果
    Set<String> q2 = new HashSet<>();
    Set<String> visited = new HashSet<>();
    int step = 0;
    // 加入初始值
    q1.add("0000");
    q2.add(target);
    while (!q1.isEmpty() && !q2.isEmpty()) {
        // 存放临时结果 (下一层的节点)
        Set<String> temp = new HashSet<>();
        for (String cur : q1) {
            if (deadendSet.contains(cur)) continue;
            if (q2.contains(cur)) return step;
            // 标记为已访问
            visited.add(cur);
            for (int i = 0; i < 4; i++) {
                String plusOne = increaseOne(cur, i);
                String minusOne = decreaseOne(cur, i);
                if (!visited.contains(plusOne)) temp.add(plusOne);
                if (!visited.contains(minusOne)) temp.add(minusOne);
            }
        }
        // 交换 q1 q2
        // q1 && q2 交替遍历
        q1 = q2;
        q2 = temp;
        step++;
    }
    return -1;
}
```

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220226/16402516458648251645864825235pN5IJY.png" alt="image-20220226164025008" style="zoom:18%;" /> 注：优化版的「判断节点是否已经访问」以及「加入访问集合中」的顺序有变化