# Spring IoC

本篇文章主要介绍 Spring IoC 以及 Spring Bean 相关的内容～～但其实本篇文章更侧重与之相关的八股问题，哈哈哈哈 (功利心极强 😊)

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

- 执行`getBean()`后，Bean 容器在配置文件中找到对应 Bean 的定义
- **实例化：**Bean 容器使用 Java 反射机制创建 Bean 的实例，调用无参构造函数
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

### <font color=#1FA774>参考文章</font>

- **[Spring Bean生命周期，好像人的一生](https://juejin.cn/post/7075168883744718856)**