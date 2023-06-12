# Spring IoC

本篇文章主要介绍 Spring IoC 以及 Spring Bean 相关的内容～～但其实本篇文章更侧重与之相关的八股问题，哈哈哈哈 (功利心极强 😊)

**<font color='red'>更新：</font>**总觉得 IoC 是 Spring 的精髓所在，同时有一身不想只是速成的傲骨 (古代文人附身～)，所以本着有多底层就多底层的原则 (其实也没有到祖坟的程度)，将核心源码过了一遍，附在文章末尾

### <font color=#1FA774>什么是 IoC</font>

IoC (Inversion Of control，控制反转) 是一种设计思想，并非一个具体技术的实现，将原本在程序中手动创建对象的控制权，交由 Spring 框架来管理

正常情况下在程序中都是通过`new`关键字来手动创建一个对象，而使用 Spring 后不再需要自己去`new`一个对象，而是直接在 IoC 容器中取出所需对象即可

IoC 容器就像一个工厂，实际上它是一个 Map 集合，每当我们需要创建对象时，只需要设置好配置文件或注解即可，完全不需要考虑对象是如何被创建出来的

比如，实际项目中一个 Service 类可能依赖很多其它的类，如果手动实例化 Service 对象需要考虑所有底层依赖类的构造函数，如果使用 IoC 的话，只需要配置好即可

在 Spring 中一般通过 XML 文件来配置 Bean，但后来感觉这样方法不太好，于是在 SpringBoot 中注解配置就慢慢开始流行～

下图是加入了 Ioc 容器的创建对象方式：

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230611/1734281686476068AQrHxB1.svg)

```java
// 没有 IoC 容器
A a = new A();

// 加入 IoC 容器
@Autowired
A a;  // 自动注入
```

### <font color=#1FA774>什么是 Spring Bean</font>

简单来说 Spring Bean 就是那些被 IoC 容器所管理的对象

我们需要告诉 IoC 容器需要管理哪些对象，这个是通过配置元数据来定义。配置元数据可以是 XML 文件、注解、Java 配置类

### <font color=#1FA774>创建 Spring Bean 的方法</font>

**使用构造函数创建 Bean**。在配置类中使用`@Bean`注解，并提供相应的构造函数，如下所示：

```java
@Configuration
public class MyConfiguration {
    @Bean
    public MyBean myBean() {
        return new MyBean();
    }
}
```

**使用静态工厂创建 Bean**。在配置类中使用`@Bean`注解，并提供一个工厂方法，如下所示：

```java
@Configuration
public class MyConfiguration {
    @Bean
    public MyBean myBean() {
        return MyBeanFactory.createMyBean();
    }
}
// 静态工厂
public class MyBeanFactory {
    public static MyBean createMyBean() {
        return new MyBean();
    }
}
```

**使用实例工厂创建 Bean**。如果 Bean 的创建依赖于其它 Bean 实例，可以使用实例工厂方式创建 Bean，如下所示：

```java
@Configuration
public class MyConfiguration {
    @Bean
    public MyBeanFactory myBeanFactory() {
        return new MyBeanFactory();
    }
    @Bean
    public MyBean myBean() {
        return myBeanFactory().createMyBean();
    }
}
// 实例工厂
public class MyBeanFactory {
    public MyBean createMyBean() {
        return new MyBean();
    }
}
```

使用注解方式创建 Bean。可以使用`@Component`注解或其衍生注解，如：`@Controller`、`@Service`、`@Repository`，让 Spring 自动扫描并创建 Bean 实例

- `@Component`：通用的注解，可标记任意类为 Spring 组件。如果一个 Bean 不知道属于哪个层，可以使用该注解
- `@Controller`：对应 Controller 层，主要接收用户请求并调用 Service 层返回数据给前端页面
- `@Service`：对应 Service 层，主要设计一些复杂的逻辑，需要用到 Dao 层
- `@Repository`：对应持久层，即 Dao 层，主要用于数据库相关的操作

