# Spring 事务

在 **[MySQL 事务](../mysql/MySQL事务.html)** 中介绍过「什么是事务」「事务的特性 (ACID)」「事务的隔离级别」，Spring 事务在这些方面其实是差不多滴，所以就不过多介绍这些内容

### <font color=#1FA774>Spring 对事务的支持方式</font>

Spring 支持两种方式的事务管理，分别为：编程式事务管理、声明式事务管理

#### <font color=#9933FF>编程式事务管理</font>

顾名思义，编程式事务管理需要通过具体的类编写部分代码来手动管理事务，比如使用`TransactionTemplate`或者`TransactionManager`

如果使用 Spring 的 XML 配置来管理 Bean，那么首先需要在文件中添加对应的 Bean：

```xml
<!-- TransactionTemplate 依赖于 TransactionManager -->
<bean id="transactionTemplate" class="org.springframework.transaction.support.TransactionTemplate">
    <property name="transactionManager" ref="transactionManager" />
</bean>
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <constructor-arg ref="myDataSource" />
</bean>
```

使用`TransactionTemplate`类进行编程式事务管理的示例如下：

```java
public void testTransactionTemplate() {
    // 获取 Spring IoC 容器
    ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
    // 根据 BeanName 获取对应的 Bean 实例
    TransactionTemplate transactionTemplate = context.getBean("transactionTemplate", TransactionTemplate.class);
    
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus status) {
            try {
                // 业务代码 ...
            } catch (Exception e) {
                e.printStackTrace();
                status.setRollbackOnly();  // 回滚
            }
        }
    });
}
```

使用`TransactionManager`类进行编程式事务管理的示例如下：

```java
public void testTransactionManager() {
    ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
    TransactionTemplate transactionTemplate = context.getBean("transactionTemplate", TransactionTemplate.class);
    PlatformTransactionManager transactionManager = transactionTemplate.getTransactionManager();
    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

    try {
        // 业务代码 ...
        transactionManager.commit(status);    // 提交事务
    } catch (Exception e) {
        e.printStackTrace();
        transactionManager.rollback(status);  // 回滚
    }
}
```

#### <font color=#9933FF>声明式事务管理</font>

除了上面不怎么方便的编程式事务管理，还有声明式事务管理，说白了就是基于注解，不需要写大量代码，一个注解就可以搞定，这个注解就是`@Transactional`

声明式事务管理是使用注解`@Transactional`，它底层基于 **[AOP](./Spring-AOP.html)** 实现，会为声明有`@Transactional`的类生成代理对象，在执行目标方法前后统一进行事务管理

在使用注解开启事务之前，需要在 Spring XML 配置文件中添加下面内容：

```xml
<tx:annotation-driven />
```

使用`TransactionManager`类进行编程式事务管理的示例如下：

```java
@Transactional(propagation = Propagation.REQUIRED)
public void testAnnotation() {
    // 业务代码 ...
}
```

### <font color=#1FA774>Spring 事务管理接口</font>

在 Spring 框架中，和事务管理相关的接口有 3 个：

- `PlatformTransactionManager`：事务管理器，Spring 事务策略的核心接口
- `TransactionDefinition`：事务定义信息，包括：事务隔离级别、传播行为、超时、只读、回滚规则
- `TransactionStatus`：事务运行的状态

我们可以把`PlatformTransactionManager`看作是事务的管理者，而`TransactionDefinition`和`TransactionStatus`可看作是对事务的描述

#### <font color=#9933FF>PlatformTransactionManager 事务管理接口</font>

Spring 并没有直接管理事务，而是提供了多种事务管理器，如：为 JDBC 提供了`DataSourceTransactionManager`；为 Hibernate 提供了`HibernateTransactionManager`；为 JPA 提供了`JpaTransactionManager`

Spring 提供的多种事务管理器都实现了`PlatformTransactionManager`接口，其中定义了三个方法：

```java
public interface PlatformTransactionManager {
    // 获取事务
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
    // 提交事务
	void commit(TransactionStatus status) throws TransactionException;
    // 回滚事务
	void rollback(TransactionStatus status) throws TransactionException;
}
```

#### <font color=#9933FF>TransactionDefinition 事务定义信息</font>

PlatformTransactionManager 通过`getTransaction()`获取一个事务，该方法的参数就是一个`TransactionDefinition`，定义了事务的一些基本属性，包括五个方面：

- 隔离级别 (同 MySQL)
- 传播行为
- 回滚规则
- 是否只读
- 事务超时

