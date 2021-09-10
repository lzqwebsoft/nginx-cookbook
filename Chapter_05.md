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

## 5.5 使用Ansible进行安装

### 问题

你需要使用Ansible作为代码来安装、配置和管理NGINX，并与你的其他Ansible配置保持一致。

### 解答

创建一个Ansible playbook来安装NGNIX和管理*nginx.conf*文件。以下是playbook安装NGINX的示例任务文件。确保它的运行并模板化配置文件：

```yaml
- name: NGINX | Installing NGINX
  package: name=nginx state=present

- name: NGINX | Starting NGINX
  service:
    name: nginx
    state: started
    enabled: yes

- name: Copy nginx configuration in place.
  template:
    src: nginx.conf.j2
    dest: "/etc/nginx/nginx.conf"
    owner: root
    group: root
    mode: 0644
  notify:
    - reload nginx
```
这里的`package`块安装NGINX，`service`块确保NGINX在系统启动时被启动和启用。`template`块模板一个*Jinja2*文件，并将结果放在<i>/etc/nginx.conf</i>文件中，其所有者和用户组为root。`template` 块还将对应的文件权限属性设置为`644`并通知 `nginx`服务重新`reload`。同样在这里模板文件没有被包含列出来。然而，安装默认的 NGINX 配置文件可能很简单，但如果使用 Jinja2 模板语言循环和变量替换则变的非常复杂。

### 讨论

Ansible 是一款基于 Python 的应用广泛且功能强大的配置管理工具。任务配置文件使用的是YAML格式，同样你可以使用Jinja2模板语言编写模板文件。Ansible 在订阅模型上提供了一个名为 Ansible Tower 的 master。它通常用于从本地机器或直接向客户端或在无主模型中构建服务器，Ansible 可以批量 SSH 连接到其服务器并运行配置。与其他配置管理工具非常相似，Ansible也有一个庞大的公共角色社区。Ansible 将其称之为 Ansible Galaxy。 你可以在你的playbooks中找到非常复杂的角色。

### 另请参见

