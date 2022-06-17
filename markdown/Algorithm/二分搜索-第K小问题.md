# 二分搜索：第 K 小问题

[719. 找出第 K 小的数对距离](https://leetcode.cn/problems/find-k-th-smallest-pair-distance/)

[1439. 有序矩阵中的第 k 个最小数组和](https://leetcode.cn/problems/find-the-kth-smallest-sum-of-a-matrix-with-sorted-rows/)



本篇文章主要介绍如何使用「二分搜索」来解决「第 K 小」的问题。如上面给出的两个题目，难度为「Hard」，在理解上可能需要花更多的时间！！

**关于「二分搜索」的详细介绍可见 [二分搜索](./二分搜索.html)**；**关于抽象类的二分搜索 [抽象类的二分搜索](./抽象类的二分搜索.html)**

今天的题目也属于「抽象类的二分搜索」类型，所有我们需要搞清楚什么是**「搜索对象」**和**「搜索范围」**

### <font color=#1FA774>找出第 K 小的数对距离</font>

**题目详情可见 [找出第 K 小的数对距离](https://leetcode.cn/problems/find-k-th-smallest-pair-distance/)**

**这里给出暴力的思路：**维护一个大小为`k`的大根堆优先队列，如果入队元素「小于」当前队顶元素，则弹出队顶元素，然后把要入队的元素压入队列中。当处理完所有元素后，队顶元素即为「第 K 小」的元素

当然，本篇文章要介绍的方法更快，更好！其实就是二分搜索啦！！！

我们先明确一下原问题：「返回**所有数对距离中**第`k`小的数对距离」

对于一个给定的数对距离`d`，如果该距离近了，那我们就「向右收缩」；如果该距离远了，那我们就「向左收缩」；如果该距离满足要求，那我们能不能找一个更小的且满足要求的距离，所以需要继续「向左收缩」(是不是很像「寻找最左相等元素」？自信点，就是「寻找最左相等元素」)

所以**「搜索对象」**是什么？？很明显就是**「距离」**嘛！！

那**「搜索范围」**又是什么呢？？

- 距离最小值可以到达多少？$0$
- 距离最大值可以到达多少？$10^6$ -> 根据题目范围得到滴！ 

明确了**「搜索对象」**和**「搜索范围」**，我们还需要搞清楚怎么确定距离近了还是远了，肯定要有一个参考对象才能确定近还是远嘛

很聪明，这个参考对象就是题目给的「第`k`小」。对于距离`d`，小于等于该距离的数量为`n`，如果`n < k`说明距离近了；如果`n > k`说明距离远了

#### <font color=#9933FF>双指针计算`n`</font>

对于一个距离`d`，怎么得到小于等于该距离的数量呢？

可以直接暴力，但这不是我们想要的，这里介绍「双指针」方法。

使用「双指针」的前提是需要数组有序，由于本题目中的距离是绝对值，所以事先对数组排序并不影响结果的正确性

假设`nums = [1, 4, 5, 8, 10, 12], d = 4`

双指针`i`表示区间的起点，`j`表示区间的终点，表示为`[i, j)`，注意是「左闭右开」

区间`[i, j)`表示对于任意 $a$ 且满足 $i < a < j$，均有 $d(i, a) \le 4$。具体如下图所示：

![1](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220617/2025301655468730OZ9wGB1.svg)

**解释：**对于区间`[0, 3)`，以`0`为起点的数对有：`(0, 1), (0, 2)`，其距离均小于等于 4

当`i = 0`满足要求的情况处理完后，`i`向前移动一格，此时`j`只需要在位置`3`的基础上继续向前扩张即可。具体如下图所示：

![2](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220617/21264916554724093DjxAw2.svg)

可以看到橙色部分是`i = 0`确定的区域，红色部分是新扩张的区域。由于`i`向前移动了一格，所以上一步满足要求的区域，此时肯定也满足要求 (**<font color='red'>有点绕，需要好好理解</font>**)

下面给出本题的代码实现：

```java
public int smallestDistancePair(int[] nums, int k) {
    // 排序
    Arrays.sort(nums);
    // 搜索范围
    int lo = 0, hi = (int) 1e6;
    while (lo <= hi) {
        int mid = lo - (lo - hi) / 2;
        // 对应距离远了，向左收缩
        if (check(nums, mid) >= k) hi = mid - 1;
        // 对应距离近了，向右收缩
        else lo = mid + 1;
    }
    return lo;
}
// 双指针计算 小于等于 x 的数量
private int check(int[] nums, int x) {
    int ans = 0;
    int n = nums.length;
    // 初始时，i = 0, j = 1
    for (int i = 0, j = 1; i < n; i++) {
        // j 向向前扩张
        while (j < n && nums[j] - nums[i] <= x) j++;
        // 对于区间 [i, j)，满足要求的数量为 j - i - 1
        ans += j - i - 1;
    }
    return ans;
}
```

### <font color=#1FA774>有序矩阵中的第 k 个最小数组和</font>

**题目详情可见 [有序矩阵中的第 k 个最小数组和](https://leetcode.cn/problems/find-the-kth-smallest-sum-of-a-matrix-with-sorted-rows/)**

我们先明确一下原问题：「返回所有可能数组中的第 k 个**最小**数组和」

对于一个给定的数组和`sum`，如果该数组和小了，那我们就「向右收缩」；如果该数组和大了，那我们就「向左收缩」；如果该数组和满足要求，那我们能不能找一个更小的且满足要求的数组和，所以需要继续「向左收缩」(是不是很像「寻找最左相等元素」？自信点，就是「寻找最左相等元素」)

所以**「搜索对象」**是什么？？很明显就是**「数组和」**嘛！！

那**「搜索范围」**又是什么呢？？

- 数组和最小值可以到达多少？肯定是该矩形的第一列组成的组数和嘛 (矩形每行递增)
- 数组和最大值可以到达多少？肯定是该矩形的最后一列组成的组数和嘛 (矩形每行递增)

明确了**「搜索对象」**和**「搜索范围」**，我们还需要搞清楚怎么确定数组和小了还是大了，肯定要有一个参考对象才能确定近还是远嘛

很聪明，这个参考对象就是题目给的「第`k`最小」。对于数组和`sum`，小于等于该数组和的数量为`n`，如果`n < k`说明数组和小了；如果`n > k`说明数组和大了

#### <font color=#9933FF>DFS 计算`n`</font>

这一部分还挺难理解的，看了好多题解，发现很多人都是一下子给出代码！！

既然这是一个 DFS 问题，肯定也是满足 DFS 套路的。**关于 DFS 的详细介绍可见 [回溯 (DFS) 算法框架](./回溯(DFS).html)**

初始状态为第一列，如下图所示：

![6](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220617/2102461655470966PxiWrO6.svg)

对于第一行，我们可以选择「不变」，也可以选择「 2 or 3 or 4 or 5」与「1」交换。假如选择了「4」与「1」交换，那么此时的子数组为「4，6，11，16」，和为「37」，如下图所示：

![7](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220617/2102591655470979Wwuab67.svg)

对于每一行，都和第一行差不多。为了更清晰的写出 DFS，下面给出「搜索树」

![9](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220617/21143416554716744vViK89.svg)

下面给出本题的代码实现：

```java
private int m, n, k;
private int[][] mat;
// 计算小于等于当前数组和的数量
private int cnt = 0;
public int kthSmallest(int[][] mat, int k) {
    this.k = k;
    this.mat = mat;
    m = mat.length;
    n = mat[0].length;
    // 搜索范围
    int left = 0, right = 0;
    for (int i = 0; i < m; i++) {
        left += mat[i][0];
        right += mat[i][n - 1];
    }
    // 把最小值设为初始值
    int init = left;
    while (left <= right) {
        int mid = left - (left - right) / 2;
        // 初始值也算一个可行解
        cnt = 1;
        dfs(0, init, mid);
        // 对应数组和大了，向左收缩
        if (cnt >= k) right = mid - 1;
        // 对应数组和小了，向右收缩
        else left = mid + 1;
    }
    return left;
}
// DFS 计算 小于等于 target 的数量
private void dfs(int row, int sum, int target) {
    // 特殊情况，直接返回
    // sum > target：当前数组和大于 target
    // cnt > k：当前小于等于 target 的数量大于 k
    // row >= m：已经到达最后一行 (结束条件)
    if (sum > target || cnt > k || row >= m) return;
    // 不做交换
    dfs(row + 1, sum, target);
    // 分别与 [1, n-1] 交换
    for (int i = 1; i < n; i++) {
        // 更新数组和：减去「第一个元素」，加上「要交换的元素」
        int newSum = sum - mat[row][0] + mat[row][i];
        // 交换后的数组和大于 target，直接跳出循环
        // 原因：由于每行元素递增，所以当前元素大了，该行后面的元素肯定也大了
        if (newSum > target) break;
        // 满足要求，cnt + 1
        cnt++;
        // 搜索下一行
        dfs(row + 1, newSum, target);
    }
}
```
