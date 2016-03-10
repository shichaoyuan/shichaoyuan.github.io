---
layout: post
title: HTTPS相关的命令和配置
date: 2016-03-10 20:05:20
tags: engineering
---

## 命令

```plain
openssl req -new -newkey rsa:2048 -nodes -keyout test.com.key -out test.com.csr
```

生成按照提示输入各种信息，其中Common Name指的就是证书认证的域名。

这里的`rsa`标示证书类型，如果不考虑兼容性，现在推荐使用`ec`。

将csr提交给CA，各种审核过后就可以下载crt证书。证书通常是pem格式，可以在Nginx中直接使用，另外阿里云的负载均衡和CDN服务也是要这种格式。

如果想要在tomcat中使用，需要转一下

```plain
openssl pkcs12 -export -in x.com.chained.crt -inkey x.com.key -out x.com.p12
//按照提示，必须输入密码
//pkcs12是另外一种格式，文件中包含证书和私钥（密码保护）

keytool -importkeystore -deststorepass [changeit] -destkeypass [changeit] -destkeystore x.com.keystore -srckeystore x.com.p12 -srcstoretype PKCS12 -srcstorepass some-password
```

生成dh参数

```plain
openssl dhparam -out dh2048.pem 2048
```

查看证书状态

```plain
openssl s_client -connect js.touclick.com:443 -status
```

## 配置

关于服务端的配置，需要考虑的问题特别多，而且未来肯定会出现新的问题。。。

所以要感谢Mozilla提供了一个生成配置的小工具：[ssl-config-generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/)

另外一个wiki（[Server Side TLS](https://wiki.mozilla.org/Security/Server_Side_TLS)）对这些配置做了一些解释，只要watch其Github仓库就可以很方便的跟进更新。

下面简单解释一下Nginx的配置：

```plain
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    # Redirect all HTTP requests to HTTPS with a 301 Moved Permanently response.
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    # certs sent to the client in SERVER HELLO are concatenated in ssl_certificate
    ssl_certificate /path/to/signed_cert_plus_intermediates;
    ssl_certificate_key /path/to/private_key;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
    ssl_dhparam /path/to/dhparam.pem;

    # intermediate configuration. tweak to your needs.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;

    ## verify chain of trust of OCSP response using Root CA and Intermediate certs
    ssl_trusted_certificate /path/to/root_CA_cert_plus_intermediates;

    resolver <IP DNS resolver>;

    ....
}
```

`ssl_certificate`指定证书文件的位置，`ssl_certificate_key`指定私钥文件的位置，`ssl_dhparam`指定DH参数文件的位置；

`ssl_session_timeout`、`ssl_session_cache`和`ssl_session_tickets`控制会话密钥重用，有助于改善性能，当然也会增加安全风险;

`ssl_protocols`指定支持的协议版本，`ssl_ciphers`指定支持的算法，`ssl_prefer_server_ciphers`指定优先选择服务端指定的算法，就是上一个字段的值，值的顺序很重要（[《从启用 HTTP/2 导致网站无法访问说起》](https://imququ.com/post/why-tls-handshake-failed-with-http2-enabled.html)这篇文章描述的问题对理解这个参数很有帮助）；

`ssl_stapling`、`ssl_stapling_verify `、`ssl_trusted_certificate`和`resolver`控制OCSP Stapling，也就是缓存证书状态，可以使浏览器省去验证流程（不过我们用Godaddy的证书，日志里总是报“获取状态超时”。。。）
