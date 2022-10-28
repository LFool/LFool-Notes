**<font color='red'>「JVM 类加载子系统」系列文章</font>**

- **[类加载的过程](./类加载的过程.html)**
- **[类加载的时机](./类加载的时机.html)**
- **[类加载器](./类加载器.html)**
- **[❤️‍🔥 剖析 [Bootstrap、Extension、Application] ClassLoader](./剖析-Bootstrap-Extension-Application-ClassLoader.html)**
- **[双亲委派模型](./双亲委派模型.html)**
- **[自定义类加载器](./自定义类加载器.html)**
- **[破坏双亲委派模型](./破坏双亲委派模型.html)**

# 剖析 [Bootstrap、Extension、Application] ClassLoader

本篇文章主要介绍三个主要类加载器的关系！！

我们需要知道的一个前提：Bootstrap ClassLoader 是 C++ 实现，而 Extension ClassLoader、Application ClassLoader 均为 Java 实现

![133](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20221026/0003221666713802MIwUJF133.svg)

如上图所示，左边是 C++ 实现，右边是 Java 实现。这里是跨语言调用，JNI 实现了由 C++ 向 Java 跨语言调用。C++ 调用的第一个 Java 类是 Launcher 类

从这个图中我们可以看出，C++ 调用 Java 创建 JVM 启动器，其中一个启动器是 Launcher，它实际是调用了`sun.misc.Launcher`类的`getLauncher()`方法

那我们就从这个方法入手看看到底是如何运行的？！

```java
public class Launcher {
    // 省略其它代码 ...
    private static Launcher launcher = new Launcher();
    // C++ 调用该方法
    public static Launcher getLauncher() {
        return launcher;
    }
}
```

可以明显看出这就是一个单例模式！！`launcher`对象在类加载完成的时候就已经初始化好了，C++ 调用`getLauncher()`方法的时候会返回`launcher`对象

我们先来看看`Launcher`的构造方法，它主要就是初始化了两个构造器：扩展类加载器、应用程序类加载器

```java
public Launcher() {
    // Create the extension class loader
    ClassLoader extcl;
    try {
        // 获取扩展类加载器，getExtClassLoader() 见下方
        extcl = ExtClassLoader.getExtClassLoader();
    } catch (IOException e) {
        throw new InternalError(
            "Could not create extension class loader", e);
    }
    // Now create the class loader to use to launch the application
    try {
        // 获取应用程序类加载器
        loader = AppClassLoader.getAppClassLoader(extcl);
    } catch (IOException e) {
        throw new InternalError(
            "Could not create application class loader", e);
    }
    // 省略其它代码 ...
}
```

在获取扩展类加载器的时候，调用`getExtClassLoader()`方法，具体见下面代码的流程：

```java
static class ExtClassLoader extends URLClassLoader {

    static {
        ClassLoader.registerAsParallelCapable();
    }
    private static volatile ExtClassLoader instance = null;

    public static ExtClassLoader getExtClassLoader() throws IOException {
        // 单例模式：双重校验锁🔒
        if (instance == null) {
            synchronized(ExtClassLoader.class) {
                if (instance == null) {
                    // createExtClassLoader() 见下方
                    instance = createExtClassLoader();
                }
            }
        }
        return instance;
    }
    private static ExtClassLoader createExtClassLoader() throws IOException {
        try {
            // doPrivileged 是一个权限校验的操作，可以先不管
            return AccessController.doPrivileged(
                new PrivilegedExceptionAction<ExtClassLoader>() {
                    public ExtClassLoader run() throws IOException {
                        final File[] dirs = getExtDirs();
                        int len = dirs.length;
                        for (int i = 0; i < len; i++) {
                            MetaIndex.registerDirectory(dirs[i]);
                        }
                        // 直接 new 了一个 ExtClassLoader，参数 dirs 代表扩展目录下的文件
                        // ExtClassLoader(dirs) 见下方
                        return new ExtClassLoader(dirs);
                    }
                });
        } catch (java.security.PrivilegedActionException e) {
            throw (IOException) e.getException();
        }
    }
    public ExtClassLoader(File[] dirs) throws IOException {
        // 调用父类构造方法，ExtClassLoader 的父类是 URLClassLoader
        // URLClassLoader 见下方
        super(getExtURLs(dirs), null, factory);
        SharedSecrets.getJavaNetAccess().
            getURLClassPath(this).initLookupCache(this);
    }
}
// 通过文件路径加载 class 类
public class URLClassLoader extends SecureClassLoader implements Closeable {
    // 省略其它代码 ...
    // 构造函数
    public URLClassLoader(URL[] urls, ClassLoader parent,
                          URLStreamHandlerFactory factory) {
        // 调用父类构造方法，URLClassLoader 的父类是 SecureClassLoader
        // SecureClassLoader 见下方
        super(parent);
        // this is to make the stack depth consistent with 1.1
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkCreateClassLoader();
        }
        acc = AccessController.getContext();
        ucp = new URLClassPath(urls, factory, acc);
    }
}
public class SecureClassLoader extends ClassLoader {
    // 省略其它代码 ...
    protected SecureClassLoader(ClassLoader parent) {
        // 调用父类构造方法，SecureClassLoader 的父类是 ClassLoader
        // ClassLoader 见下方
        super(parent);
        // this is to make the stack depth consistent with 1.1
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkCreateClassLoader();
        }
        initialized = true;
    }
}
public abstract class ClassLoader {
    // 省略其它代码 ...
    protected ClassLoader(ClassLoader parent) {
        // 见下方
        this(checkCreateClassLoader(), parent);
    }
    private ClassLoader(Void unused, ClassLoader parent) {
        // 这里的 parent 是 ExtClassLoader 中 super(getExtURLs(dirs), null, factory) 一路传下来的，所以为 null
        this.parent = parent;
        if (ParallelLoaders.isRegistered(this.getClass())) {
            parallelLockMap = new ConcurrentHashMap<>();
            package2certs = new ConcurrentHashMap<>();
            assertionLock = new Object();
        } else {
            parallelLockMap = null;
            package2certs = new Hashtable<>();
            assertionLock = this;
        }
    }
}
```

