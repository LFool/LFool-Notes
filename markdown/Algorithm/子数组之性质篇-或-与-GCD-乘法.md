# 子数组之性质篇「或｜与｜GCD｜乘法」

[2411. 按位或最大的最小子数组长度](https://leetcode.cn/problems/smallest-subarrays-with-maximum-bitwise-or/)

[898. 子数组按位或操作](https://leetcode.cn/problems/bitwise-ors-of-subarrays/)

[1521. 找到最接近目标值的函数值](https://leetcode.cn/problems/find-a-value-of-a-mysterious-function-closest-to-target/)

[6224. 最大公因数等于 K 的子数组数目](https://leetcode.cn/problems/number-of-subarrays-with-gcd-equal-to-k/)

[D. CGCDSSQ](https://codeforces.com/contest/475/problem/D)

[题目 2622: 蓝桥杯 2021 年第十二届国赛真题-和与乘积](https://www.dotcpp.com/oj/problem2622.html)



本篇文章要介绍的是一个利用某些性质的子数组模版，可利用的性质有：**或运算性质**、**与运算性质**、**GCD 性质**、**乘法性质**

该模版可以做到：

- 求出**所有**子数组按某一操作的结果
- 求出值等于该结果的子数组的个数
- 求出按某一操作的结果等于**任意给定数字**的子数组的最短长度/最长长度

下面从题目依次阐述上面说到的内容！！

### <font color=#1FA774>按位或最大的最小子数组长度</font>

**题目详情可见 [按位或最大的最小子数组长度](https://leetcode.cn/problems/smallest-subarrays-with-maximum-bitwise-or/)**

这个题目还总结了另外一种方法，**详情可见 [子数组之「与」「或」](./子数组之-与-或.html)**

「按位或」有一个特点：对于`x = x | y`，`x`的二进制中要么`1`还是`1`，要么`0`变成`1`，也就是单调非递减的特性

从最坏的情况出发，假设`x = 0`，每次改变一个比特，最终得到 $2^{29}-1<10^9$，那么在不断向右扩展的过程中，`x`的值最多只有 30 中情况

举个小例子：`x = 00000`，`x`的变化顺序可能是`x = 10000; x = 11000; x = 11100; x = 11110; x = 11111`；一定不可能出现的变化顺序：`x = 10000; x = 01000`

下面根据本题的一个例子展开讨论：`nums = [1,0,2,1,3]` (这里有一个小技巧：从后往前遍历)

![10](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221024/13320716665895276Q0MXh10.svg)

左图是没有优化的过程，右图是优化了的过程，可以看到我们将同一行相同的元素进行了合并，而且是向左合并

比如第四行：`3 [1,3], 3 [1,4]`合并成了`3 [1,3]`，为什么不合并成`3[1,4]`呢？？

题目要求「或运算值**最大**的**最小**非空子数组」，所以`[1,3]`就能实现的事情，为什么要`[1,4]`呢！！这也是为什么向左合并了，是因为可以保证子数组长度最小

这里还涉及到一个原地去重的技巧，**可见题目 [删除有序数组中的重复项](https://leetcode.cn/problems/remove-duplicates-from-sorted-array/)**。之所以可以原地去重，是因为每一层的结果都有序

下面给出代码的时间复杂度为 $O(30 \times n)$

```java
public int[] smallestSubarrays(int[] nums) {
    int n = nums.length;
    int[] ans = new int[n];
    List<int[]> ors = new ArrayList<>();
    for (int i = n - 1; i >= 0; i--) {
        // 存储 [值, 下标] 的形式，表示以 i 开头的子数组中或运算值最大的最小非空子数组 
        ors.add(new int[]{0, i});
        int k = 0;
        // 当前元素 nums[i] 分别与上一层每个元素做或运算
        for (int[] or : ors) {
            or[0] |= nums[i];
            // 原地去重
            if (ors.get(k)[0] == or[0])  // 值相同
                ors.get(k)[1] = or[1];
            else ors.set(++k, or);       // 值不同
        }
        // 截断 [k + 1, ors.size())
        ors.subList(k + 1, ors.size()).clear();
        // 第 0 个就是区间 [i, n) 内值最大的最小非空子数组
        ans[i] = ors.get(0)[1] - i + 1;
    }
    return ans;
}
```

### <font color=#1FA774>子数组按位或操作</font>

**题目详情可见 [子数组按位或操作](https://leetcode.cn/problems/bitwise-ors-of-subarrays/)**

这个题目和上一个题目几乎一样，不同在于本题只需要求出所有子数组或运算不同结果的数量，我们可以使用`Set`实现自动去重的效果

```java
public int subarrayBitwiseORs(int[] nums) {
    int n = nums.length;
    Set<Integer> ans = new HashSet<>();
    Set<Integer> ors = new HashSet<>();
    // 正反遍历都一样
    for (int i = 0; i < n; i++) {
        // 记录下一层的结果
        Set<Integer> t = new HashSet<>();
        for (int or : ors) {
            or |= nums[i];
            // or 为下一层的新值
            t.add(or);
        }
        t.add(nums[i]);
        // 交换 ors 和 t
        ors = t;
        // 将每一层的结果都存入 ans 中
        ans.addAll(ors);
    }
    return ans.size();
}
```

### <font color=#1FA774>找到最接近目标值的函数值</font>

**题目详情可见 [找到最接近目标值的函数值](https://leetcode.cn/problems/find-a-value-of-a-mysterious-function-closest-to-target/)**

「按位与」有一个特点：对于`x = x & y`，`x`的二进制中要么`0`还是`0`，要么`1`变成`0`，也就是单调非递增的特性

从最坏的情况出发，假设`x = 111...111`，每次改变一个比特，最终得到 $0$，那么在不断向右扩展的过程中，`x`的值最多只有 30 中情况

举个小例子：`x = 11111`，`x`的变化顺序可能是`x = 11110; x = 11100; x = 11000; x = 10000; x = 00000`；一定不可能出现的变化顺序：`x = 10000; x = 01000`

这个题目，我们可以用一个`Set`存储所有可能的与运算结果，和上一题一样，然后在所有的结果中寻找与目标值`target`相差最小的那一个即可

```java
public int closestToTarget(int[] arr, int target) {
    int n = arr.length;
    Set<Integer> all = new HashSet<>();
    Set<Integer> ors = new HashSet<>();
    for (int i = 0; i < n; i++) {
        Set<Integer> t = new HashSet<>();
        for (int or : ors) {
            or &= arr[i];
            t.add(or);
        }
        t.add(arr[i]);
        ors = t;
        all.addAll(ors);
    }
    int ans = Integer.MAX_VALUE;
    // 在 all 中寻找与目标值 target 相差最小值
    for (int t : all) {
        ans = Math.min(ans, Math.abs(t - target));
    }
    return ans;
}
```

### <font color=#1FA774>最大公因数等于 K 的子数组数目</font>

**题目详情可见 [最大公因数等于 K 的子数组数目](https://leetcode.cn/problems/number-of-subarrays-with-gcd-equal-to-k/)**

先复习一个知识点，求两个数的最大公因数直接`gcd(a, b)`即可；如果求多个数的最大公因数呢？如：求`a, b, c`的最大公因数

```java
int r = gcd(a, gcd(b, c));

private int gcd(int a, int b) {
    return a % b == 0 ? b : gcd(b, a % b);
}
```

![15](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221024/1559261666598366benNfs15.svg)

左图是没有优化的过程，中间的图是去除了重复 GCD 的过程，右图是删去了因子中没有 K 的情况，因为没有因子 K，必不可能组成最大公因数等于 K 的子数组，我们用`i0`记录这种情况的下标

根据 gcd 的特点，每一层的 gcd 值单调非递减，这也是可以原地去重的前提！！

下面给出代码的时间复杂度为 $O(n \ logU), \ U = \max(nums)$

```java
public int subarrayGCD(int[] nums, int k) {
    int n = nums.length, ans = 0;
    List<int[]> gcds = new ArrayList<>();
    int i0 = -1;
    // 正向遍历
    for (int i = 0; i < n; i++) {
        // 没有因子 k 的数
        if (nums[i] % k != 0) {
            // 记录下标
            i0 = i;
            // 重置 List
            gcds = new ArrayList<>();
            continue;
        }
        // 存储 [值, 下标] 的形式，表示以 i 结尾的子数组中所有 gcd 的结果 
        gcds.add(new int[]{nums[i], i});
        int j = 0;
        for (int[] g : gcds) {
            // gcd() 见上方
            g[0] = gcd(g[0], nums[i]);
            // 原地去重
            if (gcds.get(j)[0] == g[0])  // 值相同
                gcds.get(j)[1] = g[1];
            else gcds.set(++j, g);       // 值不同
        }
        // 截断 [j + 1, ors.size())
        gcds.subList(j + 1, gcds.size()).clear();
        // 每一层的数必为 k 的倍数，即：x >= nk, (n = 1,2,3...)
        // 故只需要判断第一个数是否为 k 即可
        if (gcds.get(0)[0] == k) ans += gcds.get(0)[1] - i0;
    }
    return ans;
}
```

### <font color=#1FA774>D. CGCDSSQ</font>

**题目详情可见 [D. CGCDSSQ](https://codeforces.com/contest/475/problem/D)**

这个题和上一个题目几乎一模一样，但是有一些处理上的技巧！！如果我们按照上个题的方法，每次一次查询都调用一次函数，那么这样会超时！！

我们可以一次遍历，计算出所有不同最大公因数的结果，存储起来，然后每次查询就可以只用 $O(1)$ 的时间

```java
import java.util.*;
import java.io.*;
public class Main {
    private static Map<Integer, Long> emeo = new HashMap<>();
    public static void main(String[] args) throws IOException {
        int n = MyRead.nextInt();
        int[] arr = new int[n];
        for (int i = 0; i < n; i++) arr[i] = MyRead.nextInt();
        // 一次遍历，计算出所有不同最大公因数的结果，存储在 emeo 中
        f(arr);
        int q = MyRead.nextInt();
        for (int i = 0; i < q; i++) {
            System.out.println(emeo.getOrDefault(MyRead.nextInt(), 0L));
        }
    }
    public static void f(int[] arr) {
        int n = arr.length;
        List<int[]> g = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            // 注意：此时没有上一题中右图的优化，因为此时并不是针对某一个 k，而是针对所有的值
            int k = 0;
            g.add(new int[]{arr[i], i});
            for (int[] c : g) {
                c[0] = gcd(c[0], arr[i]);
                if (g.get(k)[0] == c[0]) {
                    g.get(k)[1] = c[1];
                } else g.set(++k, c);
            }
            g.subList(k + 1, g.size()).clear();
            // 计算每一层不同最大公因数的结果
            for (int j = 0; j < g.size(); j++) {
                // 当前计算的 gcd 值
                int cur = g.get(j)[0];
                // 向左找不满足要求的数字，类似于找 i0
                // for 循环结束后，k 的下标即为 i0
                for (k = j - 1; k >= 0; k--) if (g.get(k)[0] % cur != 0) break;
                int cnt = k < 0 ? g.get(j)[1] + 1 : g.get(j)[1] - g.get(k)[1];
                emeo.put(cur, emeo.getOrDefault(cur, 0L) + cnt);
            }
        }
    }
    public static int gcd(int a, int b) {
        return a % b == 0 ? b : gcd(b, a % b);
    }
}
// 快读模版
class MyRead {
    private static final BufferedReader bf = new BufferedReader(new InputStreamReader(System.in));
    private static final StreamTokenizer st = new StreamTokenizer(bf);
    public static int nextInt() throws IOException {
        st.nextToken();
        return (int) st.nval;
    }
    public static String readLine() throws IOException {
        return bf.readLine();
    }
}
```

### <font color=#1FA774>蓝桥杯 2021 年第十二届国赛真题-和与乘积</font>

**题目详情可见 [和与乘积](https://www.dotcpp.com/oj/problem2622.html)**

![14](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221024/1635501666600550TGXO7w14.svg)

类似的，每一层的乘积值单调非递增，这也是可以原地去重的前提！！

需要注意的是，由于 1 乘任何数都等于本身，导致去重合并的时候，需要存储相同值的左右边界

比如：区间`[0,2]`和`[1,2]`的乘积都是 6，但如果只记录右边界`1`，可能导致可行解`a[0] + a[1] + a[2] = a[0] * a[1] * a[2]`丢失

所以对于每一个子数组区间，都记录两个值，分别表示左右边界，然后可以利用前缀和具有单调性，二分搜索的去寻找

可以看出「乘积」和「GCD」是一个逆过程，乘积是通过因子求原数的过程，GCD 是通过原数求因子的过程，它们俩每一层的单调性刚好相反！！

```java
import java.util.*;
class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        int[] arr = new int[n];
        long[] ps = new long[n + 1];
        for (int i = 0; i < n; i++) {
            arr[i] = sc.nextInt();
            // 前缀和
            ps[i + 1] = ps[i] + arr[i];
        }
        long ans = 0L;
        List<long[]> p = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            // 存储 [值, 左边界, 右边界] 的形式，表示子数组 [左边界, 右边界] 的乘积
            p.add(new long[]{1, i, i});
            int k = 0;
            for (long[] a : p) {
                a[0] *= arr[i];
                if (p.get(k)[0] == a[0]) {
                    p.get(k)[2] = a[2];
                } else p.set(++k, a);

            }
            p.subList(k + 1, p.size()).clear();
            for (long[] a : p) {
                // 左右边界
                int l = (int) a[1], r = (int) a[2];
                // 二分查找
                while (l < r) {
                    int m = l + r >> 1;
                    if (a[0] >= ps[i + 1] - ps[m]) r = m;
                    else l = m + 1;
                }
                // 找到！
                if (a[0] == ps[i + 1] - ps[l]) ans++;
            }
        }
        System.out.println(ans);
    }
}
```

