---
title: nginx resolver数据缓存及解析超时时间参数
date: 2018-05-08 16:03:18
tags:
    - NGINX
---

在配置nginx的resolver时，有两个与时间相关的参数。

一、valid
```
Syntax: resolver address ... [valid=time] [ipv6=on|off];
Default:    —
Context:    http, server, location

如：
resolver 127.0.0.1 [::1]:5353 valid=30s;
```

`valid` flag means how long nginx will consider answer from resolver as valid and will not ask resolver for that period.
即DNS返回结果的缓存时间，在缓存有效时间内，nginx不会再次向DNS请求解析结果。

二、resolver_timeout
```
Syntax: resolver_timeout time;
Default:    
resolver_timeout 30s;
Context:    http, server, location

```

`resolve_timeout` sets how long nginx will wait for anwser from resolver (DNS).
即nginx发起的DNS查询请求超时时间。

更多参考：

- http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver
- https://stackoverflow.com/questions/28606696/what-is-the-difference-in-resolver-valid-time-and-resolver-timeout-in-nginx 