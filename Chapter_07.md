# 第七章 安全控制

## 7.0 介绍

安全是分层完成的，你的安全模型必须有多层才能真正得到强化。在本章中，我们将通过许多不同的方法来使用 NGINX 和 NGINX Plus 来保护你的 Web 应用程序。你可以将这些安全方法中的多个方法相互结合以帮助加强安全性。接下来的这些安全内容部分，探讨了 NGINX 和 NGINX Plus 的功能，这些功能可以帮助加强您的应用程序。你可能会注意到，本章没有涉及NGINX最大的安全特性之一，ModSecurity 3.0 NGINX模块，它将NGINX变成了Web应用防火墙(WAF)。要了解有关 WAF 功能的更多信息，请下载[ModSecurity 3.0 and NGINX: Quick Start Guide](https://www.nginx.com/resources/library/modsecurity-3-nginx-quick-start-guide/)。

## 7.1 基于IP地址的访问

### 问题

你需要根据客户端的 IP 地址来控制访问。

### 解答

使用HTTP访问模块控制对受保护资源的访问:

```
location /admin/ {
    deny 10.0.0.1;
    allow 10.0.0.0/20;
    allow 2001:0db8::/32;
    deny all;
}
```

给定的`location`块允许从 `10.0.0.0/20` 中除 `10.0.0.1` 之外的任何 IPv4 地址进行访问，允许从 `2001:0db8::/32` 子网中的 IPv6 地址进行访问，并拦截来自任何其他地址的请求返回403。`allow` 和 `deny` 指令可以在 `http`、`server`和`location`的上下文中使用。按顺序检查规则，直到找到远程地址的匹配项。

### 讨论

保护互联网上有价值的资源和服务必须分层进行。NGINX提供了成为这些层之一的能力。`deny` 指令阻止对给定上下文的访问，而 `allow` 指令设置于允许被阻止访问的子集。你可以使用 IP 地址，IPv4 或 IPv6、CIDR 块范围、关键字`all`，甚至是 Unix socket。通常在保护资源时，可能会允许一个内部IP地址块，并拒绝所有请求访问。

## 7.2 允许跨域资源共享

### 问题

你正在为来自另一个域的资源提供服务，并且需要允许跨域资源共享 (CORS) 以使浏览器能够利用这些资源。

### 解答

根据请求方法更改请求头以启用 CORS：

```
map $request_method $cors_method {
    OPTIONS 11;
    GET 1;
    POST 1;
    default 0;
}
server {
    ...
    location / {
        if ($cors_method ~ '1') {
            add_header 'Access-Control-Allow-Methods'
                'GET,POST,OPTIONS';
            add_header 'Access-Control-Allow-Origin'
                '*.example.com';
            add_header 'Access-Control-Allow-Headers'
                'DNT,
                 Keep-Alive,
                 User-Agent,
                 X-Requested-With,
                 If-Modified-Since,
                 Cache-Control,
                 Content-Type';
        }
        if ($cors_method = '11') {
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }
    }
}
```

这个例子中有很多内容，通过使用`map`将 `GET` 和 `POST` 方法组合在一起进行了压缩。`OPTIONS` 请求方法向客户端返回有关此服务器的 CORS 规则的预检请求。 CORS 下允许使用 `OPTIONS`、`GET` 和 `POST` 方法。设置 `Access-Control-Allow-Origin` 请求头允许从此服务器提供的内容也可用于与此请求头匹配的源页面。预检请求可以在客户端上缓存 `1,728,000` 秒或 `20` 天。

### 讨论

当JavaScript等资源请求的资源属于其他域时，就会发生CORS。当一个请求被认为是跨源的，浏览器就需要遵守CORS规则。如果该资源没有明确允许使用的请求头，浏览器则将不会使用该资源。为了让我们的资源被其他子域使用，我们必须设置 CORS 标头，这可以通过 `add_header` 指令完成。如果请求是具有标准内容类型的 `GET`、`HEAD` 或 `POST`，并且请求没有特殊请求头，则浏览器将发出请求并且仅检查来源。其他请求方法将导致浏览器发出预检请求以检查它将为该资源遵守的服务器的条款。如果你没有正确设置这些请求头，浏览器将在尝试利用这些资源时出错。

