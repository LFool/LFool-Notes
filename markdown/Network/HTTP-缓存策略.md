# HTTP 缓存策略

当使用浏览器访问一个 Web 资源时，为了提高访问效率，往往会在浏览器和服务器之间增加本地缓存

对于首次访问，会将服务器响应的数据缓存到本地；对于后续的访问，会根据缓存策略决定是否直接使用本地缓存。简化的流程如下图所示：

![1](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230526/0700211685055621d1lOGz1.svg)

**<font color='red'>注意：</font>**缓存策略完全是根据服务器响应消息的头信息来决定滴！！

### <font color=#1FA774>与缓存有关的消息头字段</font>

先给出与缓存有关的消息头字段的一些说明：

![2](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230526/0731241685057484ZZ8guB2.svg)

### <font color=#1FA774>强制缓存</font>

顾名思义，强制缓存不需要服务器的参与，浏览器可以自己决策是否使用缓存，决策规则如下：

- 如果本地缓存中不存在要请求的资源，浏览器会直接将请求发送到服务器，完成首次资源请求，根据服务器响应的消息保存相关信息
- 如果本地缓存中存在要请求的资源，先判断缓存是否过期，如果没有过期直接使用本地缓存，如果过期就需要根据下面要介绍的协商缓存进行处理

在响应消息中，Expires 是根据 Last-Modified 和 max-age 计算出来的，max-age 表示存活多少秒后过期，而 Last-Modified 最后的修改时间

如果一个请求直接强制使用缓存，那么该请求会显示`200 OK (from disk cache)`，如下所示：

```
Request URL: http://192.168.124.13:8080/index.html
Request Method: GET
Status Code: 200 OK (from disk cache)
Remote Address: 192.168.124.13:8080
Referrer Policy: strict-origin-when-cross-origin
```

### <font color=#1FA774>协商缓存</font>

顾名思义，协商缓存是浏览器和服务器共同协商是否使用本地缓存，浏览器负责提供本地缓存的信息，如：最后修改时间和唯一标识，添加在请求消息头中，服务器收到消息后来决策是否允许浏览器使用本地缓存

If-Modified-Since 和 If-None-Match 是在浏览器第一次发送请求到服务器时，从服务器的响应消息保存。如果浏览器发现本地缓存中不存在要请求的资源，会直接将请求发送到服务器

协商缓存基于两种消息头字段实现：

- **第一种：**请求头中的 If-Modified-Since 和响应头中的 Last-Modified
- **第二种：**请求头中的 If-None-Match 和响应头中的 Etag

对于第一种，如果 If-Modified-Since 时间小于 Last-Modified，表示本地缓存中是旧数据，不可以使用本地缓存中的数据，否则可以使用缓存

对于第二种，如果 If-None-Match 和 Etag 不相等，表示本地缓存中是旧数据，不可以使用本地缓存中的数据，否则可以使用缓存

如果一个请求消息头中同时包含 If-Modified-Since 和 If-None-Match 字段，那么 If-None-Match 优先级更高。先判断 If-None-Match，如果不匹配直接无法使用缓存，如果匹配，再根据 If-Modified-Since 判断

**<font color='red'>问题：</font>**为什么 Etag 的优先级更高？？

Etag 是服务器中资源的唯一标识，它可以解决一些 Last-Modified 难以解决的问题

- 可能存在文件没有被修改，但修改时间却发生了改变的情况
- Last-Modified 的精度是秒，对于秒以内的变化，Last-Modified 无法感知；但 Etag 却可以保证秒以内多次修改都可以被感知

**<font color='red'>强制缓存和协商缓存区别：</font>**强制缓存不需要将请求发送给服务器，浏览器自己可以决策；协商缓存需要将请求先发送给服务器，由服务器决策，如果可以使用缓存会响应一个 **[304](./HTTP-状态码.html#3xx)** 的消息，表示缓存重定向

### <font color=#1FA774>总流程图</font>

下面给出缓存策略的总体流程图：

![3](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230526/0841121685061672KvZqRS3.svg)

**<font color='red'>注意：</font>**

- 使用 F5 刷新网页时，跳过强制缓存，但会检查协商缓存
- 使用 ctrl + F5 刷新页面时，直接从服务器加载，跳过强制缓存和协商缓存

### <font color=#1FA774>参考文章</font>

- **[HTTP 缓存技术](https://xiaolincoding.com/network/2_http/http_interview.html#http-缓存技术)**
- **[彻底弄懂浏览器缓存策略](https://mp.weixin.qq.com/s/Ui7Q9k4faiD5mv_LfB4Rrw)**
- **[Cache-Control在请求头和响应头里的区别](https://juejin.cn/post/6960988505816186894)**
- **[为什么我的缓存设置在 chrome 中不生效](https://xie.infoq.cn/article/70b359ab1210efb6f5b253e24)**
- **[Cache-Control和Expires无效的原因](https://www.cnblogs.com/jimaww/p/10234074.html)**