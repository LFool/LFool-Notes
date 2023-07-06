# Spring MVC

### <font color=#1FA774>MVC 模式</font>

MVC (Model-View-Controller) 常用于应用程序分层开发

- **Model (M，模型)：**表示一个存取数据的对象
- **View (V，视图)：**表示模型对象的可视化对象
- **Controller (C，控制器)：**作用于模型和视图上，控制数据流向模型，并在数据变化时更新视图

举个简单例子，可能前端只需要一个对象的部分属性，或者多个对象的部分属性，直接可以封装一个视图对象，其中包含所需的属性，可能选取一个对象的部分属性，也可能将多个对象部分属性组合

从这个例子可以看出视图和模型实现了解耦，可以根据业务要求封装前端所需要的视图对象，实现视图和模型之间的交互由 Controller 负责，下面给出关系图：

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230705/0255261688496926AavCKg1.svg)

**<font color='red'>注意：</font>**上面例子基于前后端分离的设计思想，可能跟一般情况有区别，但本质上是相同的～

### <font color=#1FA774>Spring MVC 执行流程</font>

Spring MVC 的核心思想也是基于上面介绍的 MVC 模式，但相对而言处理更加复杂一些，先给出一张执行流程图：

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230705/1540281688542828ux1MH32.svg)

**<font color='red'>注意：</font>**为了画图方便，图中的 MV 表示 ModelAndView，V 表示 View

先解释一下上图中各个组件的作用：

- **DispatcherServlet：**前端控制器，相当于 MVC 模式中的 C，负责控制整个流程，调用其它组件，用户请求首先会达到此处
- **HandlerMapping：**处理器映射，负责根据用户请求的 URL 找到 Handler 处理器
- **Handler：**处理器，涉及到具体的用户业务请求，一般由开发人员负责实现 Handler，即开发人员编写的 Controller 层
- **HandlerAdapter：**处理器适配，是适配器设计模式的应用
- **ModelAndView：**SpringMVC 封装的对象，将 Model 和 View 封装在一起
- **ViewResolver：**视图解析器，首先根据逻辑视图名解析成物理视图名，即具体的页面地址，然后生成 View 视图对象
- **View：**SpringMVC 封装的视图对象

下面给出一次完成请求的处理流程：

- 浏览器发送请求，首先会到达`DispatcherServlet`
- `DispatcherServlet`调用`HandlerMapping`查找`Handler`，并将请求涉及到的拦截器和`Handler`一起封装
- `DispatcherServlet`调用`HandlerAdapter`适配器执行`Handler`
- `Handler`完成对用户请求的处理后，返回一个`ModelAndView`给`DispatcherServlet`
- `DispatcherServlet`调用`ViewResolver`根据逻辑`View`解析到实际`View`，此时`View`是一个还没有嵌入数据的页面
- `DispatcherServlet`将`Model`传给`View`进行渲染，将对应数据嵌入`View`中
- 把`View`返回给浏览器

**<font color='red'>总结：</font>**查找 Handler -> 适配器执行 Handler -> 解析 ModelAndView -> 渲染 View -> 返回

### <font color=#1FA774>拦截器</font>

SpringMVC 实现拦截器是基于 **[AOP](./Spring-AOP.html)**，底层是 **[动态代理](../java/代理模式-静态-动态.html#动态代理)**，简单来说当用户请求`/user/login`时，并没有直接调用 Controller 对象的方法，而是通过代理对象去调用目标方法

增加一层代理的好处在于可以增强某个方法或一类方法，在执行目标方法前/后进行统一处理，实现了业务功能和非业务功能的解耦，其实这就是 AOP 机制的核心思想，而拦截器就是运用 AOP 实现滴

下面给出 SpringMVC 实现拦截器的一个小 Demo：

```java
// 实现 HandlerInterceptor 接口
public class MyInterceptor implements HandlerInterceptor {
    /**
     * 在请求之前进行调用处理 (Controller 方法调用之前)
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("拦截器 --- preHandle");
        return true;
    }
    /**
     * 在请求处理之后，视图渲染之前调用 (Controller 方法调用之后，方法返回之前)
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("拦截器 --- postHandle");
    }
    /**
     * 在请求之后进行调用处理 (Controller 方法返回之后)
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("拦截器 --- afterCompletion");
    }
}

// 在配置文件中注册拦截器
@Configuration
public class MyConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        InterceptorRegistration registration = registry.addInterceptor(new MyInterceptor());
        registration.addPathPatterns("/**");  // 所有路径都被拦截
        registration.excludePathPatterns(     // 不拦截的路径
                "/login/**"
        );
    }
}
```

### <font color=#1FA774>统一异常处理</font>

在 **[AOP](./Spring-AOP.html)** 中说异常处理也可以统一处理，即在方法抛出异常后统一处理，这样可以最大程度的降低业务逻辑和非业务逻辑的耦合程度

可以使用`@ControllerAdvice`和`@ExceptionHandler`这两个注解实现统一异常处理，也就是在指定方法抛出异常后统一处理，无需在每个方法中单独处理

在这种异常处理方式下，SpringMVC 会给所有或指定`Controller`织入异常处理逻辑，当`Controller`中的方法抛出异常后，由被`@ExceptionHandler`注解修饰的方法进行处理

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    @ResponseBody
    public Object doError(Exception exception) {
        exception.printStackTrace();
        Map<String, String> res = new HashMap<>();
        if (exception instanceof RuntimeException) {
            res.put("RuntimeException", "RuntimeException");
        } else if (exception instanceof MyException) {
            res.put("MyException", "MyException");
        } else {
            res.put("Exception", "Exception");
        }
        return res;
    }
}
```