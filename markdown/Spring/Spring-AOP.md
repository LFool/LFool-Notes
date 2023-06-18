# Spring AOP

### <font color=#1FA774>什么是 AOP</font>

AOP (Aspect-Oriented Programming，面向切面编程) 能够将那些与业务无关，却被业务共同调用的逻辑 (如：事务处理、日志管理、权限控制等) 封装起来，降低模块间的耦合程度，利于扩展和维护

AOP 是基于 **[动态代理](../java/代理模式-静态-动态.html#动态代理)** 实现，说白了就是在原有代码基础上进行一定的包装，如：在方法执行前、方法返回后、方法抛出异常后等地方进行一定的拦截处理或者叫增强处理，和动态代理的思想一模一样

更具体点，AOP 的实现是有一个代理类，实际执行的其实是生成的代理类对象，并非真正的目标类对象，由代理类对象真正的调用目标类对象的方法，在调用前后可以进行一些特殊处理，称之为增强处理

在动态代理中介绍有两种实现方式，JDK 动态代理适用于实现了接口的目标类，CGLIB 动态代理适用于没有实现接口的目标类，所以 AOP 也是如此，因为它底层的实现就是动态代理

### <font color=#1FA774>AOP 应用场景</font>

**日志记录：**日志可以用于系统审计、调试、性能检测等，在方法执行前后或异常抛出时自动记录日志，而不需要在每个方法中都编写记录日志的逻辑

**事务管理：**在方法执行前开始事务，在方法执行后根据执行结果提交或回滚事务

**安全控制：**在方法执行前进行权限检查，验证用户是否具有执行该方法的权限

**性能监控：**在方法执行前后计算方法执行时间，统计方法调用次数，用于性能监测和优化

**缓存管理：**在方法执行前检查缓存中是否存在所需数据，如果存在直接返回，不需要执行方法

**异常处理：**在方法抛出异常后统一处理，进行异常的记录、转换、重试等操作

**<font color='red'>总结：</font>**这些应用场景无非就是将多个方法中重复的代码逻辑抽离出来，放到代理类中统一处理，这样就可以不需要在每个方法中都编写相同逻辑的代码，减少冗余，增强扩展性

### <font color=#1FA774>AOP Demo</font>

下面给五个从不同角度实现 Spring AOP 的小 Demo，都是基于 XML 配置文件实现滴～～

先给出这五个例子中使用到的依赖包：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>5.0.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.0.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.28</version>
</dependency>
```

再给出这五个例子都会用到的类：

```java
// package com.lfool.entity
@Data
public class User {
    private String firstName;
    private String lastName;
    private int age;
    private String address;
}
@Data
public class Order {
    private String username;
    private String product;
}
// package com.lfool.service
public interface UserService {
    User createUser(String firstName, String lastName, int age);
    User queryUser();
}
public interface OrderService {
    Order createOrder(String username, String product);
    Order queryOrder(String username);
}
// package com.lfool.service.impl
public class UserServiceImpl implements UserService {
    private static User user = null;
    @Override
    public User createUser(String firstName, String lastName, int age) {
        user = new User();
        user.setFirstName(firstName);
        user.setLastName(lastName);
        user.setAge(age);
        return user;
    }

    @Override
    public User queryUser() {
        return user;
    }
}
public class OrderServiceImpl implements OrderService {
    private static Order order = null;
    @Override
    public Order createOrder(String username, String product) {
        order = new Order();
        order.setUsername(username);
        order.setProduct(product);
        return order;
    }

    @Override
    public Order queryOrder(String username) {
        return order;
    }
}
// package com.lfool.interceptor
// 拦截方法执行前，输出执行方法的参数
public class LogArgsAdvice implements MethodBeforeAdvice {
    @Override
    public void before(Method method, Object[] args, Object o) throws Throwable {
        System.out.println("准备执行方法：" + method.getName() + ", 参数列表：" + Arrays.toString(args));
    }
}
// 拦截方法执行后，输出执行方法的结果
public class LogResultAdvice implements AfterReturningAdvice {
    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object o1) throws Throwable {
        System.out.println("方法返回：" + returnValue);
    }
}
```

#### <font color=#9933FF>Advice 实现</font>

先给出 XML 配置文件`spring_1.2_advice.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userServiceImpl" class="com.lfool.service.impl.UserServiceImpl" />
    <bean id="orderServiceImpl" class="com.lfool.service.impl.OrderServiceImpl" />

    <!-- 定义两个 advice -->
    <bean id="logArgsAdvice" class="com.lfool.interceptor.LogArgsAdvice" />
    <bean id="logResultAdvice" class="com.lfool.interceptor.LogResultAdvice" />

    <bean id="useServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <!-- 代理的接口 -->
        <property name="proxyInterfaces">
            <list>
                <value>com.lfool.service.UserService</value>
            </list>
        </property>
        <!-- 代理的具体实现 -->
        <property name="target" ref="userServiceImpl" />
        <!-- 配置拦截器，这里可以配置 advice、advisor、interceptor -->
        <property name="interceptorNames">
            <list>
                <value>logArgsAdvice</value>
                <value>logResultAdvice</value>
            </list>
        </property>
    </bean>

    <!-- =========================================== -->
    <!-- 同理，我们也可以配置一个 orderServiceProxy...  -->
    <!-- =========================================== -->

