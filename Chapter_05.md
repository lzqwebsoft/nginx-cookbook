# 第五章 可编程性与自动化

## 5.0 介绍

可编程性是指通过编程与事物进行交互的能力。NGINX Plus的API提供了这样的功能：通过HTTP接口与NGINX Plus的配置和行为进行交互的能力。这个API提供了通过HTTP请求添加或删除上游服务器来重新配置NGINX Plus的能力。NGINX Plus的键值存储特性提供了另一层动态配置——你可以利用HTTP调用注入信息，NGINX Plus可以动态路由或控制流量。本章将介绍NGINX Plus API和该API公开的键值存储模块。

配置管理工具自动化了服务器的安装和配置，这在云计算时代是一个非常宝贵的实用工具。大型web应用程序的工程师不再需要手动配置服务器；与之对应的，他们可以使用其中的许多可用的配置管理工具。借助这些工具，工程师可以一次性编写配置和代码，以可重复、可测试和模块化的方式生成具有相同配置的多个服务器。本章介绍了一些最流行的配置管理工具，以及如何使用它们来安装NGINX和模块化基础配置。这些示例非常基础，但是演示了如何从相应的平台开始使用NGINX服务器。

## 5.1 NGINX Plus API

### 问题

你有一个动态环境，需要动态重新配置NGINX Plus。

### 解答

配置NGINX Plus API，通过API调用来添加和删除服务器:

```
upstream backend {
    zone http_backend 64k;
}
server {
    # ...
    location /api {
        api [write=on];
        # 限制访问API的指令
        # 请查看第七章
    }
    location = /dashboard.html {
       root /usr/share/nginx/html;
    }
}
```
这个NGINX Plus配置创建了一个具有共享内存区的上游服务器，在`/api`的`location`块中启用了API，并为NGINX Plus仪表盘dashboard提供了`location`。

当服务器上线时，你可以利用API添加服务器:

```bash
$ curl -X POST -d '{"server":"172.17.0.3"}' \
  'http://nginx.local/api/3/http/upstreams/backend/servers/'
```

执行结果：

```json
{
    "id":0,
    "server":"172.17.0.3:80",
    "weight":1,
    "max_conns":0,
    "max_fails":1,
    "fail_timeout":"10s",
    "slow_start":"0s",
    "route":"",
    "backup":false,
    "down":false
}
```
此示例中的`curl`命令向NGINX Plus发出请求，以将新服务器添加到后端上游配置。采用的是HTTP POST方法，并且将JSON对象作为正文参数传递。NGINX Plus API采用的是RESTful API模式；因此带参数的请求，URL采用如下方式请求：

```
/api/{version}/http/upstreams/{httpUpstreamName}/servers/
```

你可以利用NGINX Plus API列出上游池中的服务器：

```bash
$ curl 'http://nginx.local/api/3/http/upstreams/backend/servers/'
```

执行结果：

```json
[
    {
        "id":0,
        "server":"172.17.0.3:80",
        "weight":1,
        "max_conns":0,
        "max_fails":1,
        "fail_timeout":"10s",
        "slow_start":"0s",
        "route":"",
        "backup":false,
        "down":false
    }
]
```

在此示例中，`curl`命令向NGINX Plus发出请求，以列出上游池中名为`backend`的所有服务器。当前，只返回得到一个服务器，这还是上一个`curl`命令通过添加服务器API添加的。该请求将返回一个上游服务器对象，该对象包含服务器的所有可配置选项。

