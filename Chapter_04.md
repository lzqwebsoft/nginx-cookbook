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

默认的`proxy_cache_key`适用于大多数情况，为`"$scheme$proxy_host$request_uri"`。使用的变量包括Scheme，HTTP或HTTPS、发送请求的`proxy_host`和请求URI, 因而，这代表NGINX代理请求的URL。你可能会发现，为每个应用程序定义惟一的请求还有许多其他因素，比如请求参数、请求头Header、会话Session标识符等等，你将希望为它们创建自己的Hash Key。【*注: NGINX公开的文本或变量的任何组合都可以用于形成缓存键。 NGINX公开的变量列表可以在这里找到：[http://nginx.org/en/docs/varindex.html](http://nginx.org/en/docs/varindex.html)*】

选择一个好的Hash Key非常重要，应该在理解应用程序的基础上进行思考。选择静态内容的缓存键通常非常简单；使用主机名和URI就足够了。为仪表板应用程序（Dashboard application）页面等相当动态的内容选择缓存键需要更多有关用户如何与应用程序交互以及用户体验之间的差异度的知识。出于安全方面的考虑，你可能不希望在不完全了解上下文的情况下将一个用户的缓存数据呈现给另一个用户。`proxy_cache_key`指令配置要对缓存键Cache key进行hash的字符串。`proxy_cache_key`指令可以设置在`http`上下文，`server`和`location`块，从而为如何缓存请求提供了灵活控制。

## 4.3 缓存绕过

### 问题

你需要能够绕过缓存。

### 解答

使用`proxy_cache_bypass`指令通过指定非空值或非零值来实现。其中的一种方法就是在`location`块中设置一个变量，在你不需要缓存的时候等于1：

```
proxy_cache_bypass $http_cache_bypass;
```
这个配置告诉NGINX，如果名为`cache_bypass`的HTTP请求头被设置为任何不为0的值时则绕过缓存。


### 讨论

有许多场景需要不缓存请求，为此，NGINX公开了`proxy_cache_bypass`指令。因此，当值为非空或非零时，请求将被发送到上游服务器，而不是从缓存中提取。绕过缓存的不同需求和场景将由对应的应用程序决定。绕过缓存的技术可以像使用请求或响应头那样简单，也可以像多个映射块一起工作那样复杂。

由于许多原因，您可能希望绕过缓存。一个重要的原因是故障排除和调试。如果一直拉取缓存的页面，或者缓存键特定于某个用户标识符，那么重现问题将变的很困难，因此如果能绕过缓存显的很有必要。绕过缓存的选项包括但不限于在设置特定cookie、header或请求参数。还可以通过将`proxy_cache`设置为`off`；比如在`location`块中完全关闭给定上下文的缓存。

## 4.4 缓存性能

### 问题

你需要通过在客户端设置缓存来提高性能。

### 解答

使用客户端缓存控制头：

```
location ~* \.(css|js)$ {
    expires 1y;
    add_header Cache-Control "public";
}
```
这个`location`块指定客户端可以缓存CSS和Javascript文件。其中的`expires`指令指示客户端缓存的资源的有效期是一年。如上的`add_header`指令将HTTP响应头Header中添加`Cache-Control`标识，并且设置的值为`public`，这允许沿途的任何缓存服务器缓存资源。相反的如果我们指定为`private`，则仅允许客户端缓存该值。

### 讨论

导致缓存性能的有很多方面的因素，磁盘速度一定是其中最重要的因素。在NGINX配置中有很多方法可以帮助提高缓存性能，其中的一种方法就是设置响应头，这样客户端实际上缓存了响应，并且根本不会再向NGINX服务器再发出请求，而只是从自己磁盘上读取缓存。

## 4.5 清除

### 问题

你需要使缓存中的对象无效化。

### 解答

使用NGINX Plus的清除功能，`proxy_cache_purge`指令以及非空或零值变量：

```
map $request_method $purge_method {
    PURGE 1;
    default 0;
}
server {
    ...
    location / {
        ...
        proxy_cache_purge $purge_method;
    }
}
```
在本例中，如果使用`PURGE`方法请求特定对象的缓存，则该对象的缓存将被清除。下面是一个清除名为`main.js`文件缓存的`curl`示例：

```bash
$ curl -XPURGE localhost/main.js
```

### 讨论

处理静态文件的一种常见方法是在文件名中放入文件的哈希值（即请求路径加上文件的hash值），这样可以确保在推出新代码和内容时，由于URI更改，CDN会将其识别为新文件。但是，这不适用于，你设置了不适合此模型的缓存键（cache keys）的动态内容。在每个缓存场景中，都必须有清除缓存的方法。NGINX Plus提供了一种清除缓存响应的简单方法。 当`proxy_cache_purge`指令传递非零或非空值时，将清除与请求匹配的缓存项。设置清除的一种简单方法是通过map映射PURGE的请求方法（request method）。但是，你可能还希望将其与`geo_ip`模块或简单身份验证结合使用，以确保没有人能够清除你的宝贵缓存项。NGINX还允许使用`*`(通配符)，它将清除与公共URI前缀匹配的缓存项。要使用通配符，需要使用`purger=on`参数配置`proxy_cache_path`指令。

## 4.6 缓存切片

### 问题

你需要通过将文件分割成片段来提高缓存效率。

### 解答

使用NGINX slice指令及其内嵌变量将缓存结果划分为片段:

```
proxy_cache_path /tmp/mycache keys_zone=mycache:10m;
server {
    ...
    proxy_cache mycache;
    slice 1m;
    proxy_cache_key $host$uri$is_args$args$slice_range;
    proxy_set_header Range $slice_range;
    proxy_http_version 1.1;
    proxy_cache_valid 200 206 1h;

    location / {
        proxy_pass http://origin:80;
    }
}
```

### 讨论

此配置定义了一个缓存区域，并为服务器启用它。然后使用`slice`指令指示NGINX将响应切片成1MB大小的文件段。缓存文件根据`proxy_cache_key`指令存储。注意，使用了名为`slice_range`的嵌入式变量。在向原服务器发出请求时，同样的变量`slice_range`被用作请求头。并且请求HTTP版本被升级为HTTP/1.1，因为1.0不支持字段域byterange请求。响应码为`200`或`206`的缓存有效性设置为1小时，然后定义`location`和被代理的请求源。

缓存切片模块是为传递HTML5视频而开发的，该模块使用字段域byterange请求将内容伪流传输到浏览器。默认情况下，NGINX能够从它的缓存中提供字段域byterange请求。如果请求未缓存内容的字段域，则NGINX从源中请求整个文件。当你使用缓存切片模块时，NGINX仅从源请求必要的段。大于切片大小的范围请求（包括整个文件）将触发每个所需段的子请求，然后将这些段缓存。当所有的段都被缓存后，响应被组装并发送到客户端，使NGINX可以更有效地缓存并提供范围内请求的内容。缓存切片模块仅应用于不会更改的大文件。NGINX每次从源器接收到一个片段时都会对其进行ETag验证。如果源上的ETag发生更改，则NGINX将中止事务，因为缓存不再有效。如果内容确实发生变化并且文件较小，或者你的源可以在缓存填充过程中处理高峰负载。最好使用下面“另请参见”博客中描述的缓存锁模块。

### 另请参见

[Smart and Efficient Byte-Range Caching with NGINX & NGINX Plus](https://www.nginx.com/blog/smart-efficient-byte-range-caching-nginx/)

注：[RFC7233 HTTP范围请求(Range Requests)](https://blog.csdn.net/u012062760/article/details/77096479)