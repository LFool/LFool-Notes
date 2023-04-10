# x 的平方根「变题」

[69. x 的平方根](https://leetcode.cn/problems/sqrtx/)



```java
public int mySqrt(int x) {
    int l = 0, r = x;
    while (l < r) {
        int m = l + r + 1 >> 1;
        if (x / m >= m) l = m;
        else r = m - 1;
    }
    return l;
}
```

### <font color=#1FA774>变题一：保留小数位</font>

```java
// epsilon 保留小数位 -> 1e-7
public double mySqrt(double x, double epsilon){
    double l = 0, r = x;
    if (x == 0 || x == 1){
        return x;
    }
    while (l < r){
        double mid = l - (l - r) / 2;
        if (Math.abs(mid * mid - x) < epsilon){
            return mid;
        } else if (mid * mid < x){
            l = mid;
        } else {
            r = mid;
        }
    }
    return l;
}
```