**<font color='red'>注意：</font>**这些注解作用完全一样，都是创建一个 Bean，只不过可以根据注解快速判断所属层次

```java
@Component
public class MyBean {
    private String name = "LFool";
}
```

### <font color=#1FA774>@Component 和 @Bean 的区别</font>

- `@Component`注解作用于类；`@Bean`注解作用于方法
- `@Component`通过类路径扫描自动装配到 Spring 容器中；`@Bean`通常是在标有该注解的方法中返回 Bean 对象
- `@Bean`注解比`@Component`更灵活，有些时候只能通过`@Bean`来注册 Bean 对象，如：引入第三方库中的类要装配到 Spring 容器时

### <font color=#1FA774>注入 Bean 的注解</font>

有三种注解可以注入 Bean，它们分别是：`@Autowired`、`@Resource`、`@Inject`。其中，`@Autowired`和`@Resource`使用比较多一些

### <font color=#1FA774>@Autowired 和 @Resource 的区别</font>

- `@Autowired`是 Spring 提供的注解；`@Resource`是 JDK 提供的注解
- `@Autowired`默认注入方式为 byType (根据类型匹配)，如果无法匹配再使用 byName (根据名称匹配)；`@Resource`默认注入方式为 byName，如果无法匹配再使用 byType
- 当一个接口有多个实现类时，`@Autowired`和`@Resource`都需要通过名称才能正确匹配到对应的 Bean

### <font color=#1FA774>Bean 的作用域</font>

在定义 Bean 时，用户不但可以配置 Bean 的属性值及相互之间的依赖关系，还可以定义 Bean 的作用域，作用域会对 Bean 的生命周期和创建方式产生影响

配置 Bean 的作用域有两种方式：

```java
// XML 方式
<bean id="..." class="..." scope="singleton"></bean>

// 注解方式
@Bean
@Scope("singleton")
public Person personPrototype() {
 return new Person();
}
```

Spring 中 Bean 的作用域通常有下面几种：

- **singleton：**IoC 容器中只有唯一的 Bean 实例。Spring 中的 Bean 默认都是单例的，是对单例设计模式的应用
- **prototype：**每次获取都会创建一个新的 Bean 实例，也就是连续两次从 IoC 容器中获取同一个类的 Bean 实例，会得到两个不同的 Bean 实例
- **request：**每一次 HTTP 请求都会产生一个新的 Bean 实例，该 Bean 实例仅在当前 HTTP 请求内有效 (仅适用 Web 应用)
- **session：**每一次来自新 session 的 HTTP 请求都会产生一个新的 Bean 实例，该 Bean 实例仅在当前 HTTP session 内有效 (仅适用 Web 应用)
- **application/global-session：**每个 Web 应用在启动时会创建一个 Bean 实例，该 Bean 实例仅在当前应用启动时间内有效 (仅适用 Web 应用)
- **websocket：**每一次 WebSocket 会话中会创建一个 Bean 实例 (仅适用 Web 应用)

### <font color=#1FA774>Bean 的生命周期</font>

对于 Spring Bean 的生命周期来说，主要分为五个阶段：**实例化**、**属性赋值**、**初始化**、**使用**、**销毁**，但在这四个步骤中间会穿插一些小的步骤，具体过程如下：

