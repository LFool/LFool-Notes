# BFS 算法秒杀数字华容道

[773. 滑动谜题](https://leetcode.cn/problems/sliding-puzzle/)

 

本篇文章介绍如何用「BFS」解决数字华容道问题，**详情可见 [滑动谜题](https://leetcode.cn/problems/sliding-puzzle/)**

这个题目还算友好，规模是固定的 2 x 3

对于这种计算最小步数的问题，我们就要敏感地想到 BFS 算法，这里再列举几个题目：**[打开转盘锁](https://leetcode-cn.com/problems/open-the-lock/)**、**[最小基因变化](https://leetcode-cn.com/problems/minimum-genetic-mutation/)** 都可以看作求最小步数的问题，均是使用 BFS

**关于 BFS 框架的详细总结可见 [BFS 算法框架](./BFS.html)**

**关于「最小基因变化」的题解可见 [浅析：最小基因变化](./浅析：最小基因变化.html)**



我们需要搞清楚，每次可以做的选择是什么？如下图所示，对于这种情况，可以有三种选择：

![10](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/18344316581404833p6Znu10.svg)

我们的的终点是什么：

![13](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/1836371658140597LzkjKo13.svg)

现在的问题是，如何才可以把二维数组转换成更好处理的形式 -> 字符串

首先我们肯定需要把二维数组**拉平**，如下图所示：

![14](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/1840021658140802xS0rUu14.svg)

拉平后，如何找到每次可做的选择呢？

![15](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220718/1843431658141023rasEHa15.svg)

对于上图，下标为 4 的位置，可做的选择为`[1, 3, 5]`

由于规模是固定的，所以可以手动计算出拉平后每个位置的选择

```java
// neighbor[i] 表示下标为 i 的位置可做的选择
int[][] neighbor = new int[][]{
    {1, 3},
    {0, 4, 2},
    {1, 5},
    {0, 4},
    {3, 1, 5},
    {4, 2}
};
```

**<font color='red'>(PS：居然现在才知道二维数组中一维的大小可以不同，一直以为必须相同)</font>**

下面给出完整代码：

```java
public int slidingPuzzle(int[][] board) {
    int m = 2, n = 3;
    // 终点
    String target = "123450";
    // 起点
    StringBuffer sb = new StringBuffer();
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            sb.append(board[i][j]);
        }
    }
    String start = sb.toString();
    int[][] neighbor = new int[][]{
        {1, 3},
        {0, 4, 2},
        {1, 5},
        {0, 4},
        {3, 1, 5},
        {4, 2}
    };
    int step = 0;
    Queue<String> q = new LinkedList<>();
    Set<String> vis = new HashSet<>();
    q.offer(start);
    vis.add(start);
    while (!q.isEmpty()) {
        int size = q.size();
        for (int i = 0; i < size; i++) {
            String cur = q.poll();
            if (cur.equals(target)) return step;
            // 找到 0 的位置
            int idx = 0;
            for (; cur.charAt(idx) != '0'; idx++);
            // 把邻居加入队列
            for (int adj : neighbor[idx]) {
                String next = swap(cur, idx, adj);
                if (!vis.contains(next)) {
                    q.offer(next);
                    vis.add(next);
                }
            }
        }
        step++;
    }
    return -1;
}
// 交换
private String swap(String s, int i, int j) {
    char[] c = s.toCharArray();
    char t = c[i];
    c[i] = c[j];
    c[j] = t;
    return new String(c);
}
```

