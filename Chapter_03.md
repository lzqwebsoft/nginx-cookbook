# 第三章 网络流量管理

## 3.0 介绍

NGINX和NGINX Plus也被归类为网络流量控制器。 你可以使用NGINX使用许多属性智能分配请求路由流量和控制请求流程(flow)。本章介绍NGINX能够基于百分比分配客户端请求，利用客户端的地理位置，以及根据速率，连接和带宽限制的形式控制请求流量的功能。 在阅读本章时，请记住，可以混合使用这些功能，以实现无数种可能性。

## 3.1 A/B测试

### 问题

你需要将应用或文件分为两个或多个不同的版本，以测试它们的用户接受度。

### 解答

使用`split_clients`模块将一定比例的客户端请求定向到不同的上游池（upstream pool）:

```
split_clients "${remote_addr}AAA" $variant {
    20.0% "backendv2";
    * "backendv1";
}
```
`split_clients`指令对你提供的字符串作为第一个参数进行MurmurHash2哈希计算处理，得到一个32位的hash值。并且将hash值按百分比提供映射给一个变量的值作为第二个参数。第三个参数是包含键值对的对象，其中键表示权重百分比，值表示要被分配的值。键可以是百分比或星号。星号表示取所有百分比后的剩余部分。$variant变量的值为客户端IP地址计算hash的20%定向为backendv2，剩余的80%定向为backendv1。

在上面的示例中，backendv1和backendv2表示上游服务器池，可以与`proxy_pass`指令结合一起使用，如下所示：

```
location / {
    proxy_pass http://$variant
}
```

使用变量$variant，我们将网络流量请求按不同的比例分配给两个不同的应用程序服务器池（application server pools即upstream pool）。

### 讨论