在获取应用程序类加载器的时候，调用`getAppClassLoader()`方法，具体见下面代码的流程：

```java
static class AppClassLoader extends URLClassLoader {

    static {
        ClassLoader.registerAsParallelCapable();
    }
    // 参数 extcl 是上面 AppClassLoader.getAppClassLoader(extcl) 传下来的 扩展类加载器
    public static ClassLoader getAppClassLoader(final ClassLoader extcl)
        throws IOException
    {
        // 获取当前项目的 class 文件路径
        final String s = System.getProperty("java.class.path");
        final File[] path = (s == null) ? new File[0] : getClassPath(s);
        
        return AccessController.doPrivileged(
            new PrivilegedAction<AppClassLoader>() {
                public AppClassLoader run() {
                    // 将上面的获取到的 class 文件路径转化为 URL
                    URL[] urls =
                        (s == null) ? new URL[0] : pathToURLs(path);
                    // urls 表示 class 类所在的路径集合
                    // extcl 表示 扩展类加载器
                    // AppClassLoader(urls, extcl) 见下方
                    return new AppClassLoader(urls, extcl);
                }
            });
    }
    AppClassLoader(URL[] urls, ClassLoader parent) {
        // 调用父类构造方法，AppClassLoader 的父类是 URLClassLoader
        // URLClassLoader 见下方
        super(urls, parent, factory);
        ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);
        ucp.initLookupCache(this);
    }
}

public class URLClassLoader extends SecureClassLoader implements Closeable {
	public URLClassLoader(URL[] urls, ClassLoader parent,
                          URLStreamHandlerFactory factory) {
        // 调用父类构造方法，URLClassLoader 的父类是 SecureClassLoader
        // SecureClassLoader 见下方
        super(parent);
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkCreateClassLoader();
        }
        acc = AccessController.getContext();
        ucp = new URLClassPath(urls, factory, acc);
    }
}

public class SecureClassLoader extends ClassLoader {

	protected SecureClassLoader(ClassLoader parent) {
        // 调用父类构造方法，SecureClassLoader 的父类是 ClassLoader
        // ClassLoader 见下方
        super(parent);
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkCreateClassLoader();
        }
        initialized = true;
    }
}

public abstract class ClassLoader {
    // 省略其它代码 ...
    protected ClassLoader(ClassLoader parent) {
        // 见下方
        this(checkCreateClassLoader(), parent);
    }
    private ClassLoader(Void unused, ClassLoader parent) {
        // 这里的 parent 是 AppClassLoader 中 new AppClassLoader(urls, extcl); 一路传下来的，所以为 扩展类加载器
        this.parent = parent;
        if (ParallelLoaders.isRegistered(this.getClass())) {
            parallelLockMap = new ConcurrentHashMap<>();
            package2certs = new ConcurrentHashMap<>();
            assertionLock = new Object();
        } else {
            parallelLockMap = null;
            package2certs = new Hashtable<>();
            assertionLock = this;
        }
    }
}
```

**<font color='red'>结论：</font>**

- C++ 在启动 JVM 的时候，调用了 Launcher 启动类，这个启动类同时加载了 ExtClassLoader 和 AppClassLoader
- appClassLoader 的父类是 extClassLoader，extClassLoader 的父类是 bootstrapClassLoader
