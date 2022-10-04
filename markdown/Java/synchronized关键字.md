# synchronized 关键字

### <font color=#1FA774>关于锁和同步</font>

**类锁 vs 对象锁**

- 类锁：类的所有对象共享一把锁🔒
- 对象锁：每个对象拥有一把锁 🔒

同步是对同一把锁而言的，同步这个概念是在多个线程争夺同一把锁的时候才能实现的，如果多个线程争夺不同的锁，那多个线程是不能同步的

- 两个线程：一个取对象锁，一个取类锁，则不能同步
- 两个线程：一个取对象 A 的锁，一个取对象 B 的锁，则不能同步

### <font color=#1FA774>锁和 synchronized 的关系</font>

锁是 Java 中用来实现同步的工具

而之所以能对方法或者代码块实现同步的原因就是：只有拿到锁的线程才能执行 synchronized 修饰的方法或代码块，且其他线程获得锁的唯一方法是等目前拿到锁的线程执行完方法将锁释放，如果 synchronized 修饰的代码块或方法没执行完是不会释放这把锁的，这就保证了拿到锁的线程一定可以一次性把它调用的方法或代码块执行完

### <font color=#1FA774>synchronized 修饰的类型</font>

若 synchronized 修饰「静态方法」或者修饰「类的代码块」，则为类锁 🔒

若 synchronized 修饰「普通方法」或者修饰「当前对象的代码块」，则为对象锁 🔒

#### <font color=#9933FF>修饰方法</font>

- 修饰静态方法：静态方法属于类，所有对象调用的都是同一个方法
- 修饰普通方法：普通方法属于对象，每个对象调用的都是自己的方法

**<font color='red'>注意：</font>**

- 在定义接口方法时不能使用 synchronized 关键字
- 构造方法不能使用 synchronized 关键字，但可以使用 synchronized 代码块来进行同步
- synchronized 关键字不能被继承。如果子类覆盖了父类被 synchronized 关键字修饰的方法，那么子类的该方法只要没有 synchronized 关键字，那么就默认没有同步

```java
public class A {
    // 修饰静态方法
    public static synchronized void f1(int c) {
        for (int i = 0; i < 10; i++) {
            System.out.println("静态方法 " + c);
        }
    }
    // 修饰普通方法
    public synchronized void f2(int c) {
        for (int i = 0; i < 10; i++) {
            System.out.println("普通方法" + c);
        }
    }
    public static void main(String[] args) {
        // 测试静态方法
        // f1() 属于类 A，所以下方两个线程存在同步，一定是先输出所有「静态方法 1」，然后再输出所有「静态方法 2」
        new Thread(() -> A.f1(1)).start();
        new Thread(() -> A.f1(2)).start();
        // 测试普通方法
        A a1 = new A();
        A a2 = new A();
        // f2() 属于类 A 的对象，所以下方两个线程不存在同步，一个取对象 a1 的锁，一个取对象 a2 的锁
        // 输出内容「普通方法 1」「普通方法 2」无规则交替出现
        new Thread(() -> a1.f2(1)).start();
        new Thread(() -> a2.f2(2)).start();
    }
}
```

#### <font color=#9933FF>修饰代码块</font>

- 修饰类：表示对类进行同步
- 修饰当前对象：表示对当前对象进行同步

```java
public class B {
    public void f1(int c) {
        // 修饰类
        synchronized (B.class) {
            for (int i = 0; i < 10; i++) {
                System.out.println("修饰类 " + c);
            }
        }
    }
    public void f2(int c) {
        // 修饰当前对象
        synchronized (this) {
            for (int i = 0; i < 10; i++) {
                System.out.println("修饰当前对象 " + c);
            }
        }
    }
    public static void main(String[] args) {
        B b1 = new B();
        B b2 = new B();
        // 测试修饰类
        // 同步对象为类的所有对象，一定是先输出所有「修饰类 1」，然后再输出所有「修饰类 2」
        new Thread(() -> b1.f1(1)).start();
        new Thread(() -> b2.f1(2)).start();
        // 测试修饰当前对象
        // 同步对象为当前对象，下方两个线程不存在同步，一个取对象 b1 的锁，一个取对象 b2 的锁
        // 输出内容「修饰当前对象 1」「修饰当前对象 2」无规则交替出现
        new Thread(() -> b1.f2(1)).start();
        new Thread(() -> b2.f2(2)).start();
    }
}
```