当电子商务网站上需要测试不同类型的营销和前端功能的转换率时，这种类型的A/B测试非常有用。应用程序通常使用一种称为[金丝雀发布(canary release)](https://baike.baidu.com/item/%E7%81%B0%E5%BA%A6%E5%8F%91%E5%B8%83)的部署方式。在这种部署方式中，流量将会被缓慢地切换到新版本。在推出新版本的代码之前，在不同版本的应用程序之间切分客户端请求流量可能非常有用，这样可以限制发生错误的影响范围。不管在两个不同的应用程序集之间切分客户端请求流量的原因是什么，都可以在NGINX中使用`split_clients`模块简化实现这一过程。

### 另请参见

[split_client文档(英文版)](http://nginx.org/en/docs/http/ngx_http_split_clients_module.html)

上面的文档中介绍了，将网络流量请求按不同的比例分配给3个不同的文件：

```
http {
    split_clients "${remote_addr}AAA" $variant {
        0.5%               .one;
        2.0%               .two;
        *                  "";
    }

    server {
        ....
        location / {
            index index${variant}.html;
            ...
        }
        ....
    }
}
```

这3个文件分别是 index.one.html分配0.5%的网络流量，index.two.html分配2.0%的网络流量，和index.html分配97.5%的网络流量。

## 3.2 使用GeoIP模块及其数据库

### 问题

你需要安装GeoIP数据库，并启用其在NGINX中嵌入的变量，给应用程序标记记录客户端请求的所属位置。

### 解答

在[第一章](Chapter_01.md)安装NGINX时，官方开源的NGINX版本软件仓库中，提供了一个名为`nginx-module-geoip`的软件包。如果使用的NGINX Plus软件包仓库，则此软件包名为`nginx-plus-module-geoip`。使用这些软件包将安装GeoIP模块的动态版本。

RHEL/CentOS 开源版NGINX：
```bash
$ yum install nginx-module-geoip
```

Debian/Ubuntu 开源版NGINX：
```bash
$ apt-get install nginx-module-geoip
```

RHEL/CentOS NGINX Plus:
```bash
$ yum install nginx-plus-module-geoip
```

Debian/Ubuntu NGINX Plus:
```bash
$ apt-get install nginx-plus-module-geoip
```

下载GeoIP国家和城市数据库并解压缩：

```bash
$ mkdir /etc/nginx/geoip
$ cd /etc/nginx/geoip
$ wget "http://geolite.maxmind.com/\
  download/geoip/database/GeoLiteCountry/GeoIP.dat.gz"
$ gunzip GeoIP.dat.gz
$ wget "http://geolite.maxmind.com/\
  download/geoip/database/GeoLiteCity.dat.gz"
$ gunzip GeoLiteCity.dat.gz
```

这组命令在`/etc/nginx`目录中创建一个`geoip`目录，并转到该新目录，然后在其中下载并解压缩软件包。

使用本地硬盘上的国家和城市的GeoIP数据库，你现在就可以使用NGINX GeoIP模块中提供的嵌入变量转化显示基于客户端IP对应的地理地址:

```
load_module "/usr/lib64/nginx/modules/ngx_http_geoip_module.so";
http {
    geoip_country /etc/nginx/geoip/GeoIP.dat;
    geoip_city /etc/nginx/geoip/GeoLiteCity.dat;
    ...
}
```

`load_module`指令从文件系统上的路径动态加载模块。`load_module`指令仅在主上下文(main context)中有效。`geoip_country`指令设置为GeoIP.dat文件的路径，该文件包含将IP地址映射到国家/地区代码的数据库，并且仅在HTTP上下文（HTTP context）中有效。


### 讨论

这个模块中的`geoip_country`和`geoip_city`指令开启了许多嵌入变量。`geoip_country`指令开启的变量可以使你区分客户端IP地址对应的国家。这些变量包括`$geoip_coun try_code`，`$geoip_country_code3`和`$geoip_country_name`。`$geoip_coun try_code`变量返回两个字母的国家/地区代码，`$geoip_country_code3`变量返回三个字母的国家/地区代码。`$geoip_country_name`变量返回国家的全名。

`geoip_city`指令也开启了许多变量。`geoip_city`指令与`geoip_country`指令启用的变量差不多，只是使用名称不同，例如`$geoip_city_country_code`，`$geoip_city_country_code3`和`$geoip_city_country_name`。其他变量还包括`$geoip_city`，`$geoip_city_continent_code`，`$geoip_latitude`，`$geoip_longitude`和`$geoip_postal_code`，所有这些变量的返回值同它们的描述一样。`$geoip_region`和`$geoip_region_name`返回的值表示的含义为地区，州，省，联邦土地等。`$geoip_region`表示两个字母的州/省代码，而`$geoip_region_name`表示州/省全名。`$geoip_area_code`仅在美国有效，返回表示电话区号的三位数字。

使用这些变量，你可以记录客户端的有关信息。你可以选择将此信息作为header或变量传递给应用程序，或者使用NGINX以特定的方式路由你的流量。

### 另请参见

[GeoIP Update](https://github.com/maxmind/geoipupdate)


## 3.3 根据国家/地区限制访问

### 问题

你需要限制来自特定国家/地区的访问权限以符合合同或应用程序要求。

### 解答

将你要阻止或允许的国家代码映射到变量：

```
load_module "/usr/lib64/nginx/modules/ngx_http_geoip_module.so";
http {
    map $geoip_country_code $country_access {
        "US" 0;
        "RU" 0;
        default 1;
    }
    ...
}
```
上面的`map`会将新变量`$country_access`设置为1或0。如果客户端IP地址来自美国或俄罗斯，则该变量将设置为0。对于其他国家/地区，该变量将设置为1。

现在，在我们的`server`块中，我们将使用if语句拒绝对非美国或俄罗斯籍人士的访问：

```
server {
    if ($country_access = '1') {
        return 403;
    }
    ...
}
```

如果`$country_access`变量为1则，`if`表达式将会得到`True`, 如果为True，则服务器将返回未经授权的403。否则，服务器将正常运行。 从而实现阻止了非美国或俄罗斯的用户请求。

### 讨论

上面精简短小的例子，用于设置仅允许来自两个国家或地区的访问。你可以对这个例子稍加修改以满足你的需求，你还可以根据GeoIP模块提供的其他嵌入变量，以上面的做法为蓝本来实现允许或禁止访问。


## 3.4 查找始发客户端

### 问题

你需要查找始发的客户端IP地址，因为NGINX服务器前面还有一层代理。

### 解答

使用`geoip_proxy`指令来定义你的代理IP地址范围，使用`geoip_proxy_recursive`指令来查找始发IP址址:

```
load_module "/usr/lib64/nginx/modules/ngx_http_geoip_module.so";
http {
    geoip_country /etc/nginx/geoip/GeoIP.dat;
    geoip_city /etc/nginx/geoip/GeoLiteCity.dat;
    geoip_proxy 10.0.16.0/26;
    geoip_proxy_recursive on;
    ...
}
```

`geoip_proxy`指令定义了代理服务器所在的[CIDR](https://baike.baidu.com/item/%E6%97%A0%E7%B1%BB%E5%88%AB%E5%9F%9F%E9%97%B4%E8%B7%AF%E7%94%B1)范围，并指示NGINX利用请求头`X-Forwarded-For`查找客户端IP地址。`geoip_proxy_recursive`指令指示NGINX递归查看`X-Forwarded-For`请求头，以获取最终的一个客户端IP地址。

### 讨论

在实际项目中你可能会发现，如果你在NGINX前面使用了代理，则NGINX将获取的是代理的IP地址而不是客户端的IP地址。为此，当连接被指定范围内的请求打开时，使用`geoip_proxy_recursive`指令从`X-Forwarded-For`请求头中获取客户端始发IP地址。`geoip_proxy`指令接受地址或CIDR范围。当有多个代理在NGINX前面传递流量时，你可以使用`geoip_proxy_recursive`指令以递归方式搜索`X-Forwarded-For`地址以找到始发客户端。当你在NGINX的前面使用AWS ELB，Google或Azure等等的负载均衡器时，你将会需要使用这样的方法。

## 3.5 限制连接

### 问题

您需要根据预定义的关健字（例如客户端的IP地址）限制连接数。

### 解答

构造一个共享内存区域来保存连接指标，并使用`limit_conn`指令限制打开的连接：

```
http {
    limit_conn_zone $binary_remote_addr zone=limitbyaddr:10m;
    limit_conn_status 429;
    ...
    server {
        ...
        limit_conn limitbyaddr 40;
        ...
    }
}
```

上面配置中创建的共享内存区域名为`limitbyaddr`，预定义关健字使用二进制形式的客户端的IP地址。共享内存区域的大小设置为10 MB。指令`limit_conn`接受两个参数：一个`limit_conn_zone`名称以及允许的连接数。`limit_conn_status`设置当连接限制发生时，返回响应状态码为429，用以说明请求过多。`limit_conn`和`limit_conn_status`指令是在`http`、`server`和`location`上下文(context)中有效。

注：使用$binary_remote_addr变量， 可以将每条状态记录的大小减少到64个字节，这样1M的内存可以保存大约1万6千个64字节的记录。默认请求下如果限制域的存储空间耗尽了，对于后续所有请求，服务器都会返回 503 (Service Temporarily Unavailable)错误。

### 讨论

基于预定义关健字的限制连接方式，在所有客户端之间公平地共享资源，可以防止请求滥用。因此设置预定义关健字显的格外的重要。前面的示例中，我为使用客户端的IP地址做为预定义关健字，但这样也可能是不合时宜的，因为可能很多不同的客户端处在同一网络，使用同一个IP地址。比如使用了网络地址转换（NAT）的方式。限制IP地址，可能导致整组客户端都被限制。`limit_conn_zone`指令只在http上下文(context)中有效。你可以使用更多数量的可用变量在NGINX http上下文中来构建字符串，来实现连接限制。使用可以在应用程序级别标识用户的变量（例如会话cookie）可能会是一种更干脆简练的解决方案。`limit_conn_status`的默认值为503，表示服务器不可用。但你可能会发现429最好用，因为该服务器这时是可用的，HTTP状态码中，500级状态码响应表示服务器错误，而400级状态码响应表示客户端错误（即是你的客户端超过了我设置的限制，出错方是客户端）。

## 3.6 限制速率

### 问题

你需要通过预定义关健字限制请求的速率，比如根据客户端的IP地址。

### 解答

利用rate-limiting（速率限制）模块来完成请求速率的限制：

```
http {
    limit_req_zone $binary_remote_addr
        zone=limitbyaddr:10m rate=1r/s;
    limit_req_status 429;
    ...
    server {
        ...
        limit_req zone=limitbyaddr burst=10 nodelay;
        ...
    }
}
```

这面的示例配置创建了一个名为`limitbyaddr`的共享内存区域。预定义关健字使用二制进的客端IP地址。共享的内存区域大小为10MB。共享的内存区通过使有一个关键字来设置速率。`limit_req`指令采用两个可选的关键字参数：`zone`和`burst`. `zone`指令指示以什么为基础在共享内存上进行请求限制。当超过给定区域的请求速率时，请求将被延迟，直到达到由`burst`关键字参数表示的峰值，则返回503错误。`burst`关键字参数默认为0。`limit_req`还有第三个可选参数，`nodelay`，这个参数说明未达到`burst`设置的峰值之前可以毫无延时的接受处理客户端请求（达到`burst`峰值就返回429）。`limit_req_status`将返回给客户端的状态设置为特定的HTTP状态代码； `默认值为503。limit_req_status`和`limit_req`在http, server,location的上下文中可以有效被使用。`limit_req_zone`中能在http上下文中使用。速率限制在NGINX Plus中是群集感知的，是R16版本中的新增功能。

注：这里面rate单位是，每秒中允许处理的请求数，这里的速率的每秒1个请求。

### 讨论

限速模块功能强大，可防止快速请求滥用，为所有人提供优质服务。限制请求速率的原因有很多，其中之一就是安全性。你可以通过在登录页面上设置非常严格的限制来拒绝暴力攻击。你可以对所有请求设置合理的限制，从而杜决那些可能试图让你的应用程序宕机或浪费你的服务器资源的恶意用户的计划。限速模块的配置与前面[3.5节](Chapter_03.md#35-限制连接)中提到的连接限制配置非常相似。很多设置点都挺相似。你可以指定每秒请求数或每分钟请求数限制请求的速率。达到速率限制后，将记录该事件，这儿还有一个在上面的示例中没有使用过的指令,`limit_req_log_level`。它默认是errors, 你可以指定为info, notice, 或warn。NGINX Plus中的新增功能，版本R16速率限制现在可感知群集（请参见第[12.5节](Chapter_12.md)，有关区域同步示例）。


### 另请参见

* [Rate Limiting with NGINX and NGINX Plus](https://www.nginx.com/blog/rate-limiting-nginx/)

* 译：[NGINX上的限流](https://www.jianshu.com/p/2cf3d9609af3)

## 3.7 限制带宽

### 问题

你需要限制每个客户端的资源下载带宽。

### 解答

利用NGINX的`limit_rate`和`limit_rate_after`指令来限制对客户端的响应速度:

```
location /download/ {
    limit_rate_after 10m;
    limit_rate 1m;
}
```

这个`location`块的配置指定了带有前缀*download*的uri, 这个配置指定了下载超过10m的带宽后，将限制下载速度为每秒1m。带宽限制是针对每个连接的，所以您可能需要在适用的情况下设置连接限制和带宽限制。

### 讨论

限制特定连接的带宽使NGINX能够以你指定的方式在所有客户机之间共享其上传带宽。这两个指令可以完成所有的工作:`limit_rate_after`和`limit_rate`。`limit_rate_after`可以在`http`, `server`, `location`的上下文中被使用，以后`if`或位于`location`上下文中的`if`中使用，`limit_rate`的适用范围同它一样。但是，还可以通过设置一个名为`$limit_rate`的变量来设置它。`limit_rate_after`用于设置http请求传输多少字节后开始限速.`limit_rate`指令默认指定给定上下文的速率限制以每秒字节数`b`为单位。但是，您可以将`m`指定为兆字节，将`g`指定为千兆字节。两个指令的默认值都是0。0表示没有下载速度限制。此模块还允许您以编程方式来更改客户端的速度限制。