- 执行`getBean()`后，Bean 容器在配置文件中找到对应 Bean 的定义。在容器初始化后，会将配置文件中每个 Bean 生成一个对应的 BeanDefinition 对象，但这并非 Bean 实例
- **实例化：**Bean 容器使用 Java 反射机制创建 Bean 的实例，调用构造函数
- **属性赋值：**为 Bean 实例的属性设置值，如果属性本身就是 Bean，则将对其进行解析和创建
- 如果 Bean 实现了`BeanNameAware`接口，会调用`setBeanName()`方法，并将配置文件中设置的 Bean 名称作为参数传入该方法中，那么在 Bean 实例中就可以获取配置文件中为该实例设置的名称
- 如果 Bean 实现了`BeanClassLoaderAware`接口，会调用`setBeanClassLoader(ClassLoader)`方法，那么在 Bean 实例中就可以获取加载本类的类加载器
- 如果 Bean 实现了`BeanFactoryAware`接口，会调用`setBeanFactory(BeanFactory)`方法，那么在 Bean 实例中就可以使用 BeanFactory 获取其它 Bean 实例
- 如果 BeanFactory 装载了`BeanPostProcessor`处理器，会调用`postProcessBeforeInitialization(bean, beanName)`方法，可以对 Bean 实例进行加工操作，如：改变 Bean 的行为
- 如果 Bean 实现了`InitializingBean`接口，会调用`afterPropertiesSet()`方法
- **初始化：**如果`<bean>`中定义了 init-method 方法，则执行该方法
- 如果 BeanFactory 装载了`BeanPostProcessor`处理器，会调用`postProcessAfterInitialization(bean, beanName)`方法，可以对 Bean 实例再次进行加工操作
- **使用：**应用程序使用 Bean 实例
- **销毁：**如果 Bean 实现了`DisposableBean`接口，会调用`destroy()`方法销毁 Bean 实例
- 如果`<bean>`中定义了 destroy-method 方法，则执行该方法

下面给出 Bean 生命周期的简图，方便记忆：

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230612/0447141686516434ZGuYAn3.svg)

下面给出 Bean 生命周期略详细的过程图，方便理解：



![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230612/0447041686516424eMKypi2.svg)

**<font color='red'>强调：</font>**在初始化前后都有调用`BeanPostProcessor`接口的方法：前置方法`postProcessBeforeInitialization()`和后置方法`postProcessAfterInitialization()`，这两个方法的作用就是可以对 Bean 实例进行再次加工处理，如：改变 Bean 的行为。另外，Spring 容器提供的各种神奇功能，如：AOP、动态代理等，都是通过该接口实现

#### <font color=#9933FF>Demo</font>

下面给出一个小 Demo 实打实的看看 Bean 的生命周期～

项目结构如下：

```xml
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── lfool
        │           └── springbootlearn
        │               ├── test
        │               │   └── Test.java
        │               └── entity
        │                   ├── MyBean.java
        │                   └── MyBeanPostProcessor.java
        └── resources
            └── spring-config.xml
```

配置文件`spring-config.xml`内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!-- 向 Spring 容器内注册 Bean 实例 -->
    <bean name="myBeanPostProcessor" class="com.lfool.springbootlearn.entity.MyBeanPostProcessor" />
    <bean name="myBean" class="com.lfool.springbootlearn.entity.MyBean"
          init-method="init" destroy-method="destroyMethod">
        <property name="name" value="LFool" />
    </bean>
</beans>
```

`MyBean.java`内容如下：

```java
package com.lfool.springbootlearn.entity;

import lombok.Getter;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.boot.context.properties.ConfigurationProperties;

@Getter
public class MyBean implements InitializingBean, BeanNameAware, BeanClassLoaderAware, BeanFactoryAware, DisposableBean {

    private String name;

    public MyBean() {
        System.out.println("1. 调用无参构造函数");
    }

    public MyBean(String name) {
        this.name = name;
    }

    public void setName(String name) {
        this.name = name;
        System.out.println("2. 设置属性");
    }

    @Override
    public void setBeanName(String s) {
        System.out.println("3. 调用 BeanNameAware.setBeanName() 方法");
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        System.out.println("4. 调用 BeanClassLoaderAware.setBeanClassLoader() 方法");
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("5. 调用 BeanFactoryAware.setBeanFactory() 方法");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("7. 调用 InitializingBean.afterPropertiesSet() 方法");
    }

    public void init() {
        System.out.println("8. 执行自定义 init() 方法");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("10. 调用 DisposableBean.destroy() 方法");
    }

    public void destroyMethod() {
        System.out.println("11. 执行自定义 destroyMethod() 方法");
    }

    public void work() {
        System.out.println("使用中...");
    }
}
```

`MyBeanPostProcessor.java`内容如下：

```java
package com.lfool.springbootlearn.entity;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;