</beans>
```

然后给出测试类：

```java
public class Main {
    public static void main(String[] args) {
        // 启动 Spring IoC 容器
        ApplicationContext context = new ClassPathXmlApplicationContext("spring_1.2_advice.xml");
        UserService userService = (UserService) context.getBean("useServiceProxy");

        userService.createUser("Tom", "Cruise", 55);
        userService.queryUser();
    }
}
```

输出结果如下：

```
准备执行方法：createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User(firstName=Tom, lastName=Cruise, age=55, address=null)
准备执行方法：queryUser, 参数列表：[]
方法返回：User(firstName=Tom, lastName=Cruise, age=55, address=null)
```

从这个例子中可以看出 Advice 实现的两点局限性：

- Advice 实现的拦截器粒度是类级别，也就是会拦截目标类所有方法
- 需要在 XML 配置文件中为每个类配置一个代理类

#### <font color=#9933FF>Advisor 实现</font>

为了解决 Advice 实现的第一点局限性，增加了 Advisor 实现，它可以匹配到方法，也就是可以只拦截指定方法，而非目标类的所有方法

先给出 XML 配置文件`spring_1.2_advisor.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userServiceImpl" class="com.lfool.service.impl.UserServiceImpl" />
    <bean id="orderServiceImpl" class="com.lfool.service.impl.OrderServiceImpl" />

    <!-- 定义两个 advice -->
    <bean id="logArgsAdvice" class="com.lfool.interceptor.LogArgsAdvice" />
    <bean id="logResultAdvice" class="com.lfool.interceptor.LogResultAdvice" />

    <!-- 定义一个只拦截 createUser 方法的 advisor -->
    <bean id="logCreateAdvisor" class="org.springframework.aop.support.NameMatchMethodPointcutAdvisor">
        <!-- advisor 实例的内部会有一个 advice -->
        <property name="advice" ref="logArgsAdvice"/>
        <!-- 只有下面两个方法才会被拦截 -->
        <property name="mappedNames" value="createUser, createOrder" />
    </bean>

    <bean id="useServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <!-- 代理的接口 -->
        <property name="proxyInterfaces">
            <list>
                <value>com.lfool.service.UserService</value>
            </list>
        </property>
        <!-- 代理的具体实现 -->
        <property name="target" ref="userServiceImpl" />
        <!-- 配置拦截器，这里可以配置 advice、advisor、interceptor -->
        <property name="interceptorNames">
            <list>
                <value>logCreateAdvisor</value>
            </list>
        </property>
    </bean>

    <!-- =========================================== -->
    <!-- 同理，我们也可以配置一个 orderServiceProxy...  -->
    <!-- =========================================== -->

