# DNS

每一台公网服务器都有一个公网 IP，通过 IP 可以访问它，但世界上服务器那么多，而且 IP 是一串数字，难免不容易记住对应的 IP 地址，所以就有了 DNS (Domain Name System，域名系统)

DNS 记录了 IP 和域名的对应关系，在通信时需要使用 IP 地址，而域名更便于用户记住，所以用户通过域名访问一个网站时，域名服务器会自动将域名转换成 IP 地址

在配置计算机的 IP 相关信息时，往往会配置一个 DNS 服务器的 IP 地址，可以将它称之为本地 DNS 服务器，我们是借助它来获取域名对应的 IP 地址

### <font color=#1FA774>DNS 结构</font>

所有的 DNS 域名服务器是一个树状结构，最上层的是根域，如下图所示：

![9](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230524/0518271684876707CxbUrR9.svg)

### <font color=#1FA774>IP 查询过程</font>

下面模拟`www.baidu.com`的查询过程！先给出过程图：

![10](https://cdn.jsdelivr.net/gh/LFool/new-image-hosting@master/20230524/05513216848786921i751g10.svg)

**第一步：**询问配置的本地 DNS 服务器，`www.baidu.com`的 IP 地址是多少？如果本地 DNS 服务器发现自己本地缓存中没有对应的域名就去根域名服务器询问，如果有缓存中有直接返回给计算机

**第二步：**本地 DNS 服务器询问根域名服务器，根域名服务器会让本地 DNS 服务器去询问`com`域名服务器

**第三步：**本地 DNS 服务器询问`com`域名服务器，`com`域名服务器会让本地 DNS 服务器去询问`baidu`域名服务器

**第四步：**本地 DNS 服务器询问`baidu`域名服务器，`baidu`域名服务器会让本地 DNS 服务器去询问`www`域名服务器

**第五步：**本地 DNS 服务器询问`www`域名服务器，`www`域名服务直接返回对应的 IP 地址，本地 DNS 服务器会缓存「域名-IP」对应关系，但该缓存有时间期限，一段时间会过期