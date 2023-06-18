# Spring 循环依赖

### <font color=#1FA774>什么是 Spring 循环依赖</font>

**循环依赖：**对象 A 中有一个属性为对象 B，对象 B 中有一个属性为对象 A，如下所示：

```java
class A {
    B b;
}
class B {
    A a;
}
```

如果不考虑 Spring 自动注入的话，循环依赖并不是问题，直接像下面这种方式初始化即可：

```java
A a = new A();
B b = new B();
a.b = b;
b.a = a;
```

但如果考虑 Spring 的话，循环依赖就是一个问题，因为在 Spring 中，一个 Bean 的创建流程可以粗略的分为三个步骤：(**更具体可见 [Bean 的生命周期](./Spring-IoC.html#bean-的生命周期)**)

- **实例化 (Instantiate)：**通过构造函数实例化 Bean，此时 Bean 实例的属性还未赋值
- **属性赋值 (Populate)：**为上一步 Bean 实例的属性赋值
- **初始化 (Initialize)：**执行初始化方法

在完成上面三个步骤后，Spring 才会将 Bean 实例添加到缓存中 (singletonObjects，一级缓存)，以后调用`getBean()`方法时先去缓存中找，缓存中不存在才会创建

```java
// 一级缓存底层是一个 Map
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```

现在模拟 Spring 创建 Bean 实例的过程：

- BeanA 实例化 -> BeanA 属性赋值 -> 发现属性中包含 BeanB -> 在一级缓存中判断是否存在 BeanB -> 不存在，转而实例化 BeanB
- BeanB 实例化 -> BeanB 属性赋值 -> 发现属性中包含 BeanA -> 在一级缓存中判断是否存在 BeanA -> 不存在，转而实例化 BeanA

此时就出现了死循环，用个图来更清楚表示：

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230619/0537361687124256oJMhqz1.svg)

### <font color=#1FA774>简单循环依赖的解决方法</font>

简单循环依赖也就是不存在 AOP 的循环依赖，只需要增加一个二级缓存 (earlySingletonObjects) 即可

```java
// 二级缓存底层是一个 Map
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

上部分说一级缓存是存放完成「实例化 -> 属性赋值 -> 初始化」三个步骤的 Bean 实例，那么二级缓存就是存在只完成了「实例化」的 Bean 实例

加入二级缓存后的过程如下图所示：

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230619/0548061687124886Bdn8IB2.svg)

首先会将完成实例化的 Bean 加入到二级缓存中，当 Bean 三个步骤都完成后，会将 Bean 从二级缓存中移除，并加入到一级缓存中，所以二级缓存中存放的是非完全品！！

当我们需要判断缓存是否存在 Bean 实例时，先去二级缓存中寻找，如果不存在再去一级缓存中寻找，这样就可以解决循环依赖的问题

同样的，下面模拟 Spring 创建 Bean 实例的过程：

- BeanA 实例化 -> BeanA 加入二级缓存 -> BeanA 属性赋值 -> 发现属性中包含 BeanB -> 在二级缓存中判断是否存在 BeanB -> 不存在，继续在一级缓存中判断是否存在 BeanB -> 不存在，转而实例化 BeanB
- BeanB 实例化 -> BeanB 加入二级缓存 -> BeanB 属性赋值 -> 发现属性中包含 BeanA -> 在二级缓存中判断是否存在 BeanB -> 存在，继续后续步骤 -> 完成创建，将 BeanB 从二级缓存移动到一级缓存，转回实例化 BeanA

到此为止，完美解决不存在 AOP 的循环依赖问题！！

### <font color=#1FA774>AOP 循环依赖的解决方法</font>

之所以存在 AOP 的循环依赖问题需要单独拿出来讨论，是因为 AOP 实际上访问的是代理对象，而代理对象是在目标对象完成初始化后执行`postProcessAfterInitialization`方法创建

代理对象和目标对象不是同一个对象，如果按照上面加入二级缓存的方案，那么 BeanB 实例给属性赋值的就是目标对象，但实际上应该赋值为代理对象才对，但代理对象又是在初始化后才创建，这就很难搞～

**关于上述 AOP 创建代理对象的细节可见 [Spring AOP 源码剖析](./Spring-AOP.html#spring-aop-源码剖析)**

Spring 在存在 AOP 时又加了一层缓存，称为三级缓存 (singletonFactories)

```java
// 三级缓存底层是一个 Map
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

与一级缓存和二级缓存不同的在于，三级缓存的 value 不是 Object，而是 ObjectFactory，它表示对象工厂，用来创建某个对象

在 Spring 中，创建 Bean 实例的源码主要在`doCreateBean`方法中：(将非主要部分省略，**详细分析可见 [Spring IoC](Spring-IoC.html)**)

```java
// AbstractAutowireCapableBeanFactory.java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {

    // 实例化 ...

    // 解决循环依赖的问题
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        // addSingletonFactory(String, ObjectFactory) 将 <String, ObjectFactory> 添加到 singletonFactories 中
        // getEarlyBeanReference() 返回一个 ObjectFactory 对象，具体见下方
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // 属性赋值 ...
    // 初始化 ...
}

protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                // 继续往下看
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}
```

从上面可以看到调用`SmartInstantiationAwareBeanPostProcessor`接口的`getEarlyBeanReference`方法，更具体是调用`AbstractAutoProxyCreator`中的该方法：

```java
// AbstractAutoProxyCreator.java
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    // 判断 earlyProxyReferences 中是否存在；earlyProxyReferences 底层是一个 Map
    if (!this.earlyProxyReferences.contains(cacheKey)) {
        this.earlyProxyReferences.add(cacheKey);
    }
    // 创建代理对象
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

`ObjectFactory`是一个函数式接口，上面通过 lambda 表达式生成了一个对应的实现类，当调用`getEarlyBeanReference`方法时会创建一个代理对象，而 ObjectFactory 对象存放在三级缓存中

此时，整个流程如下图所示：

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230619/0636121687127772gtLP1K3.svg)

从`doCreateBean`方法中可以看出，加入三级缓存也是在实例化完成之后，属性赋值之前

当从三级缓存中取，取出来的是一个`ObjectFactory`对象，调用创建该对象的 lambda 表达式指定的方法可以提前创建代理对象

为了避免重复创建代理对象，会将创建的代理对象存入`earlyProxyReferences`中，在创建代理对象前需要判断`earlyProxyReferences`是否存在，如果不存在才开始创建代理对象

### <font color=#1FA774>参考文章</font>

- **[Spring IoC](./Spring-IoC.html)**
- **[Spring AOP](./Spring-AOP.html)**
- **[Spring中的循环依赖及解决](https://juejin.cn/post/6985337310472568839)**