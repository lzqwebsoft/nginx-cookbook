# 第二章 高性能负载均衡

## 2.0 介绍

今天的互联网用户体验需要性能和正常运行时间。为了实现这一点，需要运行同一个系统的多个副本，并将负载分配到它们上面。随着负载的增加，部署系统的另一个副本联机处理请求的这种构架技术称为**水平扩展**。基于软件的基础架构因其灵活性而越来越受欢迎，从而开辟了广阔的可能性。无论用例是小到两个高可用性用例的集合，还是大到全球范围内的数千个用例。需要一种与基础架构一样动态的负载均衡解决方案。NGINX以多种方式满足了这一需求，如HTTP、TCP和UDP负载平衡，我们将在本章中介绍。

在均衡负载时，对客户的影响只是积极的影响，这一点很重要。许多现代web架构采用无状态的应用程序层，将状态存储在共享内存或数据库中，但这并不是所有人都可以实现的。在交互式应用程序中，会话状态非常重要并且使用广泛，出于多种原因，此状态可能存储在应用服务器的本地。比如，在大数据处理的应用程序中，网络开销在性能上太昂贵，因此将会话状态存储在本地应用服务器上对于用户体验来说是非常有必要的。这种情况的另一个方面是，在会话完成之前不应释放服务器。大规模的有状态应用程序需要智能负载均衡器。NGINX Plus通过跟踪cookie或路由提供了多种方法来解决这个问题。本章讨论了与NGINX和NGINX Plus负载均衡相关的会话持久性。

确保NGINX服务的应用程序运行良好同样重要。由于多种原因，应用程序可能宕机。比如由于网络连接性，服务器故障或应用程序故障等。代理和负载均衡器必须足够智能，以检测上游服务器的故障并停止向它们传递流量，不然，客户端将被等待，得到一个超时错误。减轻服务器质量下降的一种方法是让代理检查上游服务器的健康状况。NGINX提供两种不同类型的健康检查：被动式，在开源版本中可用；主动式，只有在NGINX Plus中可用。主动式为定期与上游服务器建立连接或请求进行活动状态检查，并验证响应是否正确。被动为在客户端进行请求或连接时运行状况检查，监视上游服务器的连接或响应。你可能想要使用被动式状态检查来减少上游服务器的负载，并且你可能还想要使用主动式状态检查来确定上游服务器在为客户端提供故障服务之前的故障。本章的最后将讨论如何监视要进行负载均衡的上游应用服务器的运行状态状况。

## 2.1 HTTP负载均衡

### 问题

你需要在两个或多个HTTP服务器之间分配负载。

### 解答

使用NGINX的HTTP模块，通过`upstream`块在HTTP服务器上实现负载均衡：

```
upstream backend {
    server 10.10.12.45:80 weight=1;
    server app.example.com:80 weight=2;
}
server {
    location / {
        proxy_pass http://backend;
    }
}
```
上面的配置均衡了端口80上的两个HTTP服务器之间的负载。`weight`参数指示NGINX将两倍的连接传递给第二个服务器，`weight`参数默认为1。

### 讨论

HTTP `upstream` 模块控制HTTP的负载均衡. 这个模块定义了一个目标池 - Unix套接字，IP地址和DNS记录的任意组合，或混合。`upstream` 模块还定义了如何将任意单独的request请求分配给任何上游服务器。

每个上游目标是通过`server`指令在上游池(upstream pool)中定义的。`server`指令提供了Unix套接字，IP地址或FQDN，以及许多可选参数。可选参数提供了许多对请求路由的控制。这些参数包括平衡算法中服务器的权重（weight）; 服务器是处于待机模式（standby mode），可用（available）还是不可用（unavailable）；以及如何确定服务器是否不可用. NGINX Plus提供了许多其他方便的参数，例如与服务器的连接限制，高级DNS解析控制以及在服务器启动后能够缓慢增加与服务器的连接的能力。

## 2.2 TCP负载均衡

### 问题

你需要在两个或多个TCP服务器之间分配负载。

