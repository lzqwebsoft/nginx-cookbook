# 第一章 基础

## 1.0 介绍

要开始使用开源的NGINX或NGINX Plus，首先需要将其安装在系统上并学习一些基础知识。 这章你将会学到如何安装NGINX, 主要配置文件在哪里，以及管理命令。你还将学会如何验证安装并向默认服务器发出Request请求。

## 1.1 在 Debian/Ubuntu 下安装

### 问题

你需要将开源的NGINX服务器安装在一台Debian或Ubuntu的机器上。

### 解答

创建一个名为`/etc/apt/sources.list.d/nginx.list`的文件，并包含如下内容:

```
deb http://nginx.org/packages/mainline/OS/ CODENAME nginx
deb-src http://nginx.org/packages/mainline/OS/ CODENAME nginx
```

根据你的系统的发行版名称，替换URL后面的**OS**为`ubuntu`或`debian`。并且替换**CODENAME**为你的发行版代号，例如：Debian系统的`jessie`或`stretch`，或Ubuntu系统中的:`trusty`, `xenial`, `artful`, 或`bionic`。替换完成后，执行下面的命令：

```bash
wget http://nginx.org/keys/nginx_signing.key
apt-key add nginx_signing.key
apt-get update
apt-get install -y nginx
/etc/init.d/nginx start
```

### 讨论

你刚刚创建的文件指示apt软件包管理系统使用NGINX官方的软件包仓库。后面的命令下载NGINX GPG软件包签名密钥并将其导入apt。提供apt签名密钥可使apt系统验证来自软件仓库的软件包。`apt-get update`命令指示apt系统从其已知存储库刷新其软件包列表。软件包列表刷新后，你就可以从官方的NGINX软件包仓库拉取安装开源的NGINX服务器了。当你安装完成后，最后一条命令用于启动NGINX。

## 1.2 在 RedHat/CentOS 下安装

### 问题

你需要在RedHat或CentOS上安装开源的NGINX。

### 解答

创建名为`/etc/yum.repos.d/nginx.repo`的文件，并包含如下内容：

```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/OS/OSRELEASE/$basearch/
gpgcheck=0
enabled=1
```

同理，根据你的系统发行版本名称替换URL后面中的**OS**为`rhel`或`centos`。同时分别根据你系统的版本如6.x或7.x，替换**OSRELEASE**为`6`或`7`。接着执行如下命令:

```bash
yum -y install nginx
systemctl enable nginx
systemctl start nginx
firewall-cmd --permanent --zone=public --add-port=80/tcp
firewall-cmd --reload
```

### 讨论

你刚刚创建的文件指示yum软件包管理系统使用NGINX官方的软件包仓库。接下来的命令使用yum从NGINX开源仓库中拉取并安装开源的NGINX服务器，指示`systemd`开机启动NGINX服务, 并且现在就启动运行NGINX服务。接下来的`firewall`命令控制防火墙开放TCP协议访问80端口，因为这是HTTP服务的默认端口. 最后的reload命令重载`firewall`防火墙以提交防火墙规则变更.

## 1.3 安装 NGINX Plus

### 问题

你需要安装NGINX Plus

### 解答

