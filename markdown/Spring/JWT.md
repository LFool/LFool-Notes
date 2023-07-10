# JWT

在正式介绍 JWT 之前，先来讲讲什么是认证、授权、Cookie、Session、Token ...

### <font color=#1FA774>认证与授权</font>

**认证**是指验证当前用户的身份，常见的有使用账号密码登录、手机号验证码等方式认证身份

**授权**是指用户拥有的操作权限，比如非常核心的功能只有超级管理员可以操作，而一般核心的功能管理员可以操作，而普通用户只能操作一般的功能

一般数据库中会存储用户的认证和授权信息，比如账号密码、用户拥有的权限等信息

- 每次登录时和数据库比对账号密码，只有信息无误后才算认证 (登录) 成功
- 认证成功后会查询用户对应权限，并将权限保存在 Cookie、Session 或 Token 中

### <font color=#1FA774>Cookie、Session、Token</font>

HTTP 是无状态的，但大多数场景都需要保存用户的登录状态，就出现了三种不同的方式：Cookie、Session、Token

**Cookie：**服务器响应中包含会话状态，浏览器需要在本地保存这些状态信息，后续的请求需要将这些状态添加到请求头中一起发送给服务器，服务器便可进行会话跟踪

**Session：**基于 Cookie 实现，不同在于会话状态保存在服务器，仅仅响应 SessionID 给浏览器，后续的请求只需要将 SessionID 添加到请求头的 Cookie 中即可

**Token：**访问资源的凭证，用户认证成功后服务器会签发一个 Token，包含用户认证和授权信息，服务器不需要记录状态

**<font color='red'>问题一：</font>**禁用 Cookie 后，怎么使用 Session？

在 HTTP 请求头中有一个 Cookie 字段，Session 基于 Cookie 实现，只需要将 SessionID 添加到请求头中即可，如：`Cookie: SessionID`

所以 Cookie 对于 Session 来说只是一个传递 SessionID 的媒介，当 Cookie 被禁用后，可以将 SessionID 添加到 URL 中，如：`https://xxx.com?SessionID=xxx`

**<font color='red'>问题二：</font>**Session 和 Token 的异同？

首先，Session 和 Token 都是保存用户登录信息的机制，可以让用户在多个页面切换时服务器依旧可以自动获取登录信息，无需重新登录

然后，Session 和 Token 不同之处有：

- Session 记录的是会话信息，保存在服务器，通信过程中传递的只是 SessionID；Token 记录的是用户认证和授权信息，通信过程中传递的就是 Token
- 根据第一点，Session 机制下服务器是有状态的，根据 SessionID 查询会话状态信息；Token 机制下服务器无状态，所有信息全部保存在 Token 中，解析 Token 即可获得信息
- Session 机制是底层验证请求是否属于同一个会话，存在跨域问题；Token 机制完全由应用程序管理，可以避免同源策略 (必须同协议、域名、端口)

### <font color=#1FA774>分布式会话</font>

基于 Session 机制的认证过程如下：

- 用户向服务器发送用户名和密码
- 服务器验证通过后，在当前会话 (session) 里面保存相关数据，比如用户角色、登陆时间等
- 服务器会向用户返回一个 SessionID，写入到用户的 Cookie 中
- 用户随后的每一次请求，都会通过 Cookie 将 SessionID 传回服务器
- 服务器收到 SessionID 后，找到前期保存的数据，由此得知用户的身份

因为 Session 保存在服务器中，这种方式在单机中毫无问题，但如果是服务器集群，那么就可能出现同一个用户两次请求发送到不同服务器，导致无法获取会话状态信息的情况

针对这种情况，主要有两种解决方案：

- 不再把 Session 存储在集群中某个节点中，而是集中存储，比如存储在 Redis 中，每次都去 Redis 中获取 Session 状态信息
- 服务器不记录状态，用户的认证和授权信息全部放在 Token 中，在通信中直接传输 Token

### <font color=#1FA774>JWT</font>

前面介绍了这么多，下面正式开始介绍 JWT (JSON Web Token)。当提到 Token 时，一般都是指 JWT，虽然 Token 不同的实现方式，但 JWT 已经成为事实上的标准

JWT 原理：服务器认证成功后，生成一个 JSON 对象，发回给用户，如下：

```json
{
    "姓名": "张三",
    "角色": "管理员",
    "到期时间": "2023-07-10 07:04:58"
}
```