public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("6. 调用 BeanPostProcessor.postProcessBeforeInitialization() 方法");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("9. 调用 BeanPostProcessor.postProcessAfterInitialization() 方法");
        return bean;
    }
}
```

`Test.java`内容如下：

```java
package com.lfool.springbootlearn.controller;

import com.lfool.springbootlearn.entity.MyBean;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Test {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring-config.xml");
        MyBean myBean = (MyBean) context.getBean("myBean");
        myBean.work();
        context.destroy();
    }
}
```

输入如下：

```java
1. 调用无参构造函数
2. 设置属性
3. 调用 BeanNameAware.setBeanName() 方法
4. 调用 BeanClassLoaderAware.setBeanClassLoader() 方法
5. 调用 BeanFactoryAware.setBeanFactory() 方法
6. 调用 BeanPostProcessor.postProcessBeforeInitialization() 方法
7. 调用 InitializingBean.afterPropertiesSet() 方法
8. 执行自定义 init() 方法
9. 调用 BeanPostProcessor.postProcessAfterInitialization() 方法
使用中...
10. 调用 DisposableBean.destroy() 方法
11. 执行自定义 destroyMethod() 方法
```

### <font color=#1FA774>Spring IoC 源码剖析</font>

前文说过 Spring IoC 就是一个 Map 集合，存储所有 Bean 实例，在配置好的前提下，无须手动创建 Bean 实例，直接去 IoC 容器中取即可

配置 Bean 可以通过 XML 文件、注解、Java 配置类，这里采用最原始的配置方法：XML 文件。如果在配置 Bean 时没有指定懒加载，那么会在 Spring IoC 容器初始化时创建所有的 Bean 实例；如果指定了懒加载，那么会在调用`getBean(name)`方法时创建 Bean 实例。指定懒加载方法如下：

```xml
<bean name="myBean" class="com.lfool.ioc.entity.MyBean" lazy-init="true"></bean>
```

本部分主要从两个方面展开讨论相关源码：**Spring IoC 容器的初始化**、**Bean 的生命周期**

**<font color='red'>注意：</font>**本部分介绍的源码基于 spring-context 4.3.11.RELEASE

#### <font color=#9933FF>Spring IoC 容器的初始化</font>

首先会在程序中创建一个 spring IoC 容器，这里使用 XML 的方式：

```java
ApplicationContext context = new ClassPathXmlApplicationContext("application.xml");
```

上面创建 spring IoC 容器会调用下面的构造函数：

```java
// ClassPathXmlApplicationContext.java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    this(new String[] {configLocation}, true, null);  // 见下方
}
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) throws BeansException {
    super(parent);
    setConfigLocations(configLocations);  // 根据提供的路径，处理成配置文件数组 (以分号、逗号、空格、换行符分割)
    if (refresh) {
        refresh();  // 核心方法，见下方
    }
}
```

`refresh()`方法不仅仅可以用来第一次初始化，也可以用来重建 ApplicationContext，会将原来的 ApplicationContext 销毁，然后重新执行一次初始化，详细见下方：

```java
// AbstractApplicationContext.java
public void refresh() throws BeansException, IllegalStateException {
    // 加锁，每次只能有一个线程执行 refresh() 方法
    synchronized (this.startupShutdownMonitor) {
        // 准备工作：记录容器启动时间、标记「已启动」状态、处理配置文件中的占位符
        prepareRefresh();
        // 创建 BeanFactory，并将配置文件中的 Bean 解析提取并生成一个 BeanDefinition 对象，保存在 BeanFactory 中 (实际是一个 Map 集合)
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // 设置 BeanFactory 类加载器、添加几个 BeanPostProcessor、手动注册几个特殊的 Bean
        prepareBeanFactory(beanFactory);

        try {
            // 提供给子类的扩展点。到这里所有的 Bean 都加载、注册，但还没有初始化，只是将 Bean 对应的 BeanDefinition 加入 BeanFactory 中
            // 如果 Bean 实现了 BeanFactoryPostProcessor 接口，会在这里配置
            postProcessBeanFactory(beanFactory);
            // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册 BeanPostProcessor 实现类，有两个方法：postProcessBeforeInitialization 和 postProcessAfterInitialization，会在 Bean 初始化前后执行
            // BeanPostProcessor 和 BeanFactoryPostProcessor 的区别在于：BeanPostProcessor 只会注册并不会执行方法，BeanFactoryPostProcessor 注册并执行方法
            registerBeanPostProcessors(beanFactory);

            // 初始化当前 ApplicationContext 的 MessageSource
            initMessageSource();

            // 初始化当前 ApplicationContext 的事件广播器
            initApplicationEventMulticaster();

            // 钩子方法，可以在具体的子类中初始化一些特殊的 Bean
            onRefresh();

            // 注册事件监听器
            registerListeners();

            // 初始化所有 Bean，包括：实例化、属性赋值、初始化，但如果 Bean 指定为懒加载就不会初始化
            finishBeanFactoryInitialization(beanFactory);

            // 广播事件，ApplicationContext 初始化完成
            finishRefresh();
        } catch (BeansException ex) {
            // 省略...主要会销毁创建的 Bean，并抛出异常
        } finally {
            // 省略...
        }
    }
}
```

对于`refresh()`调用的方法，这里只挑几个重要的分析，第一个就是`obtainFreshBeanFactory()`方法：

```java
// AbstractApplicationContext.java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 如果当前 ApplicationContext 存在 BeanFactory，那么先关闭它，然后创建一个新的 BeanFactory，具体见下方
    refreshBeanFactory();
    // 返回刚刚创建的 BeanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
