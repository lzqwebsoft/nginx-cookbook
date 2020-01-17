# 第三章 网络流量管理

## 3.0 介绍

NGINX和NGINX Plus也被归类为网络流量控制器。 您可以使用NGINX使用许多属性智能分配请求路由流量和控制请求流程(flow)。本章介绍NGINX能够基于百分比分配客户端请求，利用客户端的地理位置，以及根据速率，连接和带宽限制的形式控制请求流量的功能。 在阅读本章时，请记住，可以混合使用这些功能，以实现无数种可能性。

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
`split_clients`指令对你提供的字符串作为第一个参数进行MurmurHash2哈希计算处理，得到一个32位的hash值。并且将hash值按百分比提供映射给一个变量的值作为第二个参数提供。第三个参数是包含键值对的对象，其中键表示权重百分比，值表示要被分配的值。键可以是百分比或星号。星号表示取所有百分比后的剩余部分。$variant变量的值为客户端IP地址计算hash的20%定向为backendv2，剩余的80%定向为backendv1。

在上面的示例中，backendv1和backendv2表示上游服务器池，可以与`proxy_pass`指令结合一起使用，如下所示：

```
location / {
    proxy_pass http://$variant
}
```

使用变量$variant，我们将网络流量请求按不同的比例分配给两个不同的应用程序服务器池（application server pools即upstream pool）。

### 讨论

当电子商务网站上需要测试不同类型的营销和前端功能的转换率时，这种类型的A/B测试非常有用。应用程序通常使用一种称为[金丝雀发布(canary release)](https://baike.baidu.com/item/%E7%81%B0%E5%BA%A6%E5%8F%91%E5%B8%83)的部署类型。在这种类型的部署中，流量会缓慢地切换到新版本。在推出新版本的代码时，在不同版本的应用程序之间拆分客户端请求可能很有用，以限制发生错误的影响范围。不管在两个不同的应用程序集之间拆分客户端请求的原因是什么，NGINX都通过使用`split_clients`模块简化了这一过程。

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