此后，用户与服务器通信时，都需要发回这个 JSON 对象，服务器依靠该对象认证用户身份。为了防止数据被篡改，服务器在生成该对象时会加上数字签名

有了 JWT 之后，服务器就不需要存储任何 Session 信息，也就变成无状态，也方便扩展

### <font color=#1FA774>JWT 数据结构</font>

在介绍 JWT 原理时，只是简单的说了一下 JWT 是一个 JSON 对象，但这并不严谨～～

从结果上来看，JWT 是一个很长的字符串，中间用`.`将其分割成三个部分，可以在网站 **[jwt.io](https://jwt.io/)** 上对字符串进行解码。下面给出一个 JWT 例子：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

JWT 的三个部分分别为：

- **Header (头部)**
- **Payload (负载)**
- **Signature (签名)**

Header 部分是一个 JSON 对象，描述 JWT 的元数据，通常如下：

```json
{
    "alg": "HS256",  // 表示数字签名的算法，默认 HMAC SHA256 (HS256)
    "typ": "JWT"     // 表示 Token 的类型，JWT 令牌统一写成 JWT
}
```

Payload 部分也是一个 JSON 对象，用来存放实际需要传递的数据，JWT 规定了 7 个官方字段，供选用：

- **iss (issuer)：**签发人
- **exp (expiration time)：**过期时间
- **sub (subject)：**主题
- **aud (audience)：**受众
- **nbf (Not Before)：**生效时间
- **iat (Issued At)：**签发时间
- **jti (JWT ID)：**编号

除了官方规定的字段外，还可以定义私有字段，如下所示：

```json
{
    "sub": "1234567890",
    "name": "John Doe",
    "iat": 1516239022
}
```

**<font color='red'>注意：</font>**JWT 默认不加密，任何人都可以读到，所以不要把秘密信息放在这个部分

Signature 部分是对前两个部分的签名，防止数据被篡改。首先需要一个签名密钥，该密钥只有服务器知道，不能泄漏给其它人。按照下面公式生成签名：

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

### <font color=#1FA774>JWT 使用方式</font>

客户端收到服务器返回的 JWT 后，可以存储在 Cookie 中，也可以存储在 localStorage 中。如果存储在 Cookie 中，每次请求都会自动发送，但这样不能跨域

更好的做法是将 JWT 放在 HTTP 请求头的`Authorization`字段中：

```html
Authorization: Bearer <token>
```

### <font color=#1FA774>JWT 特点</font>

- JWT 默认不加密，但也可以在生成 Token 后使用密钥加密一次
- JWT 在不加密的情况下，不能将秘密数据写入 JWT 的 Payload 部分
- JWT 可用于认证、授权，还可以用于信息交换，有效的使用 JWT 可以减少数据库查询的次数，只需要解析 JWT 即可
- JWT 最大缺点就是签发后直到过期之前始终有效，无法中途废止某个 Token
- JWT 包含了认证信息，一旦泄漏，任何人可以获取该令牌所有权限。为了减少盗用，尽量将 JWT 有效期设置短一些
- 为了避免请求报文在传输过程中被非法截获导致 JWT 泄漏，应该使用 HTTPS 协议密文传输

### <font color=#1FA774>JWT Demo</font>

下面给出一个自己实现的 JWT Demo，虽然它只是一个小 Demo，但里面涉及到的内容还挺多的，先给出项目仓库链接 **[JWT Demo](https://github.com/LFool/JWT-Demo.git)**

总结一下这个 Demo 中包含的内容：

- 使用 JWT 进行认证和授权
- 使用`@ControllerAdvice`和`@ExceptionHandler`实现全局异常处理 (**详情可见 [统一异常处理](./Spring-MVC.html#统一异常处理)**)
- 使用 Spring MVC 拦截器，拦截 Controller 层方法，执行方法前统一进行访问权限控制 (**详情可见 [拦截器](./Spring-MVC.html#font-color1fa774拦截器font)**)
- 实现自定义注解，结合拦截器，在需要进行访问权限控制的方法上添加`@NeedToken`，在不需要进行访问权限控制的方法上添加`@PassToken`，这里使用了反射机制获取方法上的注解

### <font color=#1FA774>参考文章</font>

- **[傻傻分不清之 Cookie、Session、Token、JWT](https://juejin.cn/post/6844904034181070861)**
- **[Cookie和Token](https://www.jianshu.com/p/ce9802589143)**
- **[JSON Web Token 入门教程](http://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)**