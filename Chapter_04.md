# 第四章 大规模可扩展内容缓存

## 4.0 介绍

缓存通过存储上次请求的响应给接下来的请求服务实现加速内容服务。缓存完整的响应，而不是对相同的请求再次运行计算和查询，内容缓存可减少上游服务器的负载。缓存可以提高性能并减少负载，这意味着您可以用更少的资源提供更快的服务。有策略的扩展和分布缓存服务器的位置，可以对用户体验产生巨大的影响。为了获得最佳性能，最好将内容托管在靠近消费者的地方。你还可以将内容缓存设置在靠近用户的地方。这就是内容分发网络(content delivery networks，CDN)的模式。有了NGINX，你就可以在任何可以放置NGINX服务器的地方，缓存你的内容，从而有效地让你创建自己的CDN。使用NGINX缓存，你还可以在上游发生故障时被动地缓存，并提供缓存的响应。（就是说上游戏服务器宕机了还可以响应缓存的内容）

## 4.1 缓存的位置

### 问题

你需要缓存内容，并且需要去指定缓存存储的位置。

### 解答

使用`proxy_cache_path`指令指定共享内存缓存区和内容的位置：

```
proxy_cache_path /var/nginx/cache
                keys_zone=CACHE:60m
                levels=1:2
                inactive=3h
                max_size=20g;
proxy_cache CACHE;
```
上面的缓存位置指定示例中，在文件系统中为缓存的响应创建一个目录`/var/nginx/cache`，并且创建一个名为名为`CACHE`的拥有`60MB`内存的共享内存空间。这个示例设置了目录结构级别，并定义了在`3小时`内缓存的响应未被再次请求则初释放，并且定义了缓存的最大大小为`20GB`。`proxy_cache`指令规定了特定上下文使用的缓存区域。`proxy_cache_path`在`http`上下文中有效，而`proxy_cache`指令在`http`，`server`和`location`上下文中有效。

### 讨论

要在NGINX中配置缓存，必段要指定使用的路径和区域。而在NGINX中缓存区域是用`proxy_cache_path`指令创建的。`proxy_cache_path`指定存储缓存信息的位置和共享内存空间存储活动关键字和响应元数据。这个指令的可选参数为如何维护和访问缓存提供了更多的控制。`levels`参数定义了如何创建文件结构。它的值是一个以冒号分隔的值，它声明子目录名的长度，最多有三层。NGINX根据cache key(一个hash哈希值)进行缓存, 然后NGINX将结果存储在提供的文件结构中，使用cache key作为文件路径，并根据`levels`值进行目录拆分。`inactive`参数允许控制上一次使用缓存项后将托管该缓存项的时间长度。缓存的大小也可以使用`max_size`参数进行配置。其他参数与缓存加载过程相关，该过程将缓存键从缓存在磁盘上的文件加载到共享内存区域。

注：
> `levels`参数负责设置缓存目录级别。假设cache主目录为`/var/nginx/cache`。
>
> * 如果没有特殊指明此参数值，则默认是放在cache主目录下：
> `/var/nginx/cache/d7b6e5978e3f042f52e875005925e51b`
>
> * 当levels=1:2时，表示是两级目录，1和2表示用1位和2位16进制来命名目录名称。在此例中，第一级目录用1位16进制命名，如b；第二级目录用2位16进制命名，如2b。所以此例中一级目录有16个，二级目录有16*16=256个：
> `/var/nginx/cache/b/2b/d7b6e5978e3f042f52e875005925e51b`<br>
> 总目录数为16*256=4096个。
>
> * 当levels=1:1:1时，表示是三级目录，且每级目录数均为16个：
> `/var/nginx/cache/b/c/d/d7b6e5978e3f042f52e87500592`<br>
> 总目录数为16*16*16=4096个。
>
> * 当levels=2:2:2时，表示是三级目录，且每级目录数均为16*16个：
> `/var/nginx/cache/2b/3c/4d/d7b6e5978e3f042f52e875005925e51b`<br>
> 总目录数为256*256*256个。
>
> * 当levels=2时，表示是一级目录，且目录数为16*16=256：
>
> 引用来源：[《Nginx proxy_cache_path命令之levels参数详解》](https://blog.csdn.net/cup_chenyubo/java/article/details/102687861)

## 4.2 缓存Hash Key

### 问题

你需要控制内容的缓存和查找方式。

### 解答

使用`proxy_cache_key`指令和变量来定义缓存命中或未命中的组成部分（即定义根据什么来命中缓存）:

```
proxy_cache_key "$host$request_uri $cookie_user";
```
这个缓存cache key会指示NGINX根据所请求的主机和URI以及定义用户的cookie来缓存页面。这样，你就可以缓存动态页面了，而不需要为不同用户额外再生成内容（缓存不只是静态内容，也可以缓存动态的）。

### 讨论

默认的`proxy_cache_key`适用于大多数情况，为`"$scheme$proxy_host$request_uri"`。使用的变量包括Scheme，HTTP或HTTPS、发送请求的`proxy_host`和请求URI, 因而，这代表NGINX代理请求的URL。你可能会发现，为每个应用程序定义惟一的请求还有许多其他因素，比如请求参数、请求头Header、会话Session标识符等等，你将希望为它们创建自己的Hash Key。【*注: NGINX暴露的文本或变量的任何组合都可以用于形成缓存键。 NGINX暴露的变量列表可以在这里找到：[http://nginx.org/en/docs/varindex.html](http://nginx.org/en/docs/varindex.html)*】

选择一个好的Hash Key非常重要，应该在理解应用程序的基础上进行思考。选择静态内容的缓存键通常非常简单；使用主机名和URI就足够了。为仪表板应用程序（Dashboard application）页面等相当动态的内容选择缓存键需要更多有关用户如何与应用程序交互以及用户体验之间的差异度的知识。出于安全方面的考虑，你可能不希望在不完全了解上下文的情况下将一个用户的缓存数据呈现给另一个用户。`proxy_cache_key`指令配置要对缓存键Cache key进行hash的字符串。`proxy_cache_key`指令可以设置在`http`上下文，`server`和`location`块，从而为如何缓存请求提供了灵活控制。