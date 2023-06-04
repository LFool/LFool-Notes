# GET && POST 请求

在 HTPP/1.1 中，一共有 get、post、put、head、delete、potions、trace、connect 八种不同的请求方法，但本篇文章只介绍最常用的两种 get 和 post

### <font color=#1FA774>GET</font>

GET 请求是**从服务器获取指定资源**，指定的资源经服务器返回给客户端，资源可以是静态文本、页面、图片视频等

GET 请求的参数位置一般写在 URL 中，URL 只能支持 ASCII，所以 GET 请求的参数只允许是 ASCII 字符，而且浏览器会对 URL 长度有限制，HTTP 协议本身没有对 URL 长度做任何规定

下面给出一个 GET 请求的示例：`https://www.test.com/get?xxx=111&yyy=222`。该请求中有两个参数：`xxx = 111`和`yyy = 222`

### <font color=#1FA774>POST</font>

POST 请求是根据请求负载对指定资源做出处理，一般请求携带的数据会写在消息体中，消息体中的数据可以是任意格式，且浏览器不会限制消息体大小

GET 请求侧重于获取指定的资源，而 POST 请求侧重于传输实体到服务器进行处理，如登陆时提交表格信息

下面给出一个 post 请求的示例：

```
https://www.test.com/post

xxx = 111
yyy = 222
```

该请求体中有两个参数：`xxx = 111`和`yyy = 222`

### <font color=#1FA774>安全和幂等</font>

首先说一下安全和幂等着两个概念：

- **安全：**在 HTTP 协议里，安全表示不会修改服务器中的数据，类似于只读操作
- **幂等：**多次请求操作结果相同

很显然，GET 请求是安全且幂等的，而 POST 是不安全且不幂等的。所以 GET 请求返回的结果可以被缓存，而 POST 请求返回的结果不可以被缓存