// AbstractRefreshableApplicationContext.java
protected final void refreshBeanFactory() throws BeansException {
    // 如果当前 ApplicationContext 已经存在 BeanFactory，先关闭它
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 创建一个 DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 设置序列化 ID
        beanFactory.setSerializationId(getId());
        // 这是 Bean 是否允许覆盖、是否允许循环引用
        customizeBeanFactory(beanFactory);
        // 解析配置文件，为每个 Bean 生成对应的 BeanDefinition 对象并加入 beanFactory 中
        // 这里就不展开讨论该方法的细节，主要就是解析 XML 文件
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    } catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

第二个要详细介绍就是`finishBeanFactoryInitialization()`方法：

```java
// AbstractApplicationContext.java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Initialize conversion service for this context.
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
            @Override
            public String resolveStringValue(String strVal) {
                return getEnvironment().resolvePlaceholders(strVal);
            }
        });
    }

    // 先初始化 LoadTimeWeaverAware 类型的 Bean
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // 确保配置文件中所有 Bean 都已经被解析、加载、注册
    beanFactory.freezeConfiguration();

    // 开始初始化 (重点介绍)，见下方
    beanFactory.preInstantiateSingletons();
}
// DefaultListableBeanFactory.java
public void preInstantiateSingletons() throws BeansException {
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Pre-instantiating singletons in " + this);
    }
    
    // 存储 Bean 的名称
    List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

    // 遍历所有 Bean
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // !bd.isAbstract() 表示非抽象；bd.isSingleton() 表示单例；!bd.isLazyInit() 表示非懒加载
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                        @Override
                        public Boolean run() {
                            return ((SmartFactoryBean<?>) factory).isEagerInit();
                        }
                    }, getAccessControlContext());
                }
                else {
                    isEagerInit = (factory instanceof SmartFactoryBean &&
                            ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                if (isEagerInit) {
                    getBean(beanName);
                }
            }
            else {
                // 初始化普通 Bean，见下方
                getBean(beanName);
            }
        }
    }
    
    // 到这里表示所有 Bean 已经初始化完成，如果定义的 Bean 实现了 SmartInitializingSingleton 接口，需要在这里回调 (不做重点介绍)
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
// AbstractBeanFactory.java
public Object getBean(String name) throws BeansException {
    // 该方法负责执行构造函数、属性赋值、初始化 (Bean 生命周期的前三阶段)
    // 从这里开始和调用 getBean() 获取 Bean 重复，具体见「Bean 的生命周期」源码介绍
    return doGetBean(name, null, null, false);
}
```