</beans>
```

然后给出测试类：

```java
public class Main {
    public static void main(String[] args) {
        // 启动 Spring IoC 容器
        ApplicationContext context = new ClassPathXmlApplicationContext("spring_1.2_advisor.xml");
        UserService userService = (UserService) context.getBean("useServiceProxy");

        userService.createUser("Tom", "Cruise", 55);
        userService.queryUser();
    }
}
```

输出结果如下：

```
准备执行方法：createUser, 参数列表：[Tom, Cruise, 55]
```

从结果中可以看出，它只拦截了`createUser()`方法，且只进行了方法执行前的处理，因为我们在 XML 配置文件中只配置了`logArgsAdvice`

#### <font color=#9933FF>Interceptor 实现</font>

先给出 XML 配置文件`spring_1.2_interceptor.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userServiceImpl" class="com.lfool.service.impl.UserServiceImpl" />
    <bean id="orderServiceImpl" class="com.lfool.service.impl.OrderServiceImpl" />

    <!-- 定义两个 advice -->
    <bean id="logArgsAdvice" class="com.lfool.interceptor.LogArgsAdvice" />
    <bean id="logResultAdvice" class="com.lfool.interceptor.LogResultAdvice" />

    <!-- 定义一个 interceptor -->
    <bean id="logCreateInterceptor" class="com.lfool.aop_interceptor.DebugInterceptor" />

    <bean id="useServiceProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <!-- 代理的接口 -->
        <property name="proxyInterfaces">
            <list>
                <value>com.lfool.service.UserService</value>
            </list>
        </property>
        <!-- 代理的具体实现 -->
        <property name="target" ref="userServiceImpl" />
        <!-- 配置拦截器，这里可以配置 advice、advisor、interceptor -->
        <property name="interceptorNames">
            <list>
                <value>logCreateInterceptor</value>
            </list>
        </property>
    </bean>

    <!-- =========================================== -->
    <!-- 同理，我们也可以配置一个 orderServiceProxy...  -->
    <!-- =========================================== -->

</beans>
```

然后实现`MethodInterceptor`接口：

```java
public class DebugInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("Before: invocation=[" + invocation + "]");
        // 执行 真实实现类 的方法
        Object rval = invocation.proceed();
        System.out.println("Invocation returned");
        return rval;
    }
}
```

最后给出测试类：

```java
public class Main {
    public static void main(String[] args) {
        // 启动 Spring IoC 容器
        ApplicationContext context = new ClassPathXmlApplicationContext("spring_1.2_interceptor.xml");
        UserService userService = (UserService) context.getBean("useServiceProxy");

        userService.createUser("Tom", "Cruise", 55);
        userService.queryUser();
    }
}
```

输出结果如下：

```
Before: invocation=[ReflectiveMethodInvocation: public abstract com.lfool.entity.User com.lfool.service.UserService.createUser(java.lang.String,java.lang.String,int); target is of class [com.lfool.service.impl.UserServiceImpl]]
Invocation returned
Before: invocation=[ReflectiveMethodInvocation: public abstract com.lfool.entity.User com.lfool.service.UserService.queryUser(); target is of class [com.lfool.service.impl.UserServiceImpl]]
Invocation returned
```

可以看到当我们调用目标对象的方法时，其实调用的是`invoke()`方法

#### <font color=#9933FF>BeanNameAutoProxyCreator 实现</font>

上面三种方式都是基于 ProxyFactoryBean 实现，需要在 XML 配置文件中为每个目标类设置一个`ProxyFactoryBean`，十分的不灵活；而 BeanNameAutoProxyCreator 实现就对这一局限进行了改进

先给出 XML 配置文件`spring_1.2_BeanNameAutoProxy.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userServiceImpl" class="com.lfool.service.impl.UserServiceImpl" />
    <bean id="orderServiceImpl" class="com.lfool.service.impl.OrderServiceImpl" />

    <!-- 定义两个 advice -->
    <bean id="logArgsAdvice" class="com.lfool.interceptor.LogArgsAdvice" />
    <bean id="logResultAdvice" class="com.lfool.interceptor.LogResultAdvice" />

    <bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
        <!-- 配置拦截器，这里可以配置 advice、advisor、interceptor -->
        <property name="interceptorNames">
            <list>
                <value>logArgsAdvice</value>
                <value>logResultAdvice</value>
            </list>
        </property>
        <!-- 会为 *ServiceImpl 自动创建代理类 -->
        <property name="beanNames" value="*ServiceImpl" />
    </bean>