### 解答

使用NGINX的`stream`模块在TCP服务器上使用`upstream`块进行负载均衡:

```
stream {
    upstream mysql_read {
        server read1.example.com:3306  weight=5;
        server read2.example.com:3306;
        server 10.10.12.34:3306  backup;
    }
    server {
        listen 3306;
        proxy_pass mysql_read;
    }
}
```

本例中的`server`块指示NGINX监听TCP端口3306，并在两个MySQL数据库读副本之间均衡负载，并列出另一个作为备份，如果上面的数据库们宕机则将通过该备份。此配置不能添加到`conf.d`文件夹，因为该文件夹包含在`http`块中; 你应该创建一个名为`stream.conf.d`的文件夹，在`nginx.conf`文件中开启一个新的`stream`块，并将新的流（stream）配置引入到这个文件夹。

### 讨论

TCP负载均衡由NGINX `stream`模块定义。与HTTP模块一样，`stream`模块允许你定义上游服务器池（upstream pools）并配置侦听服务器。在将服务器配置为监听给定端口时，必须定义要监听的端口，或者可选地定义地址和端口。从那里，必须配置一个目的地，不管它是另一个地址的直接反向代理还是上游资源池。

TCP负载平衡的upstream与HTTP的upstream非常相似，因为它将上游(pustream)资源定义为服务器，并使用Unix套接字，IP或完全限定域名（FQDN）进行配置，以及服务器权重，最大连接数， DNS解析器和连接启动周期；以及服务器是处于活动状态，还是处于关闭状态，还是处于备份模式。

NGINX Plus提供了更多用于TCP负载平衡的功能。NGINX Plus中提供的这些高级功能可以在本书中找到。所有负载均衡的运行状况检查将在本章后面介绍。

## 2.3 UDP负载均衡

### 问题

你需要在两个或多个UDP服务器之间分配负载。

### 解答

使用NGINX的`stream`模块进行UDP服务器的负载均衡，并定义名为`ntp`的`upstream`块：

```
stream {
    upstream ntp {
        server ntp1.example.com:123  weight=2;
        server ntp2.example.com:123;
    }
    server {
        listen 123 udp;
        proxy_pass ntp;
    }
}
```

上面这部分配置使用UDP协议均衡两个upstream网络时间协议（NTP）服务器之间的负载，简单的在`listen`指令上加上`udp`参数就说明使用了UDP负载均衡。

如果你的负载均衡服务需要在客户机和服务器之间来回发送多个包，你可以指定`reuseport`参数，像这样类型的服务示例有：OpenVPN，互联网协议语音（VoIP），虚拟桌面解决方案和数据报传输层安全性（DTLS）。下面是一个使用NGINX处理OpenVPN连接并将其代理到本地运行的OpenVPN服务的例子:

```
stream {
    server {
        listen 1195 udp reuseport;
        proxy_pass 127.0.0.1:1194;
    }
}
```

### 讨论

你可能会问：“当DNS A或SRV记录中可以有多个主机时，为什么还需要负载均衡器？”，答案是，因为我们不仅可以使用多种的均衡算法进行均衡负载，而且还可以在DNS服务器本身上进行负载均衡。UDP协议服务构成了我们在网络系统中依赖的许多服务，例如DNS，NTP和VoIP。UDP负载均衡在某些情况下可能不太常见，但在层级化环境中很有用。

你可以在`stream`模块中找到UDP负载均衡配置，和TCP负载均衡很像，它们在配置上几乎一样。主要区别在于`listen`指令指定开放套接字用于数据报。当处理数据报时，还有一些其他指令可能会在TCP中不适用的地方应用，例如`proxy_response`指令，它指定从上游服务器可以向NGINX发送多少个预期的响应。默认情况下，这是无限次的，除非设置了`proxy_timeout`限制。

`reuseport`参数指示NGINX为每个工作进程(worker process)创建单独的监听套接字(socket)。这允许LINUX内核(kernel)在工作进程(worker process)之间分配入站连接，以处理客户端和服务器之间发送的多个包。`reuseport`特性只适用于Linux内核3.9及更高版本、DragonFly BSD和FreeBSD 12及更高版本。