使用NGINX Plus API排空上游服务器的连接，为从上游池中正常删除做好准备。你可以在[2.8章节](Chapter_02.md#28-连接排空connection-draining)中找到有关连接排空的详细信息：

```bash
$ curl -X PATCH -d '{"drain":true}' \
  'http://nginx.local/api/3/http/upstreams/backend/servers/0'
```

执行结果：

```json
{
    "id":0,
    "server":"172.17.0.3:80",
    "weight":1,
    "max_conns":0,
    "max_fails":1,
    "fail_timeout":"10s",
    "slow_start":"0s",
    "route":"",
    "backup":false,
    "down":false,
    "drain":true
}
```

在此`curl`命令中，我们指定请求方法为`PATCH`，传递一个JSON主体以指示它排空服务器的连接，并通过将服务器ID附加到URI中来指定对应服务器。对应的服务器的ID，我们可以通过上面的`curl`命令中列出上游池中的服务器的示例方法中找到。

NGINX Plus将开始耗尽连接。该过程可能需要花费与应用程序session会话相同的时间。要检查已开始排空的服务器正在服务的活动连接数，可以使用以下调用查找正在消耗的服务器的active属性：

```bash
$ curl 'http://nginx.local/api/3/http/upstreams/backend'
```

执行结果：

```json
{
    "zone" : "http_backend",
    "keepalive" : 0,
    "peers" : [
        {
            "backup" : false,
            "id" : 0,
            "unavail" : 0,
            "name" : "172.17.0.3",
            "requests" : 0,
            "received" : 0,
            "state" : "draining",
            "server" : "172.17.0.3:80",
            "active" : 0,
            "weight" : 1,
            "fails" : 0,
            "sent" : 0,
            "responses" : {
                "4xx" : 0,
                "total" : 0,
                "3xx" : 0,
                "5xx" : 0,
                "2xx" : 0,
                "1xx" : 0
            },
            "health_checks" : {
                "checks" : 0,
                "unhealthy" : 0,
                "fails" : 0
            },
            "downtime" : 0
        }
    ],
    "zombies" : 0
}
```

在所有连接排空之后，请使用NGINX Plus API从上游池中完全删除服务器：

```bash
$ curl -X DELETE \
  'http://nginx.local/api/3/http/upstreams/backend/servers/0'
```

执行结果:

```json
[]
```

上面的`curl`命令与更新服务器状态的URI相同，只是请求发送的是`DELETE`方法。`DELETE`方法指示NGINX删除服务器。这个API调用返回池中仍然保留的所有服务器及其ID。因为我们刚开始时就是一个空池，通过API只添加了一个服务器，接着排空了它，最后又删除了它，所以现在又变成了一个空池。所以返回的是一个空列表。

### 讨论

NGINX Plus专有API使动态应用程序服务器可以动态地将自身在NGINX配置中添加或删除。当服务器是上线状态时，它们可以注册到上游池中，NGINX则将开始发送负载。当服务器需要删除时，服务器可以请求NGINX Plus排空连接，然后在关闭之前将自己从上游池中删除。这使得基础架构可以通过一些自动化来进行扩展，而无需人工干预。

### 另请参见

[NGINX Plus API Swagger Documentation](https://demo.nginx.com/swagger-ui/)

## 5.2 键值存储

### 问题

你需要NGINX Plus根据应用程序的输入做出动态流量管理决策。

### 解答

设置群集可感知的键值存储和API，然后添加键和值:

```
keyval_zone zone=blacklist:1M;
keyval $remote_addr $num_failures zone=blacklist;
server {
    # ...
    location / {
        if ($num_failures) {
            return 403 'Forbidden';
        }
        return 200 'OK';
    }
}
server {
    # ...
    # 限制访问API的指令
    # 请查看第6章
    location /api {
        api write=on;
    }
}
```
上面的NGINX Plus配置使用`keyval_zone`指令构建了一个名为`blacklist`的键值存储共享内存区，并配置限制该内存区为1MB大小。然后，`keyval`指令将匹配第一个参数`$remote_addr`的键值映射到区域中名为`$num_failure`的新变量。然后，使用这个新变量来确定NGINX Plus是否为请求提供服务还是返回403禁止代码。

使用此配置启动NGINX Plus服务器后，你可以使用`curl`对本地机器进行请求，并期望收到200 OK响应。

```bash
$ curl 'http://127.0.0.1/'
OK
```

现在将本地机器的IP地址添加到值为1的键值存储中:

```bash
$ curl -X POST -d '{"127.0.0.1":"1"}' \
  'http://127.0.0.1/api/3/http/keyvals/blacklist'
```

这个`curl`命令使用一个JSON对象提交一个HTTP POST请求，该JSON对象包含一个要提交到blacklist共享内存区域的键值对象。键值存储API URI的格式如下:

```
/api/{version}/http/keyvals/{httpKeyvalZoneName}
```

本地机器的IP地址现在被添加到名为blacklist的键值区域，值为1。在接下来的请求中，NGINX Plus在键值区域中查找$remote_addr，找到条目，并将该值映射到变量$num_failure。然后在if语句中对该变量求值。当变量有值而不是空值或值，则if的计算得到为True，则NGINX Plus返回403 Forbidden状态码：

```bash
$ curl 'http://127.0.0.1/'
Forbidden
```

你可以通过发出`PATCH`方法请求来更新或删除键值：

```bash
$ curl -X PATCH -d '{"127.0.0.1":null}' \
  'http://127.0.0.1/api/3/http/keyvals/blacklist'
```

如果该值为`null`，NGINX Plus则将删除这个键，最后上面的请求将再次返回200 OK。

### 讨论

键值存储是NGINX Plus的专有功能，这使得应用程序能够将信息注入NGINX Plus。在上面提供的示例中，`$remote_addr`变量用于创建动态黑名单。你可以使用NGINX Plus提供的任意变量来构建键值存储（例如，一个session cookie），并为NGINX Plus提供外部值。在NGINX Plus R16版本中，键值存储是集群感知的，这意味着你只需要向一台NGINX Plus服务器提供键值更新，它们都将收到信息。

### 另请参见

[动态带宽限制(英文)](https://www.nginx.com/blog/dynamic-bandwidth-limits-nginx-plus-key-value-store/)

## 5.3 使用Puppet进行安装

### 问题

你需要使用Puppet作为代码来安装、配置和管理NGINX，并与你的其他Puppet配置保持一致。

### 解答

创建一个安装 NGINX 的模块，管理你需要的文件，并确保 NGINX 正在运行：

```
class nginx {
    package {"nginx": ensure => 'installed',}
    service {"nginx":
        ensure => 'true',
        hasrestart => 'true',
        restart => '/etc/init.d/nginx reload',
    }
    file { "nginx.conf":
        path => '/etc/nginx/nginx.conf',
        require => Package['nginx'],
        notify => Service['nginx'],
        content => template('nginx/templates/nginx.conf.erb'),
        user=>'root',
        group=>'root',
        mode='0644';
    }
}
```
这个模块很实用的利用包管理来确保NGINX的安装。它确保系统在启动时自动运行NGINX。这个配置指出Puppet服务有一个名为`hasrestart` 的重启指令。并且我们可以使用NGINX的reload重载来覆盖`restart`命令。在这里文件资源通过嵌入式Ruby模板语言（Embedded Ruby (ERB)）来组织和管理*nginx.conf*文件。因为使用了`require`指令，模板文件将在NGINX包被安装后生成。接着，因为`notify`指令，文件资源会通知NGINX服务重新加载。在这里模板文件没有被包含列出来。然而，安装默认的 NGINX 配置文件可能很简单，但如果使用 ERB 或 EPP 模板语言循环和变量替换则变的非常复杂。

### 讨论

Puppet 是一个基于 Ruby 编程语言的配置管理工具。以特定的域语言构建模块，通过调用一个清单文件来定义服务器配置。Puppet可以在主-从或无主配置中运行。使用Puppet，清单在主服务器上运行，然后发送给从服务器。这是很重要的，因为这确保从服务器仅专注于接受为其指定的配置，而不会为其他服务器提供另外的配置。Puppet还有很多非常高级的公共模块。这些模块使你的配置工作能够更加快速的开始。一个来自GitHub上的voxpupuli的公共NGINX模块将会为你模板化出NGINX配置。

### 另请参见

[Puppet Documentation](https://puppet.com/docs/)

[Puppet Package Documentation](https://puppet.com/docs/puppet/7/types/package.html)

[Puppet Service Documentation](https://puppet.com/docs/puppet/7/types/service.html)

[Puppet File Documentation](https://puppet.com/docs/puppet/7/types/file.html)

[Puppet Templating Documentation](https://puppet.com/docs/puppet/7/lang_template.html)

[Voxpupuli NGINX Module](https://github.com/voxpupuli/puppet-nginx)

## 5.4 使用Chef进行安装

### 问题

你需要使用Chef作为代码来安装、配置和管理NGINX，并与你的其他Chef配置保持一致。

### 解答

创建一个带有配方的食谱，以安装 NGINX 并通过模板配置配置文件，并确保在配置到位后重新加载 NGINX。 以下是配方示例：

```
package 'nginx' do
    action :install
end

service 'nginx' do
    supports :status => true, :restart => true, :reload => true
    action [ :start, :enable ]
end

template 'nginx.conf' do
    path "/etc/nginx.conf"
    source "nginx.conf.erb"
    owner 'root'
    group 'root'
    mode '0644'
    notifies :reload, 'service[nginx]', :delayed
end
```

这里的`package`块安装NGINX，`service`块确保NGINX在系统启动时被启动和启用，然后向Chef的其他部分声明`nginx`服务将支持哪些操作。`template`块模板了一个ERB文件，并把它放在<i>/etc/nginx.conf</i>文件中，文件的所有者和组为root。模板块还会将`mode`(Linux文件权限)设置为`644`，并通知`nginx`服务重新`reload`，但得等到`:delayed`语句声明的 Chef 运行结束。同样在这里模板文件没有被包含列出来。然而，安装默认的 NGINX 配置文件可能很简单，但如果使用 ERB 模板语言循环和变量替换则变的非常复杂。

### 讨论

Chef是一个基于Ruby的配置管理工具。Chef可以在主从配置或单独配置中运行，现在称为Chef Zero。Chef有一个非常大的社区，里面有很多公开的实战示例，叫做超级市场。可以通过名为 Berkshelf 的命令行实用程序来安装和维护来自超级市场公开的实战示例。Chef非常能干，我们展示的只是一个小样本。超级市场中的公开的NGINX实战示例非常灵活，提供了从包管理器或源代码轻松安装NGINX的选项，以及编译和安装许多不同模块以及模板化基本配置的能力。

### 另请参见

[Chef documentation](https://docs.chef.io/)

[Chef Package](https://docs.chef.io/resources/package/)

[Chef Service](https://docs.chef.io/resources/service/)

[Chef Template](https://docs.chef.io/resources/template/)

[Chef Supermarket for NGINX](https://supermarket.chef.io/cookbooks/nginx)