```java
public interface TransactionDefinition {
	int PROPAGATION_REQUIRED = 0;
	int PROPAGATION_SUPPORTS = 1;
	int PROPAGATION_MANDATORY = 2;
	int PROPAGATION_REQUIRES_NEW = 3;
	int PROPAGATION_NOT_SUPPORTED = 4;
	int PROPAGATION_NEVER = 5;
	int PROPAGATION_NESTED = 6;
	int ISOLATION_DEFAULT = -1;
	int ISOLATION_READ_UNCOMMITTED = Connection.TRANSACTION_READ_UNCOMMITTED;
	int ISOLATION_READ_COMMITTED = Connection.TRANSACTION_READ_COMMITTED;
	int ISOLATION_REPEATABLE_READ = Connection.TRANSACTION_REPEATABLE_READ;
	int ISOLATION_SERIALIZABLE = Connection.TRANSACTION_SERIALIZABLE;
	int TIMEOUT_DEFAULT = -1;
    
    // 返回事务的传播行为，默认值为 REQUIRE
	int getPropagationBehavior();
    // 返回事务的隔离级别，默认值为 DEFAULT
	int getIsolationLevel();
    // 返回事务的超时时间，默认值为 -1。如果超过该时间但事务还没有完成，则自动回滚事务
	int getTimeout();
    // 返回是否只读，默认为 false
	boolean isReadOnly();
	@Nullable
	String getName();
}
```

#### <font color=#9933FF>TransactionStatus 事务运行状态</font>

TransactionStatus 接口用来记录事务的状态，它定义了一组方法，用来获取或判断事物的状态信息

```java
public interface TransactionStatus extends SavepointManager, Flushable {
    // 是否是新事务
	boolean isNewTransaction();
    // 是否有恢复点
	boolean hasSavepoint();
    // 设置为只读
	void setRollbackOnly();
    // 是否为只回滚
	boolean isRollbackOnly();
    // 刷新
	@Override
	void flush();
    // 是否已完成
	boolean isCompleted();
}
```

### <font color=#1FA774>事务传播行为</font>

事务传播行为是指不同事务之间相互调用后事务的状态 (多个事务合并成一个事务？每个事务独立存在？)

举个简单的例子：如果分别为方法 A、B、C 开启了事务 TA、TB、TC，当方法 A 调用方法 B 和 C 后，那么三个事务将会何去何从？？

```java
// 开启事务 TA
void A() {
    B();
    C();
}
// 开启事务 TB
void B() {}
// 开启事务 TC
void C() {}
```

上面例子中，方法 A 可以称为外部方法，方法 A 调用了方法 B 和 C，那么方法 B 和 C 可以称为内部方法。当调用方法 B 和 C 时，当前已经有事务 TA

在 TransactionDefinition 定义了 7 种传播行为：

```java
public interface TransactionDefinition {
    // 如果当前没有事务，就新建一个事务；如果当前已经存在事务，就加入到该事务中
	int PROPAGATION_REQUIRED = 0;
    // 支持当前事务，如果当前没有事务，就以非事务的方式执行
	int PROPAGATION_SUPPORTS = 1;
    // 使用当前事务，如果当前没有事务，就抛出异常
	int PROPAGATION_MANDATORY = 2;
    // 新建事务，如果当前存在事务，把当前事务挂起
	int PROPAGATION_REQUIRES_NEW = 3;
    // 以非事务的方式执行，如果当前存在事务，就把当前事务挂起
	int PROPAGATION_NOT_SUPPORTED = 4;
    // 以非事务的方式执行，如果当前存在事务，就抛出异常
	int PROPAGATION_NEVER = 5;
    // 如果当前存在事务，就在嵌套事务中执行；如果当前不存在事务，就执行与 PROPAGATION_REQUIRED 类似的操作
	int PROPAGATION_NESTED = 6;
}
```

上面 7 种传播行为，下面只详细分析常用的 3 种：PROPAGATION_REQUIRED、PROPAGATION_REQUIRES_NEW、PROPAGATION_NESTED

#### <font color=#9933FF>PROPAGATION_REQUIRED</font>

PROPAGATION_REQUIRED 是平常使用最多的一个事务传播行为，如果使用`@Transactional`没有指定传播行为，那么默认的传播行为就是 PROPAGATION_REQUIRED

如果当前没有事务，就新建一个事务；如果当前已经存在事务，就加入到该事务中。更具体地：