## 2.4 负载均衡的方式

### 问题

轮循（Round-robin）负载均衡不符合你的工作需要，因为你的工作负载或服务器池过于繁杂。

### 解答

换用其他的NGINX的负载均衡方式，如：最少连接(least connections)，最少时间(least time)，通用哈希(generic hash)，IP哈希(IP hash)，或随机(random):

```
upstream backend {
    least_conn;
    server backend.example.com;
    server backend1.example.com;
}
```

上面的示例将后端上游池（upstream pools）的负载均衡算法设置为最少连接（least connections）。除通用哈希，随机和最少时间外，所有负载均衡算法都有独立的指令，像上面的示例。这些参数指令将在下面的讨论中进行详细阐述。

### 讨论

并不是所有请求或数据包都具有相同的权重，鉴于此，轮循（round robin），甚至是前面示例中使用的加权（注：前面的设置weight）轮循，并不能满足所有应用或流量需求。NGINX提供了许多负载均衡算法，用于满足特定需求场景，你可以选择和配置这些不同的负载均衡方法。下列的负载均衡方式可用于HTTP、TCP和UDP的上游池(upstream pool)：

* 轮循(Round robin)

  这是NGINX默认采用的负载均衡方式，它按上游池（upstream pool）中服务器列表的顺序分配请求。你还可以设置`weight`加权的方式实例加权轮询，比如上游服务器的容量发生变化，你就可以使用它。权值的整数值越高，服务器在轮询中就越受青睐。权重背后的算法只是加权平均值的统计概率。

* 最少连接(Least connections)

  这个方式的是通过将请求代理到当前已打开连接数最少的上游服务器中来实现的负载均衡的（注：最小连接数法根据后端服务器当前的连接数情况，动态地选取其中积压连接数最小的一台服务器来处理当前的请求）。最少连接，同轮循铁负载一样，在决定发送连接到哪个服务器时也要考虑权重。最少连接负载均衡方式的指令是`least_conn`。

* 最少时间(Least time)

  该指令仅在NGINX Plus中可用。最少的时间类似于最少的连接，最少的连接它代理具有最少当前连接数的上游服务器，最少的时间则倾向于具有最低平均响应时间的服务器。此方式是最复杂的负载平衡算法之一，适合高性能Web应用程序的需求。该算法优于最少连接的实现方式，因为连接数量少并不一定意味着响应速度最快。必须为此指令指定`header`或`last_byte`的参数。指定`header`时，将使用接收响应头的时间。 指定`last_byte`时，将使用接收完整响应的时间。最少时间负载均衡方式的指令是`least_time`。

* 通用哈希(Generic hash)

  管理员使用给定的文本，请求或运行时的变量或两者来定义哈希。NGINX通过为当前请求生成哈希并将其放置在上游服务器(upstream servers)上来在服务器之间分配负载。当你需要对发送请求的位置进行更多控制或确定哪个上游服务器最有可能缓存数据时，此方法非常有用。值得注意的是，将服务器添加到池中或从池中删除时，请求哈希将重新分配。该算法具有一个可选参数，`consistent`，以最大程度地减少重新分配的影响。通用哈希负载均衡方式的指令是`hash`。

* 随机(Random)

  该方式用于指示NGINX从服务器权重中随机选择一个服务器。可选的`two [method]`参数指示NGINX随机选择两个服务器，然后使用定义的的的负载均衡方法（method）来均衡这两个服务器。`two`参数的缺省`method`将使用`least_conn`方式。随机负载均衡方式的指令是`random`。

* IP哈希(IP hash)

  这个方式仅对HTTP有效，IP哈希以客户端的IP地址做为哈希。与在通用哈希(Generic hash)中使用远程变量稍有不同，此算法使用IPv4地址或整个IPv6地址的前三个八位字节。此方法确保客户端请求代理绑定到同一上游服务器，只要该服务器可用。这在应用程序的会话状态不是由公共共享内存处理时，非常有用。该方法在使用哈希分配请求时也可以使用`weight`参数。IP哈希负载均衡方式的指令是`ip_hash`。