</beans>
```

然后给出测试类：

```java
public class Main {
    public static void main(String[] args) {
        // 启动 Spring IoC 容器
        ApplicationContext context = new ClassPathXmlApplicationContext("spring_1.2_BeanNameAutoProxy.xml");

        // 这里不再需要根据 BeanName 获取 Bean
        // userService 和 orderService 是一个披着接口外套的 JdkDynamicAopProxy 代理对象
        UserService userService = context.getBean(UserService.class);
        OrderService orderService = context.getBean(OrderService.class);

        userService.createUser("Tom", "Cruise", 55);
        userService.queryUser();

        orderService.createOrder("Leo", "随便买点什么");
        orderService.queryOrder("Leo");
    }
}
```

输出结果如下：

```
准备执行方法：createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User(firstName=Tom, lastName=Cruise, age=55, address=null)
准备执行方法：queryUser, 参数列表：[]
方法返回：User(firstName=Tom, lastName=Cruise, age=55, address=null)
准备执行方法：createOrder, 参数列表：[Leo, 随便买点什么]
方法返回：Order(username=Leo, product=随便买点什么)
准备执行方法：queryOrder, 参数列表：[Leo]
方法返回：Order(username=Leo, product=随便买点什么)
```

从结果中可以看出，BeanNameAutoProxyCreator 实现和 Advice 实现很像，拦截粒度都是类级别，不同之处在于它会自动生成符合正则表达式规则的所有代理类

#### <font color=#9933FF>DefaultAdvisorAutoProxyCreator 实现</font>

DefaultAdvisorAutoProxyCreator 实现可以匹配到方法，也就是可以只拦截指定方法，而非目标类的所有方法

先给出 XML 配置文件`spring_1.2_DefaultAdvisorAutoProxy.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userServiceImpl" class="com.lfool.service.impl.UserServiceImpl" />
    <bean id="orderServiceImpl" class="com.lfool.service.impl.OrderServiceImpl" />

    <!-- 定义两个 advice -->
    <bean id="logArgsAdvice" class="com.lfool.interceptor.LogArgsAdvice" />
    <bean id="logResultAdvice" class="com.lfool.interceptor.LogResultAdvice" />

    <!-- 记录 create* 方法的参数 -->
    <bean id="logArgsAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="advice" ref="logArgsAdvice" />
        <property name="pattern" value="com.lfool.service.*.create.*" />
    </bean>
    <!-- 记录 create* 方法的返回值 -->
    <bean id="logResultAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="advice" ref="logResultAdvice" />
        <property name="pattern" value="com.lfool.service.*.create.*" />
    </bean>

    <!-- 定义 DefaultAdvisorAutoProxyCreator -->
    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />

</beans>
```

然后给出测试类：

```java
public class Main {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("spring_1.2_DefaultAdvisorAutoProxy.xml");
        UserService userService = context.getBean(UserService.class);
        OrderService orderService = context.getBean(OrderService.class);

        userService.createUser("Tom", "Cruise", 55);
        userService.queryUser();

