# 第六章 身份验证

## 6.0 介绍

NGINX 能够对客户端进行身份验证。使用NGINX对客户端请求进行身份验证可以减轻服务器工作量，并提供阻止未经身份验证的请求到达你的应用程序服务器的能力。NGINX开源可用的模块包括基本身份验证和身份验证子请求。NGINX Plus专有模块提供用于验证 JSON Web Tokens (JWT)及使用认证标准OpenID Connect的第三方认证的支持。

## 6.1 HTTP 基本身份验证

### 问题

你需要通过 HTTP 基本身份验证来保护你的应用程序或内容。

### 解答

生成如下格式的文件，其中密码使用一种可用的格式进行加密或哈希运算:

```
# comment
name1:password1
name2:password2:comment
name3:password3
```

用户名是第一个字段，密码是第二个字段，分隔符是冒号"`:`"。有一个可选的第三字段，你可以用来备注标记每个用户。NGINX 可以理解几种不同的密码格式，其中之一是密码是否使用 C 函数 `crypt()` 进行加密。 该函数通过 `openssl passwd` 命令暴露给命令行。安装 `openssl` 后，你可以通过使用以下命令方式创建加密的密码字符串：

```
$ openssl passwd MyPassword1234
```

上面的输出将是可以在NGINX密码文件中使用的字符串。

在 NGINX 配置中使用 `auth_basic` 和 `auth_basic_user_file` 指令来启用基本身份验证：

```
location / {
    auth_basic "Private site";
    auth_basic_user_file conf.d/passwd;
}
```

你可以在 `http`、`service`或`location`的上下文中使用 `auth_basic` 指令。`auth_basic`指令接受一个字符串参数，当未经身份验证的用户访问时，该参数将会显示在基本身份验证弹窗中。`auth_basic_user_file`指定用户文件的路径。

### 讨论

你可以通过多种方式和多种不同的格式生成基本的身份验证密码，使其具备不同的安全级别。Apache 的 `htpasswd` 命令也可以生成密码。 `openssl` 和 `htpasswd` 命令都使用 `apr1` 算法生成密码，NGINX 都可以理解。密码也可以是轻型目录访问协议 (LDAP) 和 Dovecot使用的加盐 SHA-1 格式。NGINX 支持更多的格式和哈希算法； 然而，它们中的许多被认为是不安全的，因为它们很容易被暴力攻击破解。

你可以使用基本的身份验证来保护整个NGINX主机、指定的虚拟服务器，甚至只是特定的位置块。基本身份验证不会取代 Web 应用程序的用户身份验证，但它可以帮助保护隐私信息的安全。实际上，基本身份验证是由服务器返回一个未经授权的 401 HTTP 状态码和响应头 `WWW-Authenticate` 来实现的。这个请求头的值为`Basic realm="你的字符串"`。此响应会导致浏览器提示输入用户名和密码。用户名和密码用冒号连接和分隔，然后用base64编码，然后在名为`Authorization`的请求头中发送。`Authorization` 请求头将指定一个`Basic`和`user:password`编码字符串。服务器解码请求头并根据 `auth_basic_user_file` 提供的用户文件进行验证。因为用户名密码字符串仅仅是base64编码的，所以建议在使用带有基本身份验证时使用HTTPS协议（进一步加密，保证安全）。

## 6.2 身份认证子请求

### 问题

你有一个第三方身份验证系统，你希望对其请求进行身份验证。

### 解答

使用`http_auth_request_module`向身份验证服务发出请求，以便在提供请求之前验证身份：

```
location /private/ {
    auth_request /auth;
    auth_request_set $auth_status $upstream_status;
}
location = /auth {
    internal;
    proxy_pass              http://auth-server;
    proxy_pass_request_body off;
    proxy_set_header        Content-Length "";
    proxy_set_header        X-Original-URI $request_uri;
}
```

`auth_request`指令接受一个URI参数，该参数必须是一个本地`internal`内部的`location`。`auth_request_set`指令允许您从认证子请求中设置变量。

### 讨论