## 2.5 粘滞Cookie(Sticky Cookie)

### 问题

你需要使用NGINX Plus将一个下流客户端绑定到上游服务器。

### 解答

使用`sticky cookie`指令指示NGINX Plus创建和跟踪一个cookie:

```
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    sticky cookie
            affinity
            expires=1h
            domain=.example.com
            httponly
            secure
            path=/;
}
```

上面的配置创建并跟踪将下游客户端与上游服务器绑定在一起的cookie。在此示例中，该cookie名为“affinity”，设置为example.com，在一小时内到期，不能在客户端使用设置为`httponly`，只能通过HTTPS发送`secure`，`/`并且对所有路径均有效。

### 讨论

在`sticky`指令上使用`cookie`参数会在第一个请求上创建一个cookie，其中包含有关上游服务器的信息。NGINX Plus跟踪此cookie，使其能够继续将后续请求定向到同一服务器。`cookie`参数的第一个位置参数是要创建和跟踪的cookie的名称。其他参数提供额外的控制，指示浏览器适当的使用cookie，比如过期时间(expires)、域(domain)、路径(path)，以及cookie是否可以被客户端使用(httponly)，或者是否可以通过不安全的协议传递(secure)。

## 2.6 粘滞探知(Sticky Learn)

### 问题

你需要通过NGINX Plus使用现有的cookie将下游客户端绑定到上游服务器。

### 解答

使用`sticky learn`指令来发现和跟踪上游应用程序创建的cookie:

```
upstream backend {
    server backend1.example.com:8080;
    server backend2.example.com:8081;
    sticky learn
            create=$upstream_cookie_cookiename
            lookup=$cookie_cookiename
            zone=client_sessions:2m;
}
```

上面的示例中指示NGINX通过在响应标头中查找名为COOKIENAME的cookie来查找和跟踪会话，并通过在请求头中查找相同的cookie来查找现有会话。这个会话关联存储在一个2 MB的共享内存区域中，该区域可以跟踪大约16,000个会话。Cookie的名称将始终是由应用程序指定。常用的cookie名称例如jsessionid或phpsessionid，通常是应用程序或应用程序服务器配置中设置的默认值。

### 讨论

当应用程序创建自己的会话状态（session-state）cookie时，NGINX Plus可以在请求响应中发现它们并跟踪它们。这种类型的cookie跟踪是在`sticky`指令提供了`learn`参数时执行的。用于跟踪cookie的共享内存是通过`zone`参数以及名称和大小指定的。NGINX Plus旨在通过指定的`create`参数在上游服务器的响应（response）中查找cookie，并使用`lookup`参数搜索先前注册的服务器affinity变量。 这些参数的值是HTTP模块公开的变量。

## 2.7 粘滞路由(Sticky Routing)

### 问题

使用NGINX Plus你需要将持久性会话路由到上游服务器的方式进行精细控制。

### 解答

将`sticky`指令与`route`参数一起使用可使用有关路由请求的变量：

```
map $cookie_jsessionid $route_cookie {
    ~.+\.(?P<route>\w+)$ $route;
}
map $request_uri $route_uri {
    ~jsessionid=.+\.(?P<route>\w+)$ $route;
}
upstream backend {
    server backend1.example.com route=a;
    server backend2.example.com route=b;
    sticky route $route_cookie $route_uri;
}
```

上面的示例尝试首先通过以下方式从Cookie中提取Java session ID：首先将Java会话ID cookie的值映射到具有第一个`map`块的变量，然后使用第二个`map`块通过在请求URI中查找名为`jsessionid`的参数来进行映射将值赋给变量。带有`route`参数的`sticky`指令将传递任意数量的变量。第一个非零或非空值用于路由, 如果使用了jsessionid cookie，则将请求路由到*backend1*； 如果使用URI参数，则将请求路由到*backend2*。尽管此示例基于Java的通用session ID，但同样适用于其他开发语言的session，例如`phpsessionid`，或应用程序为该session ID生成的任何保证的唯一标识符。

