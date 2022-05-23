# 浅析：无权值最短路径算法（BFS）

[675. 为高尔夫比赛砍树](https://leetcode.cn/problems/cut-off-trees-for-golf-event/)



本篇文章介绍一种简单类型的「最短路径」算法

提到「最短路径」算法，首先可以想到的肯定是「Dijkstra 算法」，但是「Dijkstra 算法」一般适用于<font color='red'>有权值且权值为正</font>的情况。**关于「Dijkstra 算法」详细实现可见 [最短路径-Dijkstra](./最短路径-Dijkstra.html)**

而我们今天要介绍的题目是无权值的类型，所以如果使用「Dijkstra 算法」，就略显复杂了！！

同时，对于「Dijkstra 算法」，一般计算的是其他所有点到起点的距离；而有些题目仅仅只需要计算两个点的距离而已！

### <font color=#1FA774>问题分析</font>

对于今天的问题，有两个前提条件「没有两棵树的高度是相同的」且「需要按照树的高度从低向高砍掉所有的树」

所以我们只需要对所有树的高度升序排列，然后依次计算两点之间的距离即可！！最后我们的问题就抽象成「计算两点之间的距离」

整理一下这个问题的特点

- 边与边之间无权值
- 只需求两点间的距离

我们可以使用 BFS 一层一层的向外扩张，每扩张一层，距离 ➕1。如下图：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220523/1619301653293970dkiJP22.svg" alt="2" style="zoom:67%;" />

当我们在向外扩张时，记录一下层数，就可以得到两点之间的距离

### <font color=#1FA774>代码实现</font>

```java
class Solution {
    private int m;
    private int n;
    private int[][] graph;
    // 记录所有的树，其三元组为 [h, i, j]
    private List<int[]> tree = new ArrayList<>();
    private int[][] dirs = new int[][]{{0,1},{0,-1},{1,0},{-1,0}};
    public int cutOffTree(List<List<Integer>> forest) {
        m = forest.size();
        n = forest.get(0).size();
        graph = new int[m][n];
        if (forest.get(0).get(0) == 0) return -1;
        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) {
                graph[i][j] = forest.get(i).get(j);
                // 加入 tree 中
                if (graph[i][j] > 1) tree.add(new int[]{graph[i][j], i, j});
            }
        }
        // 根据 h 排序
        Collections.sort(tree, Comparator.comparingInt(o -> o[0]));
        int ans = 0;
        int x = 0, y = 0;
        for (int[] t : tree) {
            int nx = t[1], ny = t[2];
            int dist = bfs(x, y, nx, ny);
            if (dist == -1) return -1;
            ans += dist;
            x = nx;
            y = ny;
        }
        return ans;
    }
    private int bfs(int X, int Y, int P, int Q) {
        if (X == P && Y == Q) return 0;
        // 记录节点访问情况
        boolean[][] visited = new boolean[m][n];
        // 双端队列
        Deque<int[]> deque = new ArrayDeque<>();
        deque.addLast(new int[]{X, Y});
        visited[X][Y] = true;
        int dist = 0;
        while (!deque.isEmpty()) {
            int size = deque.size();
            for (int i = 0; i < size; i++) {
                int[] cur = deque.pollFirst();
                int x = cur[0], y = cur[1];
                for (int[] dir : dirs) {
                    int nx = x + dir[0], ny = y + dir[1];
                    // 非法情况
                    if (nx < 0 || nx >= m || ny < 0 || ny >= n || graph[nx][ny] == 0 || visited[nx][ny]) continue;
                    // 达到终点
                    if (nx == P && ny == Q) return dist + 1;
                    deque.addLast(new int[]{nx, ny});
                    visited[nx][ny] = true;
                }
            }
            dist++;
        }
        return -1;
    }
}
```

