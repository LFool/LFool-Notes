# 数组中的第K个最大元素「变题」

[215. 数组中的第K个最大元素](https://leetcode.cn/problems/kth-largest-element-in-an-array/)



开头先来一个小插曲，**关于「快排」相关总结可见 [详解快排及其应用](./详解快排及其应用.html)**；**关于「详解堆排序」相关总结可见 [详解堆排序「优先队列」](./详解堆排序-优先队列.html)** 

这个题可以用两种方法：堆排序 && 快排

#### <font color=#9933FF>方法一：堆排序</font>

看到很多人说不能使用 Java 中的 `PriorityQueue`，需要自己手动实现，所以这里使用自己实现的小根堆

时间复杂度：`O(NlogK)`；空间复杂度：`O(K)`

```java
class Solution {
    public int findKthLargest(int[] nums, int k) {
        MyPQ pq = new MyPQ(k);
        for (int num : nums) {
            if (pq.size() < k) pq.offer(num);
            else if (pq.peek() < num) {
                pq.poll();
                pq.offer(num);
            }
        }
        return pq.peek();
    }
}
class MyPQ {
    private static final int CAPACITY = 11;
    private int[] pq;
    private int size;
    public MyPQ() {
        pq = new int[CAPACITY];
        size = 0;
    }
    public MyPQ(int c) {
        pq = new int[c + 1];
        size = 0;
    }
    public void offer(int x) {
        pq[++size] = x;
        up(size);
    }
    public int poll() {
        int x = pq[1];
        swap(1, size--);
        down(1);
        return x;
    }
    public int peek() {
        return pq[1];
    }
    public int size() {
        return size;
    }
    private void up(int k) {
        while (k > 1 && pq[k] < pq[k / 2]) {
            swap(k, k / 2);
            k >>= 1;
        }
    }
    private void down(int k) {
        while (2 * k <= size) {
            int j = 2 * k;
            if (j < size && pq[j] > pq[j + 1]) j++;
            if (pq[k] < pq[j]) break;
            swap(k, j);
            k = j;
        }
    }
    private void swap(int i, int j) {
        int t = pq[i];
        pq[i] = pq[j];
        pq[j] = t;
    }
}
```

#### <font color=#9933FF>方法二：快排</font>

时间复杂度：`O(N)`；空间复杂度：`O(1)`

**关于时间复杂度的分析，可见 [时间复杂度分析](./详解快排及其应用.html#算法应用)**

```java
private int k;
private Random r = new Random();
public int findKthLargest(int[] nums, int k) {
    this.k = k;
    return sort(nums, 0, nums.length - 1);
}
private int sort(int[] nums, int lo, int hi) {
    int p = partition(nums, lo, hi);
    if (p > k - 1) return sort(nums, lo, p - 1);
    else if (p < k - 1) return sort(nums, p + 1, hi);
    return nums[p];
}
private int partition(int[] nums, int lo, int hi) {
    int p = r.nextInt(hi - lo + 1) + lo;
    swap(nums, lo, p);
    int pivot = nums[lo];
    int i = lo + 1, j = hi;
    while (i <= j) {
        while (i <= hi && nums[i] >= pivot) i++;
        while (j > lo && nums[j] < pivot) j--;
        if (i >= j) break;
        swap(nums, i, j);
    }
    swap(nums, lo, j);
    return j;
}
private void swap(int[] nums, int i, int j) {
    int t = nums[i];
    nums[i] = nums[j];
    nums[j] = t;
}
```

### <font color=#1FA774>变题一：无序数组找中位数</font>

对于一组有序数据：$X_1, X_2, \cdots, X_n$

当 $n$ 为奇数时，中位数：$X_{(n + 1) / 2}$；当 $n$ 为偶数时，中位数：$\frac{X_{n/2} + X_{n/2 + 1}}{2}$

所以，该问题转化为：当 $n$ 为奇数时，求第 $(n+1)/2$ 大；当 $n$ 为偶数时，求第 $n/2$ 和 $n/2 + 1$ 大，然后取平均值

### <font color=#1FA774>变题二：判断一个数是否为第k大 (考虑数组中有重复元素)</font>

假设求第 2 大，此时数组为：`[5, 5, 4, 4, 4, 3]`

按照上述方法求，那么第 2 大的数为 5，但此时第二大的数应该为 4

这里使用「堆排序」的方法，用一个`Set`记录已经出现过的数字，如果再次出现，则跳过

具体代码如下：

```java
public int findKthLargest(int[] nums, int k) {
    MyPQ pq = new MyPQ(k);
    Set<Integer> set = new HashSet<>();
    for (int num : nums) {
        if (set.contains(num)) continue;
        else set.add(num);
        if (pq.size() < k) pq.offer(num);
        else if (pq.peek() < num) {
            pq.poll();
            pq.offer(num);
        }
    }
    return pq.peek();
}
```