### 讨论

有时，你可能希望通过更精细的控制将流量定向到特定服务器。`sticky`指令加上`route`参数就是为了达到这个目的而构建的。与通用哈希（generic hash）负载均衡算法相比，粘滞路由（Sticky Route）可为你提供更好的控制，实际跟踪和粘性。客户端首先根据指定的路由路由到上游服务器，然后后续请求将在cookie或URI中携带路由信息。`sticky route`接受计算多个参数，第一个非空变量用于路由到服务器。`map`映射块可用于选择性地解析变量并将其另存为其他变量以在路由中使用。实质上，`sticky route`指令会在NGINX Plus的共享内存区域内创建一个会话，用于跟踪你设置给上游服务器的客户端会话标识符。并将具有该会话标识符的请求始终传递到与初始请求相同的上游服务器。

## 2.8 连接排空(Connection Draining)

### 问题

在维护与NGINX Plus的会话的同时，出于维护或其他原因，你需要优雅地删除服务器。

### 解决

通过NGINX Plus API使用`drain`参数(详见[第5章](Chapter_05.md))来指示NGINX停止发送未被跟踪的新连接:

```bash
$ curl -X POST -d '{"drain":true}' \
  'http://nginx.local/api/3/http/upstreams/backend/servers/0'
```
```json
{
    "id":0,
    "server":"172.17.0.3:80",
    "weight":1,
    "max_conns":0,
    "max_fails":1,
    "fail_timeout":
    "10s","slow_start":
    "0s",
    "route":"",
    "backup":false,
    "down":false,
    "drain":true
}
```

### 讨论

当会话状态存储在服务器本地时，必须先排空连接和持久会话，然后再将其从池中删除。排空连接是在与服务器的会话从上游池中删除之前，自然终止与服务器的会话的过程。你可以通过在服务器指令中添加`drain`参数来配置特定服务器的排空。设置了`drain`参数后，NGINX Plus停止向该服务器发送新会话，但是允许当前会话在其会话期间继续提供服务。你还可以通过将`drain`参数添加到upstream server指令来切换该配置。

## 2.9 被动健康检查

### 问题

你需要被动检查上游服务器的运行状况。

### 解答

使用NGINX健康检查与负载平衡，以确保只有健康的上游服务器被利用:

```
upstream backend {
    server backend1.example.com:1234 max_fails=3 fail_timeout=3s;
    server backend2.example.com:1234 max_fails=3 fail_timeout=3s;
}
```

上面的配置被动地监视上游运行状况，将max_failed指令设置为3，将fail_timeout设置为3秒。这些指令参数在Stream和HTTP服务器中以相同的方式工作。

### 讨论

NGINX的开源版本中提供了被动健康检查。在客户端请求连接通过NGINX时，被动监控失败或超时的连接。被动的健康检查默认是启动的。上面提动的参数可以帮助你更好的控制监控行为。健康监控对于所有类型的负载平衡都很重要，不仅从用户体验的角度来看如此，对于业务连续性也很同样重要。NGINX提供被动地监控上游的HTTP，TCP和UDP服务器，以确保它们正常运行。

## 2.10 主动健康检查

### 问题

你需要使用NGINX Plus主动检查你的上游服务器的运行状况。

### 解答

对于HTTP，在`localtion`块中使用`health_check`指令：

```
http {
    server {
        ...
        location / {
            proxy_pass http://backend;
            health_check interval=2s
            fails=2
            passes=5
            uri=/
            match=welcome;
        }
    }
    # 响应状态码为200, 内容类型为"text/html",
    # 并且响应body包含 "Welcome to nginx!"
    match welcome {
        status 200;
        header Content-Type = text/html;
        body ~ "Welcome to nginx!";
    }
}
```