[Ansible Documentation](https://docs.ansible.com/)

[Ansible Packages](https://docs.ansible.com/ansible/package_module.html)

[Ansible Service](https://docs.ansible.com/ansible/service_module.html)

[Ansible Template](https://docs.ansible.com/ansible/template_module.html)

[Ansible Galaxy](https://galaxy.ansible.com/)


## 5.6 使用SaltStack进行安装

### 问题

你需要使用SaltStack作为代码来安装、配置和管理NGINX，并与你的其他SaltStack配置保持一致。

### 解答

通过包管理模块安装NGINX，并管理您需要的配置文件。下面是一个示例状态文件(sls)，它将安装`nginx`包，并确保服务正在运行，在系统重启时启用，并在配置文件更改时重新`reload`:

```yaml
nginx:
  pkg:
    - installed
  service:
    - name: nginx
    - running
    - enable: True
    - reload: True
    - watch:
      - file: /etc/nginx/nginx.conf
/etc/nginx/nginx.conf:
  file:
    - managed
    - source: salt://path/to/nginx.conf
    - user: root
    - group: root
    - template: jinja
    - mode: 644
    - require:
      - pkg: nginx
```
这是一个通过包管理工具安装NGINX和管理*ningx.conf*文件的基础示例。NGINX包将会被安装，并且对应的service服务运行，并在系统重启时自动运行。使用SaltStack，你可以声明一个由Salt管理的文件，如示例中所示，并通过许多不同的模板语言进行模板化。同样在这里模板文件没有被包含列出来。然而，安装默认的 NGINX 配置文件可能很简单，但如果使用 Jinja2 模板语言循环和变量替换则变的非常复杂。因为`require`语句，这个配置还指定了NGINX必须在管理文件之前安装。文件就位后，NGINX 会因为`watch` 指令重新加载
服务，在这里是reload而不是restart，是因为 `reload` 指令设置为 `True`。

### 讨论

SaltStack 是一个强大的配置管理工具，它使用YAML文件定义服务器状态。SaltStack的模块可以使用Python编写。Salt 公开了用于状态和文件的 Jinja2 模板语言。但是，对于文件，还有许多其他选项，例如Mako、Python 本身等。Salt可以在主从配置以及无主配置中工作。从服务器被称为minions。然而，主从传输通信与其他传输通信不同，这并使得SaltStack与众不同。使用 Salt，您可以选择 ZeroMQ、TCP 或可靠异步事件传输 (Reliable Asynchronous Event Transport(RAET)) 来传输到 Salt 代理，或者你不能使用代理，master 可以使用 SSH 代替。因为传输层在默认情况下是异步的，所以构建SaltStack时能够将其消息传递给大量负载较低的主服务器。

### 另请参见

[SaltStack](https://docs.saltproject.io/en/latest/)

[Installed Packages](https://docs.saltproject.io/en/latest/ref/states/all/salt.states.pkg.html#salt.states.pkg.installed)

[Managed Files](https://docs.saltproject.io/en/latest/ref/states/all/salt.states.file.html#salt.states.file.managed)

[Templating with Jinja](https://docs.saltproject.io/en/latest/topics/jinja/index.html)

## 5.7 使用 Consul 模板进行自动化配置

### 问题

你需要通过使用Consul来自动化你的NGINX配置，以响应你环境中的变化。

### 解答

使用 `consul-template` 守护程序和模板文件来模板化你选择的 NGINX 配置文件：

```
upstream backend { {{range service "app.backend"}}
    server {{.Address}};{{end}}
}
```

这个例子是一个 Consul 模板文件，它模板化了一个`upstream`配置块。 该模板将循环遍历 Consul 中定义为`app.backend`的节点。对于 Consul 中的每个节点，模板将生成一个带有该节点 IP 地址的服务器指令

`consul-template`守护进程通过命令行运行，可以在每次配置文件模板化有变化时重新加载NGINX:

```
# consul-template -consul consul.example.internal -template \
  template:/etc/nginx/conf.d/upstream.conf:"nginx -s reload"
```

此命令指示 `consul-template` 守护进程连接到 `consul.example.internal` 处的 Consul 集群，并使用当前工作目录中名为template的文件对该文件进行模板化处理，并将生成的内容输出到<i>/etc/nginx/conf.d/
upstream.conf</i>，然后在每次模板文件更改时重新加载NGINX。 这里的`-template`选项接受模板文件的字符串、输出位置和模板过程发生后要运行的命令字符串；这三个变量用冒号":"隔开。如果正在运行的命令有空格，请确保将其用双引号括起来。`-consul` 选项告诉守护进程连接到哪一个 Consul 集群。

### 讨论

Consul 是一个强大的服务发现工具和配置存储。Consul以类似目录的结构存储关于节点和键值对的信息，并允许使用restful API进行交互。Consul 还为每个客户端提供了一个 DNS 接口，允许对连接到集群的节点进行域名查找。每一个使用 Consul 集群的项目被分割为一个 `consul-template` 守护进程；这个工具模板化文件以响应 Consul 节点、服务或键值对的变化。这使得 Consul 成为自动化 NGINX 的一个强有力的选择。使用 `consul-template` 你还可以指示守护程序在模板发生更改后运行特定命令。通过这，我们可以重新加载NGINX配置，并使你的NGINX配置随之生效。使用 Consul，您可以对每个客户端进行健康检查，以检查预期服务的健康状况。有了这种故障检测，你就能够相应地将NGINX配置模板化，只将流量发送到健康的主机。

### 另请参见

[Consul Home Page](https://www.consul.io/)

[Introduction to Consul Template](https://www.hashicorp.com/blog/introducing-consul-template)

[Consul Template GitHub](https://github.com/hashicorp/consul-template)