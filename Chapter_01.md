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

## 1.5 重要的文件，命令和目录

### 问题

你需要理解几个重要的NGINX命令和文件目录。

### 解答

#### NGINX 的文件和目录

`/etc/nginx/`

> `/etc/nginx/`目录是NGINX服务器的默认配置根目录，在这个目录中，你将会找到配置
> 指导NGINX如何运行操作的文件。

`/etc/nginx/nginx.conf`

> The /etc/nginx/nginx.conf file is the default configuration entry
point used by the NGINX service. This configuration file sets up
global settings for things like worker process, tuning, logging,
loading dynamic modules, and references to other NGINX configuration
files. In a default configuration, the /etc/nginx/
nginx.conf file includes the top-level http block, which includes
all configuration files in the directory described next

`/etc/nginx/conf.d/`

> The /etc/nginx/conf.d/ directory contains the default HTTP
server configuration file. Files in this directory ending in .conf
are included in the top-level http block from within the /etc/
nginx/nginx.conf file. It’s best practice to utilize include statements
and organize your configuration in this way to keep your
configuration files concise. In some package repositories, this
folder is named sites-enabled, and configuration files are linked
from a folder named site-available; this convention is deprecated.

`/var/log/nginx/`

> The /var/log/nginx/ directory is the default log location for
NGINX. Within this directory you will find an access.log file and
an error.log file. The access log contains an entry for each
request NGINX serves. The error log file contains error events
and debug information if the debug module is enabled.

#### NGINX 的命令

`nginx -h`

> Shows the NGINX help menu.

`nginx -v`

> Shows the NGINX version.

`nginx -V`
> Shows the NGINX version, build information, and configuration
arguments, which shows the modules built in to the
NGINX binary.

`nginx -t`

>Tests the NGINX configuration.

`nginx -T`

> Tests the NGINX configuration and prints the validated configuration
to the screen. This command is useful when seeking
support.

`nginx -s signal`

> The -s flag sends a signal to the NGINX master process. You
can send signals such as stop, quit, reload, and reopen. The
stop signal discontinues the NGINX process immediately. The
quit signal stops the NGINX process after it finishes processing
inflight requests. The reload signal reloads the configuration.
The reopen signal instructs NGINX to reopen log files.

### 讨论

With an understanding of these key files, directories, and commands,
you’re in a good position to start working with NGINX.
With this knowledge, you can alter the default configuration files
and test your changes by using the nginx -t command. If your test is successful, you also know how to instruct NGINX to reload its
configuration using the nginx -s reload command.