# 「数位 DP」详解

[233. 数字 1 的个数](https://leetcode.cn/problems/number-of-digit-one/)

[面试题 17.06. 2出现的次数](https://leetcode.cn/problems/number-of-2s-in-range-lcci/)

[788. 旋转数字](https://leetcode.cn/problems/rotated-digits/)

[600. 不含连续1的非负整数](https://leetcode.cn/problems/non-negative-integers-without-consecutive-ones/)

[1012. 至少有 1 位重复的数字](https://leetcode.cn/problems/numbers-with-repeated-digits/)

[2376. 统计特殊整数](https://leetcode.cn/problems/count-special-integers/)

[902. 最大为 N 的数字组合](https://leetcode.cn/problems/numbers-at-most-n-given-digit-set/)



「数位 DP」属于会就不难，不会就巨难！！本篇文章就来好好详细的总结一下「数位 DP」吧吧吧！！

### <font color=#1FA774>傻瓜式遍历</font>

现在有一个需求：遍历`<= n`的所有数字 (OS：what the f**! 逗我玩呢，这么简单！！)

```java
for (int i = 0; i <= n; i++) {
    // ...
}
```

### <font color=#1FA774>换种遍历方式吧「按位遍历」</font>

熟悉「DP」和「记忆化搜索 (回溯 + 备忘录)」的同学肯定很清楚「状态」这个概念，「DP」中最难的也就是推出「状态转移方程」

「DP」是通过已知的状态推出未知的状态；而「记忆化搜索」是记录已经算出的状态，下次再遇到该状态时，直接复用即可，就不需要重复计算了

总之，它们俩的核心就是记录「状态」，避免重复计算，降低耗时

如果对上面说的内容不熟悉的同学，可以先学习下面推荐的文章：

- **[动态规划解题套路框架](./动态规划解题套路框架.html)**
- **[回溯 (DFS) 算法框架](./回溯(DFS).html)**
- **[目标和 -「回溯」&「动规」](./目标和-回溯-动规.html)**
- **[下降路径最小和 -「回溯」&「动规」](./下降路径最小和-回溯-动规.html)**

「数位 DP」这类题目一般都是对一个数的数位动手脚，比如：求小于等于`n`的所有数中，数字`1`出现的个数

如果按照「傻瓜式遍历」，对于每一个数，都需要求出所有数位，然后判断，而且对于不同的数字都需要重新求所有数位，状态无法复用！！

那怎样的遍历才可以使「状态」得以复用呢？？这里以`n = 1234`为例展开分析

首先我们需要以每一**位**为单位，为了方便处理，先把数字`n`转化成字符数组`s = ['1', '2', '3', '4']`

现在考虑处于每一位上时面临的选择：(从左向右，即从高位开始) (**<font color='red'>温馨提示：结合下面的图一起食用</font>**)

- 当处于第`0`位的时候，可以选择`0, 1`
- 当处于第`1`位的时候，如果第`0`位选择的是`0`，那么此时就可以选择`0 - 9`；如果第`0`位选择的是`1`，那么此时的选择就不能超过原数第`1`位的值，即此时只能选择`0 - 2`
- 当处于第`2`位的时候，如果第`0`位选择的是`1`，第`1`位选择的是`2`，那么此时的选择就不能超过原数第`2`位的值，那么此时只能选择`0 - 3`；其他情况可以选择`0 - 9`
- 当处于第`3`位的时候，也是同理

下图中的`isLimit`标记是否受到了限制。若为真，则第`i`位填入的数字至多为`s[i]`，否则可以是`9`。如果在受到限制的情况下填了`s[i]`，那么后续填入的数字仍会受到限制 🚫

当我们依次在`0 - 3`位做了选择后，会形成一条路径，即为一个`<= 1234`的数，那么所有这样的路径构成的集合就是所有`<= 1234`的数，且从`0`开始

这样就实现了另外一种遍历的方式啦，即「按位遍历」

![1](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220928/1622061664353326jwa8w01.svg)

下面给出这种遍历方式的代码：

```java
private List<Integer> list = new ArrayList<>();
public void getAllNum(int n) {
    char[] s = Integer.toString(n).toCharArray();
    // 处于第 0 位的时候，选择是被限制的，只能选择不超过第 0 位的值
    traversal(s, 0, 0, true);
    // list 就是所有数的集合啦
}
// 从第 i 位开始遍历
// path 记录路径
// isLimit 如图所示，为了防止大小超过 n
private void traversal(char[] s, int i, int path, boolean isLimit) {
    // 结束条件
    if (i == s.length) {
        list.add(path);
        return ;
    }
    // 确定选择的上界
    // 如果 isLimit 为 true，那么可选择的上界不能超过该位的值；否则可以一直选择到 9
    int up = isLimit ? s[i] - '0' : 9;
    for (int d = 0; d <= up; d++) {
        // 递归遍历下一位
        // 下一位的 isLimit 确定方法：当前位被限制了，而且选择的值是上界
        // 继续按照上图，举个例子：当处于第 0 位时，isLimit 为 true，
        // 如果此时选择上界 1，那么遍历第 1 位的时候也是被限制的；
        // 但是如果此时选择的不是上界 1，那么遍历第 1 位的时候就没有被限制
        traversal(s, i + 1, path * 10 + d, isLimit && d == up);
    }
}
```

### <font color=#1FA774>主角出场</font>

铺垫了这么多，现在由题目 **[数字 1 的个数](https://leetcode.cn/problems/number-of-digit-one/)** 引出今天的主角！！

题目很短，意思也很好懂，但也很容易超时。如果直接暴力的话，思路很简单，遍历每一个数，然后计算每一个数中 1 的个数即可，伪代码如下：

```java
int sum = 0;
for (int i = 0; i <= n; i++) {
    sum += f(计算数字 i 中 1 的个数);
}
```

`n`的范围：$0 \le n \le 10^9$，求一个数中 1 的个数最坏需要循环 10 次 ($10^9$ 大概是 $2^{30}$)，所以最坏的时间复杂度为 $O(10\times n)$

按照「按位遍历」的思路，先照葫芦画瓢，写一个遍历所有数字的代码 (这里不存储所有结果，只把个数返回一下)：

```java
private char[] s;
public int countDigitOne(int n) {
    s = Integer.toString(n).toCharArray();
    return f(0, true);
}
// 返回从第 i 位开始的 <= n 的个数
private int f(int i, boolean isLimit) {
    // 结束条件
    if (i == s.length) return 1;
    int up = isLimit ? s[i] - '0' : 9;
    int res = 0;
    for (int d = 0; d <= up; d++) {
        res += f(i + 1, isLimit && d == up);
    }
    return res;
}
```

是不是和上一部分的遍历代码一模一样！！

现在把返回值的定义变一下：返回从第`i`位开始，数字中`1`的个数

只需要多加一个参数即可：

```java
private char[] s;
public int countDigitOne(int n) {
    s = Integer.toString(n).toCharArray();
    return f(0, 0 ,true);
}
// 返回从第 i 位开始，数字中 1 的个数
private int f(int i, int oneCnt, boolean isLimit) {
    // 结束条件
    // 前文说过，结束时就表示一条完整的路径，即一个 <= n 的数字，而 ontCnt 则表示该数字中 1 的个数
    // 所以直接返回 oneCnt 即可！！
    if (i == s.length) return oneCnt;
    int up = isLimit ? s[i] - '0' : 9;
    int res = 0;
    for (int d = 0; d <= up; d++) {
        // 如果 d = 1，oneCnt 就➕1
        res += f(i + 1, oneCnt + (d == 1 ? 1 : 0), isLimit && d == up);
    }
    return res;
}
```

这个代码其实已经满足题目的要求，但也还是会超时！！不过不过不过，这种遍历方式的状态可以复用

我们来分析一下，到底哪些状态可以复用！！为了简单分析，我们把上面的图抽离出部分内容单独分析

![2](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220928/1914071664363647JuuYWs2.svg)

假设第一种情况的结果已经计算出来

如果我们处于第三种情况下的第`1`位的时候，**思考：后面的部分还需要再次计算吗？**

- 显然不需要，因为在第一种情况的时候已经算过了，只要我们将第一种情况计算的结果保存一下即可再次复用

如果我们处于第二种情况下的第`1`位的时候，**思考：后面的部分还需要再次计算吗？**

- 这次是需要滴！有人可能有疑问了，为啥第三种情况不需要计算，而第二种情况就需要了

- 设绿色部分的结果为`x`，如果直接复用，那么第二种情况最终返回的结果为`x`，显然有问题呀，因为第二种情况的第`1`位为 1，所以第二种情况的结果应该是`x + 1`才对呀

- 出现这个问题的原因在于：我们不能只通过位数来表示一种状态，还需要根据当前已有 1 数量，即参数`oneCnt`，所以我们可以用一个二维数组`emeo[][]`来表示所有状态

还有最后一个问题，蓝色部分的结果可以复用吗？

- 显然也是不可以的，蓝色部分由于限制的原因，只能选择`0 - 3`，状态和上面绿色的部分是不一样的。`isLimit`限制至多只会出现 1 次，到时候特判一下即可

说了这么多，都是教你如何唯一标识一个「状态」，其实这里有一个小窍门，记录「状态」是为了减少递归的次数，也就是说一次递归的结果对应着一个「状态」。所以看递归函数中的参数有哪些，就可以判断如何才能唯一标识一个「状态」

下面的代码中，递归函数为`f(int i, int oneCnt, boolean isLimit) `，是不是和我们的状态对应上了！！下面给出的例子中会进一步阐明这个小窍门！

下面给出最终的代码：

```java
private char[] s;
private int[][] emeo;  // 备忘录
public int countDigitOne(int n) {
    s = Integer.toString(n).toCharArray();
    int m = s.length;
    // 根据题目给出的范围，一个数中 1 的数量最多只有 10 种情况
    emeo = new int[m][10];
    // 初始化为 -1
    for (int i = 0; i < m; i++) Arrays.fill(emeo[i], -1);
    return f(0, 0 ,true);
}
// 回从第 i 位开始，数字中 1 的个数
private int f(int i, int oneCnt, boolean isLimit) {
    // 结束条件：到达此处的路径均为可行解，oneCnt 表示该可行解中 1 的数量
    if (i == s.length) return oneCnt;
    // 「没有限制」且「该状态结果已经计算出」，则直接返回
    if (!isLimit && emeo[i][oneCnt] != -1) return emeo[i][oneCnt];
    int up = isLimit ? s[i] - '0' : 9;
    int res = 0;
    for (int d = 0; d <= up; d++) {
        res += f(i + 1, oneCnt + (d == 1 ? 1 : 0), isLimit && d == up);
    }
    // 记录该状态的结果
    if (!isLimit) emeo[i][oneCnt] = res;
    return res;
}
```

### <font color=#1FA774>题目实战</font>

#### <font color=#9933FF>面试题 17.06. 2出现的次数</font>

**题目详情可见 [面试题 17.06. 2出现的次数](https://leetcode.cn/problems/number-of-2s-in-range-lcci/)**

几乎和上面分析的题目一模一样，这里直接给出代码：

```java
private char[] s;
private int[][] emeo;
public int numberOf2sInRange(int n) {
    s = Integer.toString(n).toCharArray();
    int m = s.length;
    emeo = new int[m][10];
    for (int i = 0; i < m; i++) Arrays.fill(emeo[i], -1);
    return f(0, 0, true, false);
}
private int f(int i, int twoCnt, boolean isLimit) {
    if (i == s.length) return twoCnt;
    if (!isLimit && emeo[i][twoCnt] != -1) return emeo[i][twoCnt];
    int up = isLimit ? s[i] - '0' : 9;
    int res = 0;
    for (int d = 0; d <= up; d++) {
        res += f(i + 1, twoCnt + (d == 2 ? 1 : 0), isLimit && d == up);
    }
    if (!isLimit) emeo[i][twoCnt] = res;
    return res;
}
```

#### <font color=#9933FF>旋转数字</font>

**题目详情可见 [旋转数字](https://leetcode.cn/problems/rotated-digits/)**

「好数」必须包含旋转后为不同的数字，用数组`ISDIFF[]`记录每个数字的旋转情况，见注释

通过一次剪枝，过滤掉了存在不能旋转数字的情况，见注释

对于备忘录`emeo[][]`，一维表示位数`i`，二维表示是否包含旋转后为不同的数字`hasDiff`，是不是和递归函数的参数对应上了！

```java
private char[] s;
private int[][] emeo;
// ISDIFF[i] == -1 表示 i 不能旋转，如：3，4，7
// ISDIFF[i] == 0 表示 i 旋转后为同一个数字，如：0，1，8
// ISDIFF[i] == 1 表示 i 旋转后为不同的数字，如：2，5，6，9
private int[] ISDIFF = new int[]{0, 0, 1, -1, -1, 1, 1, -1, 0, 1};
public int rotatedDigits(int n) {
    s = Integer.toString(n).toCharArray();
    int m = s.length;
    emeo = new int[m][2];
    for (int i = 0; i < m; i++) Arrays.fill(emeo[i], -1);
    return f(0, 0, true);
}
// hasDiff 表示数字中是否包含旋转后为不同的数字，包含则为 1，不包含则为 0
private int f(int i, int hasDiff, boolean isLimit) {
    // 结束条件：到达此处的路径根据 hasDiff 判断是否为可行解，直接返回 hasDiff 即可
    if (i == s.length) return hasDiff;
    if (!isLimit && emeo[i][hasDiff] != -1) return emeo[i][hasDiff];
    int up = isLimit ? s[i] - '0' : 9;
    int res = 0;
    for (int d = 0; d <= up; d++) {
        // -1 表示存在不能旋转的数字，如：3，4，7，直接跳过
        if (ISDIFF[d] != -1) {
            // 如果 ISDIFF[d] = 1，则表示旋转后为不同数字，hasDiff | ISDIFF[d] 或运算后值为 1
            res += f(i + 1, hasDiff | ISDIFF[d], isLimit && d == up);
        }
    }
    if (!isLimit) emeo[i][hasDiff] = res;
    return res;
}
```

#### <font color=#9933FF>不含连续1的非负整数</font>

**题目详情可见 [不含连续1的非负整数](https://leetcode.cn/problems/non-negative-integers-without-consecutive-ones/)**

第一步：先转化成二进制

通过一次剪枝，过滤掉了存在 1 的情况，见注释

对于备忘录`emeo[][]`，一维表示位数`i`，二维表示是否包含旋转后为不同的数字`prev`，是不是和递归函数的参数对应上了！

```java
private char[] s;
private int[][] emeo;
public int findIntegers(int n) {
    // 转化为二进制
    s = Integer.toString(n, 2).toCharArray();
    int m = s.length;
    emeo = new int[m][2];
    for (int i = 0; i < m; i++) Arrays.fill(emeo[i], -1);
    return f(0, 0, true);
}
// prev 表示 i - 1 位的数字，即：前一个数字
private int f(int i, int prev, boolean isLimit) {
    // 结束条件：到达此处的路径均为可行解，直接返回 1 即可
    if (i == s.length) return 1;
    if (!isLimit && emeo[i][prev] != -1) return emeo[i][prev];
    int up = isLimit ? s[i] - '0' : 1;
    int res = 0;
    for (int d = 0; d <= up; d++) {
        // prev == 1 && d == 1 表示包含连续的 1，直接跳过
        if (!(prev == 1 && d == 1)) res += f(i + 1, d, isLimit && d == up);
    }
    if (!isLimit) emeo[i][prev] = res;
    return res;
}
```

#### <font color=#9933FF>至少有 1 位重复的数字</font>

**题目详情可见 [至少有 1 位重复的数字](https://leetcode.cn/problems/numbers-with-repeated-digits/)**

这个题目和上面给出的题目又一丢丢的不一样，在上面的题目中，`0001`和`1`其实是等价的，因为题目中并没有对`0`有特殊要求

但是在本题中，要求「至少有 1 位重复的数字」，此时`0001`和`1`就不等价了，`0001`是一个满足要求的数字，而`1`是不满足要求的数字，而在`[0, n]`中，`0001`其实是非法的，所以会出现错误 ❌

这里引入一个新的参数`isNum`，表示`i`前面是否已经选了数字。`isNum`主要的作用是过滤掉「前导零」，如果对「前导零」没有要求，则可以不使用该参数

先抛开`isLimit`限制，如果`i`前面已经选了数字，那么`i`可以选择的数字为`[0, 9]`；如果`i`前面没有选数字，那么`i`可以选择的数字为`[1, 9]`

对于备忘录`emeo[][]`，一维表示位数`i`，二维表示`mask`，是不是和递归函数的参数对应上了！

`isNum`和`isLimit`一样，在判断「状态」是否存在时，需要特判一波！

```java
private char[] s;
private int[][] emeo;
public int numDupDigitsAtMostN(int n) {
    s = Integer.toString(n).toCharArray();
    int m = s.length;
    emeo = new int[m][1024];
    for (int i = 0; i < m; i++) Arrays.fill(emeo[i], -1);
    // 结果取反
    return n - f(0, 0, true, false);
}
// 表示没有重复的字数的个数，最后 n - f() 即可
// mask 大小为 1024，长度为 10 位，记录 0 - 9 选择的情况，若 i 已选择，则 mask 中第 i 位为 1
private int f(int i, int mask, boolean isLimit, boolean isNum) {
    // 结束条件：如果 isNum 为 false，表示一直没有选择，直接返回 0；否则返回 1
    // 如果到结束条件时，isNum 仍为 false，还有一种表示含义，即表示 0
    if (i == s.length) return isNum ? 1 : 0;
    // isNum 同 isLimit，需要特判一下
    if (!isLimit && isNum && emeo[i][mask] != -1) return emeo[i][mask];
    // 下界
    int down = isNum ? 0 : 1;
    int up = isLimit ? s[i] - '0' : 9;
    // f(i + 1, mask, false, false) 表示仍然不选择
    int res = isNum ? 0 : f(i + 1, mask, false, false);
    for (int d = down; d <= up; d++) {
        // 如果 d 没有选择过
        if (((mask >> d) & 1) == 0) {
            res += f(i + 1, mask | (1 << d), isLimit && d == up, true);
        }
    }
    if (!isLimit && isNum) emeo[i][mask] = res;
    return res;
}
```

#### <font color=#9933FF>统计特殊整数</font>

**题目详情可见 [统计特殊整数](https://leetcode.cn/problems/count-special-integers/)**

这个题目和上个题目一模一样，上个题目的要求「至少有 1 位重复的数字」，我们转化成了「n - 没有重复的数组」

所以转化后的`f()`函数刚好和本题的要求一致！

```java
class Solution {
    private char[] s;
    private int[][] emeo;
    public int countSpecialNumbers(int n) {
        s = Integer.toString(n).toCharArray();
        int m = s.length;
        emeo = new int[m][1024];
        for (int i = 0; i < m; i++) Arrays.fill(emeo[i], -1);
        return f(0, 0, true, false);
    }
    private int f(int i, int mask, boolean isLimit, boolean isNum) {
        if (i == s.length) return isNum ? 1 : 0;
        if (!isLimit && isNum && emeo[i][mask] != -1) return emeo[i][mask];
        int down = isNum ? 0 : 1;
        int up = isLimit ? s[i] - '0' : 9;
        int res = isNum ? 0 : f(i + 1, mask, false, false);
        for (int d = down; d <= up; d++) {
            if (((mask >> d) & 1) == 0) res += f(i + 1, mask | (1 << d), isLimit && d == up, true);
        }
        if (!isLimit && isNum) emeo[i][mask] = res;
        return res;
    }
}
```

#### <font color=#9933FF>最大为 N 的数字组合</font>

**题目详情可见 [最大为 N 的数字组合](https://leetcode.cn/problems/numbers-at-most-n-given-digit-set/)**

和上面的大同小异！

```java
private char[] s;
private int[] emeo;
private int[] dd;
public int atMostNGivenDigitSet(String[] digits, int n) {
    s = Integer.toString(n).toCharArray();
    dd = new int[digits.length];
    for (int i = 0; i < digits.length; i++) dd[i] = Integer.parseInt(digits[i]);
    int m = s.length;
    emeo = new int[m];
    Arrays.fill(emeo, -1);
    return f(0, true, false);
}
private int f(int i, boolean isLimit, boolean isNum) {
    if (i == s.length) return isNum ? 1 : 0;
    if (!isLimit && isNum && emeo[i] != -1) return emeo[i];
    int up = isLimit ? s[i] - '0' : 9;
    int res = isNum ? 0 : f(i + 1, false, false);
    for (int d : dd) {
        if (d <= up) {
            res += f(i + 1, isLimit && d == up, true);
        }
    }
    if (!isLimit && isNum) emeo[i] = res;
    return res;
}
```