访问[http://cs.nginx.com/repo_setup](http://cs.nginx.com/repo_setup).从下拉菜单中选择你要安装的操作系统，然后按照说明进行操作. 说明类似于开源NGINX的安装；但是，您需要安装证书才能进行身份验证到NGINX Plus软件仓库。

### 讨论

NGINX会维护该软件仓库的软件保持最新，并提供有关安装NGINX Plus的说明。根据你的操作系统和版本，这些说明略有不同，但是有一个共同点：
你必须登录NGINX的网站才能下载证书和密钥key，以确保你的系统可以验证到NGINX Plus软件仓库。

## 1.4 验证安装

### 问题

你需要验证你的安装并检查安装的版本。

### 解答

验证你的安装并检查安装的版本，你可以使用如下命令：

```bash
$ nginx -v
nginx version: nginx/1.15.3
```

如上所示，输出对应NGINX版本。

你可以使用如下命令查检NGINX是否正在运行：

```bash
$ ps -ef | grep nginx
root 1738 1 0 19:54 ? 00:00:00 nginx: master process
nginx 1739 1738 0 19:54 ? 00:00:00 nginx: worker process
```
`ps`命令列出了正在运行的进程。通过管道传输到`grep`，你可以在输出中查询搜索指定的关键词。这个示例中使用`grep`搜索`nginx`。结果显示了两个正在运行的`master`和`worker`进程。如果`nginx`正在运行，你将会看到一个`master`进程或至少一个`worker`进程。有关NGINX的启动操作详情，比如使用`init.d`或`systemd`方式，如何将NGINX作为守护程序(daemon)启动，请参阅下一节。

要验证NGINX是否正确响应Request请求，用你的机器上面的浏览器发出请求或使用`curl`命令：

```bash
$ curl localhost
```

你将会看到NGINX使用的默认HTML欢迎页面。

### 讨论

可以直接使用`nginx`二进制NGINX命令工具来检查安装的版本，列出安装的modules模块，测试配置正确和向主master进程发送信号。NGINX必须正在运行才能服务响应request请求。确定NGINX是作为守护程序(daemon)运行，还是在前端运行，使用`ps`命令是种有效的命令手段。NGINX默认的配置确保其做为一个在80端口运行的静态HTTP站点服务器。你可以在机器上通过发送URL地址为`localhost`的HTTP request请求，或使用主机的IP地址或主机名来测试访问这个默认的站点。

## 1.5 关键的文件，命令和目录

### 问题

你需要理解的几个关键的NGINX命令和目录文件。

### 解答

#### NGINX 的文件和目录

`/etc/nginx/`

> `/etc/nginx/`目录是NGINX服务器的默认配置根目录，在这个目录中，你将会找到配置
> 指导NGINX如何运行操作的文件。

`/etc/nginx/nginx.conf`

> `/etc/nginx/nginx.conf`是NGINX服务使用的默认入口配置文件，这个配置文件为诸如工作进程，调优，日志记录，加载动态模块以及对其他NGINX配置文件的引用之类的设置设置全局设置。在默认配置中，`/etc/nginx/nginx.conf`文件包含顶级**http**块. 并且包含include的所有配置文件的目录的描述。

`/etc/nginx/conf.d/`

> `/etc/nginx/conf.d/`目录包含了默认的HTTP配置的文件.该目录中以.conf结尾的文件
包含在`/etc/nginx/nginx.conf`文件中的顶级**http**块中.最佳做法是利用include语句并以此方式组织你的配置，以使配置文件保持简洁。在其他的软件库中，这个文件夹的名字是`sites-enabled`，并且其中的配置文件是link链接的名为`site-available`目录中的配置文件，这样的约束现在已不再推荐。

`/var/log/nginx/`

> `/var/log/nginx/`目录是默认的NGINX日志目录。在里面你可以找到一个 `access.log` 和一个 `error.log` 文件。 `access.log`日志文件包含每个请求访问
NGINX服务的条目. `error.log`日志文件包含错误事件和调试信息（如果启用了调试模块）。

#### NGINX 命令

`nginx -h`

> 显示NGINX的使用帮助信息

`nginx -v`

> 显示NGINX的版本

`nginx -V`

> 显示NGINX版本，构建信息和配置参数，显示NGINX二进制文件中构建内置的模块。

`nginx -t`

> 测试NGINX的配置正确性。

`nginx -T`

> 测试NGINX配置并打印经过验证的配置到屏幕。寻求支持时，此命令很有用。

`nginx -s signal`

> **-s** 标志向NGINX 主（master）进程发送信号(signal)。你可以发送的信号(signal)为：`stop`, `quit`, `reload`和`reopen`。`stop`信号用于立即中断停止NGINX进程 。`quit`用于在处理完机上请求后停止NGINX进程。`reload`信号用于重新加载NGINX配置。`reopen`信号指示NGINX重新打开日志文件。

### 讨论

了解了这些关键的文件，目录和命令后，你将可以开始使用NGINX。有了这些知识，你可以更改默认配置文件并使用`nginx -t`命令测试你的更改。如果测试成功，你还可以使用`nginx -s reload`命令指示NGINX重新加载修改后的配置。

## 1.6 静态内容服务

### 问题

使用NGINX提供静态内容服务。

### 解答

覆盖`/etc/nginx/conf.d/default.conf`下默认的HTTP服务配置，使用如下配置示例：

```
server {
    listen 80 default_server;
    server_name www.example.com;
    location / {
        root /usr/share/nginx/html;
        # alias /usr/share/nginx/html;
        index index.html index.htm;
    }
}
```

### 讨论

这个配置指定从`/usr/share/nginx/html`目录上通过80端口上提供HTTP静态文件服务。这个配置的第一行定义了一个新的**server**块，这为NGINX定义了一个新的监听上下文。第二行指定NGINX监听80端口，参数`default_server`指定NGINX将此服务器用作端口80的默认上下文。`server_name`指令定义request请求到达到该服务器的主机名或名称。如果配置未将上下文定义为`default_server`,则NGINX只有当接收的HTTP请求头的值与`server_name`提供的匹配时才将请求定向到该服务器。

**location**块定义根据URL中的路径定义配置。路径或域后面URL的一部分，称为URI。NGINX将最好地将请求的URI匹配到`localtion`块。上面的例子中`/`匹配所有的请求。`root`指令显示了NGINX在为给定上下文提供内容时在哪里寻找静态文件，查找所请求的文件时，请求的URI会附加到`root`指令的值的后面。如果我们为`location`指令提供了URI前缀。除非我们使用别名`alias`目录而不是`root`，否则它将包含在附加路径中。最后，`index`指令为NGINX提供了一个默认文件，或者一个要检查的文件列表，以防URI中未提供其他路径。

## 1.7 优雅的Reload

### 问题

你需要在不丢失包的情况下重新加载配置。

### 解答

使用NGINX的reload方法，在不停止服务器的情况下，实现配置的优美reload:

```bash
$ nginx -s reload
```

上面的示例中，使用NGINX二进制工具命令向NGINX主（master）进程发送reload信号，实现NGINX系统的重新加载。

### 讨论

重新加载NGINX配置而不停止服务器提供即时更改配置的能力，而无需丢弃任何数据包。在高运行时间的动态环境中，您将需要在某些时候更改负载均衡配置，NGINX允许你在保持负载均衡在线的情况下执行此操作。这个功能提供了许多可能性，比如在运行的实时环境中重新运行配置管理，或者构建支持一个应用程序，集群模块的动态配置和重新加载NGINX以满足环境的需要。