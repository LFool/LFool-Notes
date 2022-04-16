# nSum

[1. 两数之和](https://leetcode-cn.com/problems/two-sum/)

[15. 三数之和](https://leetcode-cn.com/problems/3sum/)

[18. 四数之和](https://leetcode-cn.com/problems/4sum/)

 

这一类型的题目意思很简单，就是 n 个数的和等于目标值

有一种无限套娃的感觉。哈哈哈哈哈哈！！！

### <font color=#1FA774>两数和</font>

利用有序数组双指针的思想，左右收缩

```java
public List<List<Integer>> twoSum(int[] nums, int target) {
    // 数组需要排序
    Arrays.sort(nums);
    // 存储结果
    List<List<Integer>> res = new ArrayList<>();
    // 注意开闭，全闭区间
    int lo = 0, hi = nums.length - 1;
    // 结束条件：lo == hi
    while (lo < hi) {
        int left = nums[lo], right = nums[hi];
        int sum = left + right;
        if (sum < target) {
            // 排除重复元素
            while (lo < hi && left == nums[lo]) lo++;
        } else if (sum > target) {
            // 排除重复元素
            while (lo < hi && right == nums[hi]) hi--;
        } else {
            List<Integer> list = new ArrayList<>();
            list.add(left);
            list.add(right);
            res.add(list);
            // 排除重复元素
            while (lo < hi && left == nums[lo]) lo++;
            while (lo < hi && right == nums[hi]) hi--;
        }
    }
    return res;
}
```

### <font color=#1FA774>三数和</font>

「三数和」利用「二数和」，即：确定一个数，就变成了「二数和」

```java
public List<List<Integer>> threeSum(int[] nums) {
    // 排序
    Arrays.sort(nums);
    List<List<Integer>> res = new ArrayList<>();
    for (int i = 0; i < nums.length; i++) {
        // 确定 num[i]，从 i + 1 开始寻找「二数和」
        List<List<Integer>> twoSumList = twoSum(nums, i + 1, 0 - nums[i]);
        // 结果汇合
        for (List<Integer> list : twoSumList) {
            list.add(nums[i]);
            res.add(list);
        }
        // 排除重复元素
        while (i < nums.length - 1 && nums[i] == nums[i + 1]) i++;
    }
    return res;
}
// start : 开始下标，不再从 0 开始
private List<List<Integer>> twoSum(int[] nums, int start, int target) {
    List<List<Integer>> res = new ArrayList<>();
    int lo = start, hi = nums.length - 1;
    while (lo < hi) {
        int left = nums[lo], right = nums[hi];
        int sum = left + right;
        if (sum < target) {
            while (lo < hi && left == nums[lo]) lo++;
        } else if (sum > target) {
            while (lo < hi && right == nums[hi]) hi--;
        } else {
            List<Integer> list = new ArrayList<>();
            list.add(left);
            list.add(right);
            res.add(list);
            while (lo < hi && left == nums[lo]) lo++;
            while (lo < hi && right == nums[hi]) hi--;
        }
    }
    return res;
}
```

### <font color=#1FA774>四数和</font>

「四数和」利用「三数和」，即：确定一个数，就变成了「三数和」

```java
public List<List<Integer>> fourSum(int[] nums, int target) {
    // 排序
    Arrays.sort(nums);
    List<List<Integer>> res = new ArrayList<>();
    for (int i = 0; i < nums.length; i++) {
        // 确定 num[i]，从 i + 1 开始寻找「三数和」
        List<List<Integer>> threeSumList = threeSum(nums, i + 1, target - nums[i]);
        for (List<Integer> list : threeSumList) {
            list.add(nums[i]);
            res.add(list);
        }
        // 排除重复元素
        while (i < nums.length - 1 && nums[i] == nums[i + 1]) i++;
    }
    return res;
}
// start : 开始下标，不再从 0 开始
private List<List<Integer>> threeSum(int[] nums, int start, int target) {
    List<List<Integer>> res = new ArrayList<>();
    for (int i = start; i < nums.length; i++) {
        List<List<Integer>> twoSumList = twoSum(nums, i + 1, target - nums[i]);
        for (List<Integer> list : twoSumList) {
            list.add(nums[i]);
            res.add(list);
        }
        while (i < nums.length - 1 && nums[i] == nums[i + 1]) i++;
    }
    return res;
}
private List<List<Integer>> twoSum(int[] nums, int start, int target) {
    List<List<Integer>> res = new ArrayList<>();
    int lo = start, hi = nums.length - 1;
    while (lo < hi) {
        int left = nums[lo], right = nums[hi];
        int sum = left + right;
        if (sum < target) {
            while (lo < hi && left == nums[lo]) lo++;
        } else if (sum > target) {
            while (lo < hi && right == nums[hi]) hi--;
        } else {
            List<Integer> list = new ArrayList<>();
            list.add(left);
            list.add(right);
            res.add(list);
            while (lo < hi && left == nums[lo]) lo++;
            while (lo < hi && right == nums[hi]) hi--;
        }
    }
    return res;
}
```

### <font color=#1FA774>N 数和</font>

**<font color='red'>看了上面的「二数和」「三数和」「四数和」，是否有一种无限套娃的感觉</font>**

现在将上面的代码重构，变成 N 数和

```java
// nums : 有序数组
// start : 开始下标
// target : 目标数
// k : 数量
public List<List<Integer>> nSum(int[] nums, int start, int target, int k) {
    int n = nums.length;
    List<List<Integer>> res = new ArrayList<>();
    // k > 2 : 递归调用 k - 1 k -2 k - 3
    if (k > 2) {
        for (int i = start; i < n; i++) {
            List<List<Integer>> sumList = nSum(nums, i + 1, target - nums[i], k - 1);
            for (List<Integer> list : sumList) {
                list.add(nums[i]);
                res.add(list);
            }
            while (i < n - 1 && nums[i] == nums[i + 1]) i++;
        }
    } else {
        // k = 2 : 使用「二数和」
        int lo = start, hi = n - 1;
        while (lo < hi) {
            int left = nums[lo], right = nums[hi];
            int sum = left + right;
            if (sum < target) {
                while (lo < hi && left == nums[lo]) lo++;
            } else if(sum > target) {
                while (lo < hi && right == nums[hi]) hi--;
            } else {
                List<Integer> list = new ArrayList<>();
                list.add(left);
                list.add(right);
                res.add(list);
                while (lo < hi && left == nums[lo]) lo++;
                while (lo < hi && right == nums[hi]) hi--;
            }
        }
    }
    return res;
}
```

至此，大功告成！！！！