`http_auth_request_module`支持对NGINX服务器处理的每个请求进行身份验证。该模块在服务原始请求之前发出一个子请求，以确定该请求是否有权访问它所请求的资源。整个原始请求被代理到这个子请求的`location`。身份验证`location`充当子请求的典型代理，并发送原始请求，包括原始请求主体和请求头。子请求的HTTP状态代码决定是否有授予访问权限。如果子请求返回一个HTTP 200状态码，则身份验证成功，请求得到满足。如果子请求返回HTTP 401或403，则将为原始请求返回相同的结果。如果你的身份验证服务返回没有请求正文，您可以使用`proxy_pass_request_body`指令删除请求正文，如上所示。这种做法将减少请求的大小和时间。因为响应体被丢弃，`Content-Length`头必须设置为空字符串。如果你的身份验证服务器需要知道请求访问的 URI，则你需要将该URI值放在你的身份验证服务检查验证的自定义请求头中。如果你确实希望将子请求中的某些内容保留给身份验证服务器，例如响应标头或其他信息，则可以使用 `auth_request_set` 指令从响应数据中创建新的变量。

## 6.3 验证 JWT

### 问题

在使用 NGINX Plus 处理请求之前，你需要验证 JWT。

### 解答

使用NGINX Plus的HTTP JWT认证模块来验证令牌签名，并将JWT声明和头作为NGINX变量嵌入:

```
location /api/ {
    auth_jwt "api";
    auth_jwt_key_file conf/keys.json;
}
```

此配置支持对该`location`的jwt进行验证。`auth_jwt` 指令传递一个字符串，作为身份验证域（Authentication realm）。`auth_jwt` 接受一个可选令牌参数变量，用于保存JWT。默认情况下，根据JWT标准使用Authentication请求头。`auth_jwt` 指令还可用于从继承的配置中取消所需的 JWT 身份验证的影响。要`off`关闭身份验证，只将关键字传递给 `auth_jwt` 指令即可。同样要取消继承的身份验证要求，也只需将 `off` 关键字传递给 `auth_jwt` 指令即可。`auth_jwt_key_file`接受单个参数，该参数是标准JSON Web key格式的密钥文件的路径。

### 讨论

NGINX Plus能够验证令牌的JSON web签名类型，而不是整个令牌都是加密的JSON web加密类型。NGINX Plus支持验证使用HS256、RS256和ES256算法签名的签名。让 NGINX Plus 验证令牌可以节省向身份验证服务发出子请求所需的时间和资源，NGINX Plus解码JWT请求头和有效载荷（payload），并捕获标准请求头和声明(claims)到嵌入式变量中供你使用。

### 另请参见

