---
layout: post
title: 使用OpenResty控制CDN回源主机
date: 2016-03-07 20:05:20
tags: engineering
---

年前粗略看了一下[《OpenResty最佳实践》](https://www.gitbook.com/book/moonbingbing/openresty-best-practices/details)，感觉OpenResty是个好东西呀，但是一下子又找不到使用场景，所以就放到一边了。最近遇到一个需求，感觉用OpenResty正合适，所以终于在生产环境实践了一把。

## 需求

一个JavaScript脚本分发服务：

```plain

        key
浏览器 --------------> 分发服务
GET /js?key=xxxx

        302 CDN地址
浏览器 <-------------- 分发服务
Location: cdn.x.com/digest/xxxx.js

//分发服务根据key获取用户的配置（用户可以通过web界面修改，需要尽快生效）
//以及配置对应的静态js文件（分布在分发服务的本地硬盘），
//计算配置和静态内容的摘要，拼接到CDN的URL中，
//当配置或静态内容更新后，重定向新的URL，CDN将触发回源流程。


        CDN地址              回源
浏览器 --------------> CDN --------------> 分发服务

```

由于该服务的请求量还是挺大的（每天的重定向请求量在七百六十万左右，回源请求量在八万左右），所以部署了两个分发服务，前面挡了一个Nginx做负载均衡。

本来做Nginx是为了容错，结果就因为这个还带来了一个小问题：

由于两台分发服务的配置更新存在时间差，特别是静态文件，所以设想一下，服务A配置已经更新，而服务B没有更新，一个请求来到服务A，重定向到新的URL，而回源的请求来到服务B，B返回旧的JS脚本，被CDN缓存，那么新配置的生效时间将会推迟到CDN缓存失效。。。

## 解决方案

首先想到的是将回源服务独立出来，如果有配置更新，首先等待回源服务生效，然后再更新重定向服务。这个方案的缺点就是麻烦，得多部署一套服务，前端人员更新JS静态文件得等挺久，所以否掉了。

然后想到的一个方案是，重定向的时候在URL中标示出回源主机，举例来说，A返回重定向地址，回源请求来到Nginx，Nginx根据URL判断需要转发到A，而不是B，如此就不会出现上面提到的问题，另外即使某台服务宕机，另外一台也可以正常提供服务。

这块逻辑不应该耦合到原有的分发服务，所以就是想到了使用OpenResty解决：

```plain
http {

...

    upstream js_backend {
        server 10.1.1.20:8080;
        server 10.1.1.21:8080;
        keepalive 64;
    }

...

    server {
        listen       80;

...

        location ^~ /20/ {
            proxy_pass http://10.1.1.20:8080/;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }

        location ^~ /21/ {
            proxy_pass http://10.1.1.21:8080/;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }

        location = /js {
            proxy_pass http://js_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";

            header_filter_by_lua_block {
                if ngx.status == 302 then
                    local regex = "^([0-9]+).([0-9]+).([0-9]+).([0-9]+):([0-9]+)$"
                    local m, err = ngx.re.match(ngx.var.upstream_addr, regex)
                    if m then
                        local loc = ngx.header["Location"]
                        local s = loc:find("/", 9)
                        ngx.header["Location"] = table.concat({loc:sub(1, s), m[4], "/", loc:sub(s+1, -1)})
                    else
                        ngx.log(ngx.ERR, err)
                    end
                end
            }
        }
        
        ...
        
    }
}
    
```

这里没有把`upstream_addr`完全拼进去，然后根据该段转发，主要是考虑到安全上的问题。

服务切了过来，一切正常，性能也没受到影响。（刚切过来时，回源请求比较多，受了点儿影响）

总的来说，对于OpenResty印象相当好，如果场景合适，以后会多多使用。
