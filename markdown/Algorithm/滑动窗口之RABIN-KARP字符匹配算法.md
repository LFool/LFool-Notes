# 滑动窗口之 RABIN KARP 字符匹配算法

[187. 重复的DNA序列](https://leetcode.cn/problems/repeated-dna-sequences/)

[28. 实现 strStr()](https://leetcode.cn/problems/implement-strstr/)



对于字符串匹配算法，一般可以通过「暴力」「KMP 算法」来解决。今天介绍一种基于滑动窗口的 RABIN KARP 字符匹配算法

**关于滑动窗口的详细总结可见 [滑动窗口技巧](./滑动窗口.html)**

### <font color=#1FA774>引入</font>

现在给定一个纯数字的字符串`s = "9876"`，如何把它转化成数字呢？

```mysql
int r = 0;
for (int i = 0; i < s.length(); i++) {
	r = r * 10 + s.charAt(i) - 'a';
}
```

那么如何去掉上面字符串的最高位呢？

```java
// 9876 - 9 * 10 ^ 3 = 876
r = r - (s.charAt(0) - '0') * (int) Math.pow(10, s.length() - 1);
```

下面介绍的 RABIN KARP 字符匹配算法就是基于这个原理！！

### <font color=#1FA774>重复的DNA序列</font>

**题目详情可见 [重复的DNA序列](https://leetcode.cn/problems/repeated-dna-sequences/)**

这个题目大家肯定可以想到暴力的方法：

```java
public List<String> findRepeatedDnaSequences(String s) {
    List<String> ans = new ArrayList<>();
    Map<String, Integer> map = new HashMap<>();
    for (int i = 0; i + 10 <= s.length(); i++) {
        String cur = s.substring(i, i + 10);
        map.put(cur, map.getOrDefault(cur, 0) + 1);
        if (map.get(cur) == 2) ans.add(cur);
    }
    return ans;
}
```

暴力的时间复杂度为：`O(NL)`，其中`L`是每次截取字符串的耗时

我们能不能想办法把这个`L`给去掉呢？？？？这样时间复杂度就直接是`O(N)`

把`'A'`, `'C'`, `'G'`,`'T'`映射成`0, 1, 2, 3`四个数字，所以字符串`ACGT`就变成了数`0123`

比较两个字符串是否相等，只需要比较对应的树是否相等即可！换句话来说，就是把比较字符串换成了比较字符串的 Hash 值

```java
public List<String> findRepeatedDnaSequences(String s) {
    // 先求出 s 中每个字符对应的数字
    int[] nums = new int[s.length()];
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (c == 'A') nums[i] = 0;
        else if (c == 'C') nums[i] = 1;
        else if (c == 'G') nums[i] = 2;
        else nums[i] = 3;
    }
    List<String> ans = new ArrayList<>();
    Map<Integer, Integer> map = new HashMap<>();
    int window = 0;
    int left = 0, right = 0;
    while (right < s.length()) {
        // 此时等价于 4 进制，所以✖️4 即可
        window = window * 4 + nums[right++];
        // 满足长度为 10
        if (right - left == 10) {
            map.put(window, map.getOrDefault(window, 0) + 1);
            if (map.get(window) == 2) ans.add(s.substring(left, right));
            // 删除高位数字
            window = window - nums[left++] * (int) Math.pow(4, 9);
        }
    }
    return ans;
}
```

滑动窗口算法本身的时间复杂度是`O(N)`，再看看窗口滑动的过程中的操作耗时，给`ans`添加子串的过程用到了`substring`方法需要`O(L)`的复杂度，但一般情况下`substring`方法不会调用很多次，只有极端情况 (比如字符串全都是相同的字符) 下才会每次滑动窗口时都调用`substring`方法

所以我们可以说这个算法一般情况下的平均时间复杂度是`O(N)`，极端情况下的时间复杂度会退化成`O(NL)`


### <font color=#1FA774>实现 strStr()</font>

**题目详情可见 [实现 strStr()](https://leetcode.cn/problems/implement-strstr/)**

这个题目大家肯定也可以想到暴力的方法：(和上一题暴力的方法大同小异)

```java
public int strStr(String haystack, String needle) {
    int n = needle.length();
    for (int i = 0; i + n <= haystack.length(); i++) {
        if (needle.equals(haystack.substring(i, i + n))) return i;
    }
    return -1;
}
```

这个题目也可以使用上面介绍的方法，但不同的点在于，本题的字符集比较大

虽然题目中说是只有小写英文字母，但是我们权当包含了整个 ASCII 码的集合，即有 256 位，相当于是 256 进制

所以问题就很明显了，无论用什么类型来存，肯定会溢出，这个时候「模运算」就可以大显身手了！！

使用「模运算」就会存在冲突的问题，我们从两个方面来解决这个问题：

- 尽量选取一个较大的素数当作「模数」
- 当数值相等时，为了避免相等，可以同时比较一下字符

```java
public int strStr(String haystack, String needle) {
    long mod = 1658598167;
    int L = needle.length();
    int R = 256;
    long RL = 1;
    for (int i = 1; i <= L - 1; i++) {
        RL = (RL * R) % mod;
    }
    // 求出 needle 对应的数值
    long need = 0;
    for (int i = 0; i < needle.length(); i++) {
        need = (R * need + needle.charAt(i)) % mod;
    }
    long window = 0;
    int left = 0, right = 0;
    while (right < haystack.length()) {
        int c = haystack.charAt(right++);
        window = ((window * R) % mod + c) % mod;
        if (right - left == needle.length()) {
            // 数值相等 且 字符相等
            if (window == need && needle.equals(haystack.substring(left, right))) return left;
            int d = haystack.charAt(left++);
            window = (window - (d * RL) % mod + mod) % mod;
        }
    }
    return -1;
}
```