[RFC Standard Documentation of JSON Web Signature](https://datatracker.ietf.org/doc/html/rfc7515)

[RFC Standard Documentation of JSON Web Algorithms](https://datatracker.ietf.org/doc/html/rfc7518)

[RFC Standard Documentation of JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)

[NGINX Embedded Variables](http://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html#variables)

[Detailed NGINX Blog](https://www.nginx.com/blog/authenticating-api-clients-jwt-nginx-plus/)

## 6.4 创建JSON Web Keys

### 问题

你需要一个 JSON Web Key 供 NGINX Plus 使用。

### 解答

NGINX Plus采用RFC标准中指定的JWK (JSON Web Key)格式。这个标准允许在JWK文件中包含一个关键对象数组。

以下是key文件的示例：

```json
{"keys":
    [
        {
            "kty":"oct",
            "kid":"0001",
            "k":"OctetSequenceKeyValue"
        },
        {
            "kty":"EC",
            "kid":"0002",
            "crv":"P-256",
            "x": "XCoordinateValue",
            "y": "YCoordinateValue",
            "d": "PrivateExponent",
            "use": "sig"
        },
        {
            "kty":"RSA",
            "kid":"0003",
            "n": "Modulus",
            "e": "Exponent",
            "d": "PrivateExponent"
        }
    ]
}
```

如上所示的JWK文件演示了RFC标准中提到的三种初始密钥类型。这些密钥的格式都是 RFC 标准的一部分。 `kty` 属性所指的就是密钥类型。这个文件显示了三种密钥类型:Octet Sequence (oct)、EllipticCurve (EC)和RSA类型。`kid` 属性是密钥 ID。 这些密钥的其他属性在该类型密钥的标准中都有相应的规定。查看这些标准的 RFC 文档可获取更多信息。

### 讨论

有许多不同语言的库可用于生成 JSON Web Key。建议创建一个关键服务，作为中央 JWK 权限的管理，以定期创建和轮换你的 JWK。为了增强安全性，建议使你的 JWK 与 SSL/TLS 认证一样安全。使用适当的用户和组权限保护你的密钥文件。将它们保存在主机的内存中是最佳做法。你可以通过创建一个内存文件系统来实现，比如ramfs。定期轮换密钥也很重要；你可以选择创建一个密钥服务来创建公钥和私钥，并通过 API 将它们提供给应用程序和NGINX。

### 另请参见

[RFC standardization documentation of JSON Web Key](https://datatracker.ietf.org/doc/html/rfc7517)

## 6.5 通过现有 OpenID Connect SSO 对用户进行身份验证

### 问题

你想将 OpenID Connect 身份验证验证整合到 NGINX Plus。

### 解答

使用 NGINX Plus 自带的 JWT 模块来保护一个 `location` 或 `server`，并指示 `auth_jwt` 指令使用 `$cookie_auth_token` 作为要验证的令牌：

```
location /private/ {
    auth_jwt "Google Oauth" token=$cookie_auth_token;
    auth_jwt_key_file /etc/nginx/google_certs.jwk;
}
```

此配置指示 NGINX Plus 使用 JWT 验证保护 <i>/private/</i> URI路径。Google OAuth 2.0 OpenID Connect 使用 cookie `auth_token` 而不是默认的bearer token。因此，你必须指示 NGINX 在此 cookie 中而不是在 NGINX Plus 的默认位置中查找令牌。我们将在[6.6节](Chapter_06.md#66-从 Google 获取 JSON Web Key)中介绍，如何设置`auth_jwt_key_file`的文件路径。

### 讨论

上面的配置演示了如何使用 NGINX Plus 验证 Google OAuth 2.0 OpenID Connect JSON Web 令牌。NGINX Plus的HTTP JWT认证模块能够验证任何符合RFC的JSON Web签名规范的JSON Web令牌，允许任何使用JSON Web令牌的SSO授权快速的在NGINX Plus层中进行验证。OpenID 1.0 协议是 OAuth 2.0 身份验证协议之上的一层，它添加了身份，允许使用 JWT 来证明发送请求的用户的身份。通过令牌的签名，NGINX Plus可以验证令牌自签名以来没有被修改过。通过这种方式，Google使用了一个异步签名方法，并使其能够在保持私有JWK秘密的同时发布公共JWK。NGINX Plus 还可以控制 OpenID Connect 1.0 的授权代码流，使 NGINX Plus 成为 OpenID Connect 的中继方。该功能能够与大多数主流身份标识提供商集成，包括CA Single Sign-On（以前称为 SiteMinder）、ForgeRock OpenAM、Keycloak、Okta、OneLogin和Ping Identity.有关 NGINX Plus 作为 OpenID Connect 身份验证依赖方的更多信息和参考实现，请查看 [NGINX Inc OpenID Connect GitHub Repository](https://github.com/nginxinc/nginx-openid-connect)。

### 另请参见

[Detailed NGINX Blog on OpenID Connect](https://www.nginx.com/blog/authenticating-users-existing-applications-openid-connect-nginx-plus/)

[OpenID Connect](https://openid.net/connect/)

## 6.6 从 Google 获取 JSON Web Key

### 问题

你需要从 Google 获取 JSON Web Key，以便在使用 NGINX Plus 验证 OpenID Connect 令牌时使用。

### 解决

利用 Cron 每小时请求一组新的密钥，以确保你始终是最新的：

```bash
0 * * * * root wget https://www.googleapis.com/oauth2/v3/ \
   certs-O /etc/nginx/google_certs.jwk
```

此代码片段是 crontab 文件中的一行。 类 Unix 系统对于 crontab 文件的存放位置有很多选择。每个用户都会有一个用户特定的 crontab，并且<i> /etc/ </i> 目录中还有许多文件和目录。

### 讨论

Cron 是在类 Unix 系统上运行计划任务的常用方法。 JSON Web Keys 应定期轮换以确保密钥的安全，进而确保你的系统安全。为确保你始终拥有来自 Google 的最新密钥，你需要定期检查新的 JWK。 这个 Cron 解决方案只是这样做的其中一种方式。

### 另请参见

[Cron](https://linux.die.net/man/8/cron)