这里的健康检查为HTTP服务器通过每两秒钟对URI'/'发出HTTP请求来检查上游服务器的运行状况。上游服务器必须通过五次连续的运行状况检查才能被视为运行状况良好。如果他们两次连续检查均未通过，则被视为不健康。来自上游服务器的响应必须与定义的`match`块匹配，该块将状态码定义为200，内容类型为"text/html"，并且响应body包含 "Welcome to nginx!"。HTTP `match`块具有三个指令：`status`，`header`和`body`。 所有这三个指令也都具有比较标志。

TCP/UDP服务的stream健康检查非常类似:

```
stream {
    ...
    server {
        listen 1234;
        proxy_pass stream_backend;
        health_check interval=10s
        passes=2
        fails=3;
        health_check_timeout 5s;
    }
    ...
}
```

在本例中，TCP服务器被配置为监听端口1234，并代理到上游的一组服务器，并主动检查这些服务器的健康状况。除了`uri`之外，stream的`health_check`指令采用与HTTP中相同的所有参数，并且stream版本有一个参数将检查协议切换为`udp`。在此示例中，时间间隔设置为10秒，要求两次通过被则认为是健康的，而三次失败则被认为是不健康的。主动的stream健康检查还能够验证来自上游服务器的响应，但是，stream服务器的`match`块只有两个指令：`send`和`expect`。`send`指令是要发送的原始数据（raw data），`expect`是要匹配确切的响应或正则表达式。

### 讨论

NGINX Plus中的主动健康检查不断向源服务器发出请求，以检查它们的健康状况。这些健康检查不仅可以测量响应代码，在NGINX Plus中，主动的HTTP健康检查基于上游服务器响应的一系列接受标准进行监控。你可以配置主动健康检查监视，以确定上游服务器检查的频率、服务器必须通过多少次检查才能被认为是健康的、失败的多少次数会被认为不健康，以及对响应结果预测比对。`match`参数指向一个`match`块，该块定义了响应的接受标准。`match`块还定义了在流上下文中用于TCP/UPD时要发送到上游服务器的数据。这些功能使NGINX能够确保上游服务器始终处于健康状态。

## 2.11 缓慢启动

### 问题

在承担全部生产负载之前，你的应用程序需要进行缓慢启动。

### 解答

使用server指令上的slow_start参数在指定时间内逐渐增加连接数，将服务器引入到上游负载平衡池：

```
upstream {
    zone backend 64k;
    server server1.example.com slow_start=20s;
    server server2.example.com slow_start=15s;
}
```

在将服务器指令配置重新引入到池中之后，它们将会被缓慢增加到上游服务器的流量。server1将在20秒内缓慢增加其连接数，而server2将在15秒内缓慢增加其连接数。

### 讨论

缓慢启动的概念就是在一段时间内缓慢增加代理到服务器的请求数量。缓慢启动使应用程序可以通过填充缓存来预热，启动数据库连接，而不会在启动后立即被连接淹没。当运行健康检查失败的服务器开始再次通过并重新进入负载均衡池时，此功能将生效。

## 2.12 TCP健康检查

### 问题

你需要检查上游TCP服务器的健康状况，并从池中删除不健康的服务器。

### 解答

使用`server`块中的`health_check`指令进行主动的健康检查:

```
stream {
    server {
        listen 3306;
        proxy_pass read_backend;
        health_check interval=10 passes=2 fails=3;
    }
}
```

该示例主动地监视上游服务器。如果上游服务器不能响应NGINX发起的3个或多个TCP连接，则会被认为是不健康的。NGINX每10秒检查一次。只有通过2次健康检查，服务器才会被认为是健康的。

### 讨论

NGINX Plus可以主动或被动地检测TCP健康状况。被动健康状态监视是通过记录客户机和上游服务器之间的通信来完成的。如果上游服务器超时或拒绝连接，被动健康检查将认为该服务器不健康。主动健康检查将启动自己的可配置检查来确定健康状况。活动健康状况检查不仅测试到上游服务器的连接，而且可以预测比对给定的响应。
