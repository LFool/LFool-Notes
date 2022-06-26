# 🥹含泪总结周赛中的三道「DP」问题

[6100. 统计放置房子的方式数](https://leetcode.cn/problems/count-number-of-ways-to-place-houses/)

[6107. 不同骰子序列的数目](https://leetcode.cn/problems/number-of-distinct-roll-sequences/)

[5229. 拼接数组的最大分数](https://leetcode.cn/problems/maximum-score-of-spliced-array/)

### <font color=#1FA774>统计放置房子的方式数</font>

**题目详情可见 [统计放置房子的方式数](https://leetcode.cn/problems/count-number-of-ways-to-place-houses/)**

**<font color='red'>对于计数问题，大多数都是用「动态规划」</font>**

本题在理解上不难，就是求出街道一边所有符合要求的排列数量。同理，另一边排列数量是相等的！

但是问题就是「如何求所有符合要求的排列数量」(折磨了好久)

这里给出`dp[][]`数组的定义：

- `dp[i][1]`表示第`i`个位置放置一所房子的所有符合要求的排列数量
- `dp[i][0]`表示第`i`个位置不放置一所房子的所有符合要求的排列数量

那么「状态转移方程」是什么呢？

- 对于第`i`个位置<font color='red'>放置</font>一所房子，那么第`i - 1`个位置<font color='red'>肯定就不能放置</font>一所房子

- 对于第`i`个位置<font color='red'>不放置</font>一所房子，那么第`i - 1`个位置<font color='red'>可以放置也可以不放置</font>一所房子

那么「base case」是什么呢？

- 对于第 1 个位置，放置一所房子的方案数量为 1，即：`dp[1][1] = 1`
- 对于第 1 个位置，不放置一所房子的方案数量为 1，即：`dp[1][0] = 1`

最后的结果就是「第`n`个位置放置 ➕ 第`n`个位置不放置」

下面给出完整代码：

```java
public int countHousePlacements(int n) {
    int mod = (int) 1e9 + 7;
    int[][] dp = new int[n + 1][2];
    // base case
    dp[1][1] = 1;
    dp[1][0] = 1;
    for (int i = 2; i <= n; i++) {
        // 放置
        dp[i][1] = dp[i - 1][0] % mod;
        // 不放置
        dp[i][0] = (dp[i - 1][0] + dp[i - 1][1]) % mod;
    }
    long cnt = (dp[n][1] + dp[n][0]) % mod;
    return (int) (cnt * cnt % mod);
}
```

其实这个题目和 **[将字符串翻转到单调递增](https://leetcode.cn/problems/flip-string-to-monotone-increasing/)** 有相似之处，有兴趣的可以去看看，这里给出代码：

```java
// dp[i][0] 表示第 i 个位置变成 0 需要的最小翻转次数
// dp[i][1] 表示第 i 个位置变成 1 需要的最小翻转次数
public int minFlipsMonoIncr(String s) {
    int n = s.length();
    int[][] dp = new int[n + 1][2];
    for (int i = 1; i <= n; i++) {
        int cur = s.charAt(i - 1) - '0';
        // 第 i 个位置为 0，那么第 i - 1 个位置必须也为 0
        // 如果 cur = 1，就需要增加一次翻转次数
        dp[i][0] = dp[i - 1][0] + (cur == 0 ? 0 : 1);
        // 第 i 个位置为 1，那么第 i - 1 个位置可以为 0，也可以为 1
        dp[i][1] = Math.min(dp[i - 1][1], dp[i - 1][0]) + (cur == 0 ? 1 : 0);
    }
    return Math.min(dp[n][0], dp[n][1]);
}
```

### <font color=#1FA774>不同骰子序列的数目</font>

**题目详情可见 [不同骰子序列的数目](https://leetcode.cn/problems/number-of-distinct-roll-sequences/)**

该问题有两个约束条件：

- 序列中任意**相邻**数字的**最大公约数**为 1
- 序列中相等的值之间，至少有 2 个其他值的数字 (言外之意：第`i`个数字不能和前两个数字相同)

这里给出`dp[][][]`数组的定义：

- `dp[cnt][i][j]`表示第`cnt`轮倒数第二个数为`i`，倒数第一个数为`j`的**不同**骰子序列的数目

那么「状态转移方程」是什么呢？

- 对于满足上述两个约束条件时，`dp[cnt][j][cur] += dp[cnt - 1][i][j];`

那么「base case」是什么呢？

- 对于第 1 轮，**不同**骰子序列的数目为 6
- 对于第 2 轮，满足上述两个约束条件时，`dp[2][i][j] = 1`

最后的结果就是「第`n`轮倒数两个数的所有排列之和」

下面给出完整代码：

```java
class Solution {
    public int distinctSequences(int n) {
        // base case 1
        if (n == 1) return 6;
        int mod = (int) 1e9 + 7;
        int[][] g = new int[7][7];
        // 提前求出所有公约数
        for (int i = 1; i <= 6; i++) {
            for (int j = 1; j <= 6; j++) g[i][j] = gcd(i, j);
        }
        int[][][] dp = new int[n + 1][7][7];
        // base case 2
        for (int i = 1; i <= 6; i++) {
            for (int j = 1; j <= 6; j++) {
                if (i != j && g[i][j] == 1) dp[2][i][j] = 1;
            }
        }
        for (int cnt = 3; cnt <= n; cnt++) {
            // 倒数第二个数
            for (int i = 1; i <= 6; i++) {
                // 倒数第一个数
                for (int j = 1; j <= 6; j++) {
                    if (i != j && g[i][j] == 1) {
                        // 当前数
                        for (int cur = 1; cur <= 6; cur++) {
                            if (cur != i && cur != j && g[j][cur] == 1) {
                                dp[cnt][j][cur] += dp[cnt - 1][i][j];
                                dp[cnt][j][cur] %= mod;
                            }
                        }
                    }
                }
            }
        }
        // 最终结果
        int ans = 0;
        for (int i = 1; i <= 6; i++) {
            for (int j = 1; j <= 6; j++) {
                ans = (ans + dp[n][i][j]) % mod;
            }
        }
        return ans;
    }
    // 求公约数
    private int gcd(int a, int b) {
        return b == 0 ? a : gcd(b, a % b);
    }
}
```

下面再给出一种「记忆化搜索」方法实现的代码：(思路和上面的 DP 差不多)

```java
class Solution {
    private int[][] g;
    private int[][][] emeo;
    private int mod = (int) 1e9 + 7;
    public int distinctSequences(int n) {
        if (n == 1) return 6;
        g = new int[7][7];
        for (int i = 1; i <= 6; i++) {
            for (int j = 1; j <= 6; j++) g[i][j] = gcd(i, j);
        }
        emeo = new int[n + 1][7][7];
        return backtrack(n, 1, 0, 0);
    }
    private int backtrack(int n, int cur, int last2, int last1) {
        if (cur > n) return 1;
        if (emeo[cur][last2][last1] != 0) return emeo[cur][last2][last1];
        int ans = 0;
        for (int i = 1; i <= 6; i++) {
            if (last1 != 0 && g[last1][i] != 1) continue;
            if ((last1 != 0 && i == last1) || (last2 != 0 && i == last2)) continue;
            ans = (ans + backtrack(n, cur + 1, last1, i)) % mod;
        }
        emeo[cur][last2][last1] = ans;
        return ans;
    }
    private int gcd(int a, int b) {
        return b == 0 ? a : gcd(b, a % b);
    }
}
```

### <font color=#1FA774>拼接数组的最大分数</font>

**题目详情可见 [拼接数组的最大分数](https://leetcode.cn/problems/maximum-score-of-spliced-array/)**

这个题目在比赛中一下就想到了用「前缀和」解决，大概思路：分别求出两个数组的前缀和，然后求出差值最大的子数组之和

这无疑需要使用双重循环遍历所有子数组，最后直接超时！！！

比赛结束后，看到了一句话，醍醐灌顶！！**<font color='red'>「转化成求最大子数组之和」</font>**

以两个数组的差值为目标数组，然后求出该差值数组的最大子数组之和即可，**可参考 [最大子数组和](./秒杀子数组类题目.html#最大子数组和)**

下面给出完整代码：

```java
public int maximumsSplicedArray(int[] nums1, int[] nums2) {
    return Math.max(helper(nums1, nums2), helper(nums2, nums1));
}
// 差值数组为 nums2[i] - nums1[i]
private int helper(int[] nums1, int[] nums2) {
    int n = nums1.length;
    int max = nums2[0] - nums1[0];
    int dp = max, sum = nums1[0];
    for (int i = 1; i < n; i++) {
        int diff = nums2[i] - nums1[i];
        sum += nums1[i];
        if (dp < 0) dp = diff;
        else dp += diff;
        max = Math.max(max, dp);
    }
    return max > 0 ? sum + max : sum;
}
```