        orderService.createOrder("Leo", "随便买点什么");
        orderService.queryOrder("Leo");
    }
}
```

输出结果如下：

```
准备执行方法：createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User(firstName=Tom, lastName=Cruise, age=55, address=null)
准备执行方法：createOrder, 参数列表：[Leo, 随便买点什么]
方法返回：Order(username=Leo, product=随便买点什么)
```

从结果中可以看出，只拦截了`create*()`方法，并没有拦截`query*()`方法

### <font color=#1FA774>Spring AOP 源码剖析</font>

下面主要分析 BeanNameAutoProxyCreator 实现的源码，关注点在于 Spring 如何为目标对象自动创建代理对象～～

在 XML 配置文件中会配置`BeanNameAutoProxyCreator`，它继承`AbstractAutoProxyCreator`类

`AbstractAutoProxyCreator`类实现了`BeanPostProcessor`接口的`postProcessBeforeInitialization`方法和`postProcessAfterInitialization`方法

在 Spring IoC 容器启动自动注入对象时，会在初始化阶段前执行`postProcessBeforeInitialization`方法，初始化后执行`postProcessAfterInitialization`方法。**关于 Spring IoC 相关内容可见 [Spring IoC](./Spring-IoC.html)**

为目标对象自动创建代理对象就在`postProcessAfterInitialization`方法中完成，这也是 AOP 基于后处理器实现的原因

先来看看`AbstractAutoProxyCreator`类：

```java
// AbstractAutoProxyCreator.java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            // 创建代理对象，具体见下方
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // 检查参数，跳过已经执行过代理对象生成，或者已知的不需要生成代理对象的 Bean
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // Create proxy if we have advice.
    // 查看当前 Bean 所有 AOP 增强设置，最终通过 AOPUtils 工具类实现
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 创建代理对象，具体见下方
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }
    // 实例化代理工厂类
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);
    
    // 当全局使用动态代理时，设置是否需要对目标 Bean 强制使用 CGLIB 动态代理
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }
    // 构建 AOP Advisor
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    // 设置 proxyFactory 属性
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
    
    // 创建代理对象，具体见下方
    return proxyFactory.getProxy(getProxyClassLoader());
}
```

继续看`ProxyFactory`类：

```java
// ProxyFactory.java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
// ProxyCreatorSupport.java，是 ProxyFactory 的父类
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    // 创建代理对象，具体见下方
    return getAopProxyFactory().createAopProxy(this);
}
```

最后看`DefaultAopProxyFactory`类：

```java
// DefaultAopProxyFactory.java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    // 如果 optimize 或 proxyTargetClass 属性设置为 true，或者未指定代理接口，则使用 CGLIB 动态代理
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        // 目标类本身是接口或者代理对象，仍然使用 JDK 动态代理
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            // 使用 JDK 动态代理
            return new JdkDynamicAopProxy(config);
        }
        // 使用 CGLIB 动态代理
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        // 使用 JDK 动态代理
        return new JdkDynamicAopProxy(config);
    }
}
```

**<font color='red'>总结：</font>**Spring AOP 的实现基于 **[动态代理](../java/代理模式-静态-动态.html#动态代理)**。如果目标类实现了接口，Spring 会使用 JDK 动态代理；如果目标类没有实现接口，Spring 会使用 CGLIB 动态代理

这里给出个人的一个猜想，不知对错，有懂的可以在首页联系作者微信！！

在写 Spring 项目时，为什么建议在 Service 层先定义接口，然后再写对应实现类？？

我认为有一个原因可能是，目标类实现了接口就可以使用 JDK 动态代理，而 JDK 动态代理效率相比于 CGLIB 动态代理更加优秀，而且随着 JDK 版本的升级，这个优势更加明显

### <font color=#1FA774>参考文章</font>

- **[Spring AOP 使用介绍，从前世到今生](https://javadoop.com/post/spring-aop-intro)**
- **[Spring AOP 源码解析](https://javadoop.com/post/spring-aop-source)**