#### <font color=#9933FF>Bean 的生命周期</font>

当 Bean 指定为懒加载，那么会在第一次调用`getBean()`时初始化；当 Bean 指定为非懒加载，那么会在 Spring IoC 容器初始化时完成 Bean 的初始化。下面假设 Bean 被指定为懒加载模式！！

首先调用`getBean()`方法的细节如下：

```java
// AbstractApplicationContext.java
public Object getBean(String name) throws BeansException {
    assertBeanFactoryActive();
    return getBeanFactory().getBean(name);
}
// AbstractBeanFactory.java
public Object getBean(String name) throws BeansException {
    // 从这里开始，和非懒加载下的 Spring IoC 容器初始化过程重合
    return doGetBean(name, null, null, false);
}
```

`doGetBean()`方法会根据 Bean 的作用域不同采用不同的创建策略，具体如下：

```java
// AbstractBeanFactory.java
protected <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {

    final String beanName = transformedBeanName(name);
    Object bean;

    // 尝试先从 BeanFactory 中获取
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 从 BeanFactory 中获取成功
        if (logger.isDebugEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // 从 BeanFactory 中获取失败
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // Check if bean definition exists in this factory.
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }

        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    registerDependentBean(dep, beanName);
                    getBean(dep);
                }
            }

            // Bean 的作用域为 singleton
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        try {
                            // 创建 Bean，见下方
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
            // Bean 的作用域为 prototype
            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                        @Override
                        public Object getObject() throws BeansException {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
        try {
            return getTypeConverter().convertIfNecessary(bean, requiredType);
        }
        catch (TypeMismatchException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

下面来看看`createBean()`方法：

```java
// AbstractAutowireCapableBeanFactory.java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }
    // doCreateBean 真正初始化 Bean 的方法，见下方
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isDebugEnabled()) {
        logger.debug("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
}
```

`doCreateBean()`方法才真正开始初始化 Bean，如下：

```java
// AbstractAutowireCapableBeanFactory.java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 执行 Bean 的构造函数，对应 Bean 生命周期的第 1 步
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 属性赋值，对应 Bean 生命周期的第 2 步
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            // 对应 Bean 生命周期的初始化阶段，见下方
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                            "Bean with name '" + beanName + "' has been injected into other beans [" +
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                            "] in its raw version as part of a circular reference, but has eventually been " +
                            "wrapped. This means that said other beans do not use the final version of the " +
                            "bean. This is often the result of over-eager type matching - consider using " +
                            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

`initializeBean()`对应 Bean 生命周期的初始化阶段，如下：

```java
// AbstractAutowireCapableBeanFactory.java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                invokeAwareMethods(beanName, bean);
                return null;
            }
        }, getAccessControlContext());
    }
    else {
        // 调用 Bean 实现 *.Aware 接口的方法，对应 Bean 生命周期的第 3、4、5 步
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 调用 Bean 实现 BeanPostProcessor 接口的前置处理方法，对应 Bean 生命周期的第 6 步
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 调用 Bean 实现 InitializingBean 接口的方法，对应 Bean 生命周期的第 7 步
        // 调用 Bean 自定义的 init 方法，对应 Bean 生命周期的第 8 步
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }

    if (mbd == null || !mbd.isSynthetic()) {
        // 调用 Bean 实现 BeanPostProcessor 接口的后置处理方法，对应 Bean 生命周期的第 9 步
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

至此，Bean 生命周期对应的源码已经介绍到第 9 步 (和 **[Demo](./Spring-IoC.html#demo)** 部分中输出的步骤一致)，后续只剩下使用和销毁，这里就不过多赘述

### <font color=#1FA774>参考文章</font>

- **[Spring Bean生命周期，好像人的一生](https://juejin.cn/post/7075168883744718856)**
- **[Spring IOC 容器源码分析](https://javadoop.com/post/spring-ioc)**