- 如果外部方法没有开启事务，`PROPAGATION_REQUIRED`修饰的内部方法会为自己开启新事务，且多个内部方法开启的事务之间相互独立，互不干扰
- 如果外部方法开启了被`PROPAGATION_REQUIRED`修饰的事务，那么所有`PROPAGATION_REQUIRED`修饰的内部方法和外部方法都属于同一个事务，只要一个方法回滚，整个事务回滚

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230620/0450071687207807jsNqgC1.svg)

#### <font color=#9933FF>PROPAGATION_REQUIRES_NEW</font>

新建事务，如果当前存在事务，把当前事务挂起。更具体地：

- 如果外部方法没有开启事务，`PROPAGATION_REQUIRES_NEW`修饰的内部方法会为自己开启新事务，且多个内部方法开启的事务之间相互独立，互不干扰
- 如果外部方法开启了被`PROPAGATION_REQUIRED`修饰的事务，那么所有`PROPAGATION_REQUIRED`修饰的内部方法之间相互独立，同时内部方法也和外部方法相互独立，互不干扰

所以，无论外部方法有无开启事务，被`PROPAGATION_REQUIRED`修饰的内部方法都与外部方法相互独立，互不干扰

但是，如果内部方法抛出异常，且在外部方法中没有被 catch 捕获，那么外部方法就能感知到该异常，所以会回滚外部方法所在的事务

**<font color='red'>强调：</font>**如果方法 A 回滚，方法 B 和 C 不会受影响；如果方法 B 和 C 回滚，方法 A 也不会受影响 (主要想和 PROPAGATION_NESTED 区分)

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230620/0505191687208719Bd460E2.svg)

#### <font color=#9933FF>PROPAGATION_NESTED</font>

如果当前存在事务，就在嵌套事务中执行；如果当前不存在事务，就执行与 PROPAGATION_REQUIRED 类似的操作

继续延续上面的例子，如果方法 A 回滚，方法 B 和 C 也会跟着回滚；如果方法 B 和 C 回滚，方法 A 不会回滚

举个生动的小例子：父亲做啥孩子必须跟着做啥，但孩子做啥父亲不一定跟着做啥

### <font color=#1FA774>@Transactional 底层原理</font>

`@Transactional`基于 **[AOP](./Spring-AOP.html)** 实现，AOP 又基于 **[动态代理](../java/代理模式-静态-动态.html#动态代理)** 实现，会为目标对象动态生成一个代理对象，实际上执行的是代理对象中的方法，由代理对象调用目标方法，这样代理对象就可以在调用目标方法前后执行一些特殊处理，称之为功能增强。如果目标对象实现了接口，那么 Spring 会使用 JDK 动态代理，否则会使用 CGLIB 动态代理

更具体的，如果一个类或类中的`public`方法上使用了`@Transactional`，**[Spring IoC](./Spring-IoC.html)** 容器会在启动时为其创建一个代理对象，在调用被`@Transactional`修饰的方法时，实际调用的是代理对象的`invoke()`方法，该方法的作用：执行目标方法前开启事务，目标方法执行过程中抛出异常时回滚事务，目标方法顺利执行完后提交事务

**最后再啰嗦一句：**所以动态代理主要是增强目标方法执行前、后、抛出异常三个时刻！！

### <font color=#1FA774>Spring AOP 自调用问题</font>

上面介绍过，AOP 工作原理其实是执行代理对象中的方法，这样可以增强功能，实现事务管理

但如果同一个类中的方法自调用的话，那么执行的就是目标对象中的方法，也就无法走动态代理，进而无法让事务生效，如下面代码所示：

```java
@service
public class MyService {
    private void A() {
        B();
        // 业务代码 ...
    }
    @Transactional
    public void B() {
        // 业务代码 ...
    }
}
```

如果直接调用`A()`方法，那么就会使`B()`方法开启的事务失效，因为没有走代理

**解决方法：**使用 AspectJ 代理 Spring AOP 代理

### <font color=#1FA774>@Transactional 注意事项</font>

- `@Transactional`注解只有作用到 public 方法上才会生效，不推荐在接口上使用
- 避免同一个类中自调用`@Transactional`注解方法，会导致事务失效
- 被`@Transactional`注解的方法所在的类必须被 Spring IoC 容器管理，否则会导致事务失效 (🩸🩸🩸 血淋淋的教训！！！)
- 底层数据库必须支持事务，否则事务不生效，比如 InnoDB 存储引擎支持事务，但 MyISAM 存储引擎不支持事务

### <font color=#1FA774>参考文章</font>

- **[Spring 事务详解](https://javaguide.cn/system-design/framework/spring/spring-transaction.html)**

- **[Spring事务传播行为详解](https://segmentfault.com/a/1190000013341344)**