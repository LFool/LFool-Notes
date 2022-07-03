# 🥹含泪总结周赛中的两道「DP」问题

[6109. 知道秘密的人数](https://leetcode.cn/problems/number-of-people-aware-of-a-secret/)

[6110. 网格图中递增路径的数目](https://leetcode.cn/problems/number-of-increasing-paths-in-a-grid/)

### <font color=#1FA774>知道秘密的人数</font>

**题目详情可见 [知道秘密的人数](https://leetcode.cn/problems/number-of-people-aware-of-a-secret/)**

比赛的时候一直在思考状态转移，自己心中想的`dp[i]`含义是「第`i`天知道秘密的总人数」，一直不知道怎么推出状态转移

直到看到别人的定义：`dp[i]`表示第`i`天**<font color='red'>新</font>**知道秘密的人数，瞬间就懂了！！！

所以第`i`天新增的人数都来自`(i - forget, i - delay]`区间内的人的分享，故有：$dp[i] = \sum\limits^{i - delay}_{j = i - forget + 1}dp[j]$

对于每次求`dp[i]`都有一个区间内的累加过程，所以可以利用「前缀和」优化一波！！

```java
public int peopleAwareOfSecret(int n, int delay, int forget) {
    int mod = (int) 1e9 + 7;
    long[] f = new long[n + 1];
    long[] preSum = new long[n + 1];
    f[1] = 1;
    preSum[1] = 1;
    for (int i = 2; i <= n; i++) {
        int l = Math.max(0, i - forget);
        int r = Math.max(0, i - delay);
        // 注意：对于相减的求余需要加上一个 mod，转化成正数范围内！！
        f[i] = (preSum[r] - preSum[l] + mod) % mod;
        preSum[i] = (preSum[i - 1] + f[i]) % mod;
    }
    return (int) (preSum[n] - preSum[Math.max(0, n - forget)] + mod) % mod;
}
```

### <font color=#1FA774>网格图中递增路径的数目</font>

**题目详情可见 [网格图中递增路径的数目](https://leetcode.cn/problems/number-of-increasing-paths-in-a-grid/)**

这个题目就是一个「记忆化搜索」，为了避免路径重复，可以求出以某一个点为结尾的所有路径，所有把所有点都遍历一遍即可！

```java
private int m, n;
private int[][] grid;
private int[][] emeo;
private int mod = (int) 1e9 + 7;
private int[][] dirs = new int[][] { {-1, 0}, {1, 0}, {0, -1}, {0, 1} };
public int countPaths(int[][] grid) {
    int ans = 0;
    this.grid = grid;
    this.m = grid.length;
    this.n = grid[0].length;
    emeo = new int[m][n];
    // 遍历所有点
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            ans = (ans + dp(i, j)) % mod;
        }
    }
    return ans;
}
// 以点 (i, j) 结尾的所有路径
private int dp(int i, int j) {
    // 已经求过了，直接返回
    if (emeo[i][j] != 0) return emeo[i][j];
    long ans = 1L;
    // 往四个方向递归遍历
    for (int[] dir : dirs) {
        int ii = i + dir[0], jj = j + dir[1];
        if (ii >= 0 && ii < m && jj >= 0 && jj < n && grid[ii][jj] < grid[i][j]) {
            ans = (ans + dp(ii, jj)) % mod;
        }
    }
    // 保存结果
    emeo[i][j] = (int) ans;
    return emeo[i][j];
}
```