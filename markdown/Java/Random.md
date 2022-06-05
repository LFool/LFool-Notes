# Random()

`Random`类生成伪随机数流，`Random`类使用 48 位种子

`Random`的实例是线程安全的，但`Random`在并发中使用性能较差，可以在并发环境中使用`ThreadLocalRandom`

`Random`的实例在密码学上不是安全的，们可以使用`SecureRandom`来获得加密安全的伪随机数

`Math.random()`用于在更简单的用例中获取伪随机数

**<font color='red'>注意：</font>**使用`Random`只得到伪随机数而不是实际随机数。伪随机数是使用确定性过程生成的，它们在统计上近似是随机的

### <font color=#1FA774>Random</font>

`Random`有两种构造方法：

- `Random()`：创建一个新的随机数生成器
- `Random(long seed)`：使用单个 long 种子创建一个新的随机数生成器

**下面介绍`Random`的方法**

- `doubles()`：返回伪随机 double 的流
- `ints()`：返回伪随机 int 的流
- `longs()`：返回伪随机 long 的流

上面三个方法均有三个可选参数：`streamSize`，`randomNumberOrigin`，`randomNumberBound`

例如：`random.doubles(5, 0, 10)`表示返回 5 个范围在`[0, 10)`的随机数；注意：范围区间是**「前闭后开」**

**再介绍一些其他方法**

- `nextBoolean()`：返回下一个伪随机 boolean 值
- `nextBytes(byte[] bytes)`：生成随机字节
- `nextDouble()`：返回 0.0 到 1.0 之间的下一个伪随机 double 值
- `nextFloat()`：返回 0.0 到 1.0 之间的下一个伪随机 float 值
- `nextGaussian()`：返回下一个伪随机、高斯分布 double 值，均值为 0.0，标准差为 1.0
- `nextInt()`：返回下一个伪随机 int 值
- `nextLong()`返回下一个伪随机 long 值
- `setSeed()`：设置随机数生成器的种子

**注意：**

- `nextDouble()`和`nextFloat()`返回的都是`[0.0, 1.0)`范围内的一个值；注意：范围区间是**「前闭后开」**
- `nextInt()`和`nextLong()`分别返回一个 32 位和 64 位的值
- `nextInt(int bound)`返回一个`[0, bound)`范围内的一个值；注意：范围区间是**「前闭后开」**

下面给出几个例子：

```java
Random random = new Random();

// random.doubles(streamSize, randomNumberOrigin, randomNumberBound) 方法的应用例子
double[] rd = random.doubles(5, 0, 10).toArray();

// nextBytes(byte[] bytes) 方法的应用例子
byte[] bytes = new byte[5];
random.nextBytes(bytes);
System.out.println(Arrays.toString(bytes));

// nextDouble() 方法的应用例子
// 结果：0.35761840335259487
random.nextDouble();

// nextInt() 方法的应用例子
// 结果：871207694
random.nextInt();

// nextInt(bound)
// 结果：3
random.nextInt(5)
```

### <font color=#1FA774>随机种子</font>

先看一个例子：

```java
Random ran1 = new Random(10);
System.out.println("使用种子为 10 的 Random 对象生成 [0,10) 内随机整数序列: ");
for (int i = 0; i < 10; i++) {
    System.out.print(ran1.nextInt(10) + " ");
}
System.out.println();
Random ran2 = new Random(10);
System.out.println("使用另一个种子为 10 的 Random 对象生成 [0,10) 内随机整数序列: ");
for (int i = 0; i < 10; i++) {
    System.out.print(ran2.nextInt(10) + " ");
}
```

输出结果：

```
使用种子为 10 的 Random 对象生成 [0,10) 内随机整数序列: 
3 0 3 0 6 6 7 8 1 4 
使用另一个种子为 10 的 Random 对象生成 [0,10) 内随机整数序列: 
3 0 3 0 6 6 7 8 1 4 
```

不难发现生成的两组 10 个随机数都是一样的，这是因为我们使用了相同的随机种子

**<font color='red'>对于种子相同的Random对象，生成的随机数序列是一样的</font>**

在没带参数构造函数生成的 Random 对象的种子缺省是**「当前系统时间的毫秒数」**

所以在高并发下，每个用户线程生成随机数时，可能出现「系统时间的毫秒数」相同的情况

如果这个时候我们并没有自定义随机种子，极有可能出现多个用户生成的随机数相同的情况

具体可以看下面这个例子：

```java
public class MyThread {
    private static final int[] cnt = new int[1000000000];

    private static void createThread() {
        new Thread(() -> {
            // 创建 100 个 funThread 线程
            for (int i = 0; i < 100; i++) {
                funThread();
            }
        }).start();
    }

    private static void funThread() {
        new Thread(MyThread::testMethod).start();
    }

    private static void testMethod() {
        Random random = new Random();
        cnt[random.nextInt(1000000000)]++;
    }

    public static void main(String[] args) throws InterruptedException {
        // 创建 100 个线程
        for (int i = 0; i < 1000; i++) {
            createThread();
        }
        Thread.sleep(10000);
        for (int i = 0; i < 1000000000; i++) {
            if (cnt[i] > 1) {
                System.out.println(i + "_" + cnt[i]);
            }
        }
    }
}
```

### <font color=#1FA774>Math.random()</font>

`Math.random()`返回`[0.0, 1.0)`正伪随机 double 值，和`random.nextDouble();`功能等价
