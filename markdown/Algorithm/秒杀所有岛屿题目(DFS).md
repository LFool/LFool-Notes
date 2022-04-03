[toc]

# 秒杀所有岛屿题目(DFS)

> 本质其实就是 DFS（深搜）
>
> DFS 与二叉树的遍历有异曲同工之处

```java
// 递归：「当前节点」「该做什么」「什么时候做」
// FloodFill：如果当前位置是岛屿，则填充为海水
//   - 充当了 visited[] 的作用
private void dfs(int[][] grid, int i, int j) {
    int m = grid.length;
    int n = grid[0].length;
    // 越界检查
    if (i < 0 || i >= m || j < 0 || j >= n) return ;
    // 如果是海水
    if (grid[i][j] == 0) return ;
    // 否则：1 -> 0
    grid[i][j] = 0;
    // 递归处理上下左右
    dfs(grid, i - 1, j); // 上
    dfs(grid, i + 1, j); // 下
    dfs(grid, i, j - 1); // 左
    dfs(grid, i, j + 1); // 右
}
```

## [岛屿数量](https://leetcode-cn.com/problems/number-of-islands/) 

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220108/21010916416468691641646869730ba1HBh.jpeg" alt="1" style="zoom:50%;" />

```java
public int numIslands(char[][] grid) {
    // 记录数量
    int res = 0;
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            // 如果当前位置是岛屿
            // 利用 dfs 将与之相连的所有位置均值为海水
            if (grid[i][j] == '1') {
                res++;
                dfs(grid, i, j);
            }
        }
    }
    return res;
}
private void dfs(int[][] grid, int i, int j) {
    // ...
}
```

## [统计封闭岛屿的数目](https://leetcode-cn.com/problems/number-of-closed-islands/)

> 与岸边相连的岛屿不算封闭岛屿
>
> **思路：**首选除去与岸边相连的岛屿，然后按照正常思路计算
>
> 相似题目：[飞地的数量](https://leetcode-cn.com/problems/number-of-enclaves/)

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220108/21023616416469561641646956056xHe2ME.png" alt="img" style="zoom: 67%;" />

```java
public int closedIsland(int[][] grid) {
    // 除去上下两边
    for (int i = 0; i < grid.length; i++) {
        if (grid[i][0] == 0) dfs(grid, i, 0);
        if (grid[i][grid[0].length - 1] == 0) dfs(grid, i, grid[0].length - 1);
    }
    // 除去左右两边
    for (int j = 0; j < grid[0].length; j++) {
        if (grid[0][j] == 0) dfs(grid, 0, j);
        if (grid[grid.length - 1][j] == 0) dfs(grid, grid.length - 1, j);
    }
    // 正常计算
    int res = 0;
    for (int i = 1; i < grid.length - 1; i++) {
        for (int j = 1; j < grid[0].length - 1; j++) {
            if (grid[i][j] == 0) {
                res++;
                dfs(grid, i, j);
            }
        }
    }
    return res;
}
```

## [岛屿的最大面积](https://leetcode-cn.com/problems/max-area-of-island/)

```java
private int sum = 0;
public int maxAreaOfIsland(int[][] grid) {
    int res = 0;
    int m = grid.length;
    int n = grid[0].length;
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == 1) {
                sum = 0;
                dfs(grid, i, j);
                // 取最值
                res = Math.max(res, sum);
            }
        }
    }
    return res;
}
private void dfs(int[][] grid, int i, int j) {
    int m = grid.length;
    int n = grid[0].length;
    if (i < 0 || i >= m || j < 0 || j >= n) return ;
    if (grid[i][j] == 0) return ;
    grid[i][j] = 0;
    sum ++;
    dfs(grid, i + 1, j);
    dfs(grid, i - 1, j);
    dfs(grid, i, j + 1);
    dfs(grid, i, j - 1);
}
```

## [统计子岛屿](https://leetcode-cn.com/problems/count-sub-islands/)

> 子岛屿概念：如果 B 某一位置是岛屿，A 相同位置也必须是岛屿
>
> 解题思路：如果 B 是岛屿，但 A 不是岛屿，则对 B 进行 FloodFill

![img](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220108/21114216416475021641647502296vX56tU.png)

```java
public int countSubIslands(int[][] grid1, int[][] grid2) {
    int m = grid2.length;
    int n = grid2[0].length;
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid2[i][j] == 1 && grid1[i][j] != 1) {
                dfs(grid2, i, j);
            }
        }
    }
    int res = 0;
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid2[i][j] == 1) {
                res++;
                dfs(grid2, i, j);
            }
        }
    }
    return res;
}
private void dfs(int[][] grid, int i, int j) {
    // ...
}
```

## [不同岛屿数量](https://leetcode-cn.com/problems/number-of-distinct-islands/)

> 思路：对岛屿进行序列化，存入 HashSet 中