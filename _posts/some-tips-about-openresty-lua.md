---
title: 关于openresty lua使用的一些tips
date: 2018-04-12 11:34:24
tags: 
    - Lua
    - OpenResty
---

1. nginx 是多 worker 进程的模型，所以除了共享内存字典是所有 worker 进程共享之外，其他的数据都是每 worker 一份的，无论是在 init_by_lua 里面创建的全局变量，还是 Lua 模块里的状态变量。 

2. 在某个请求里面更新某个 Lua 变量，只是更新了当前处理这个请求的 nginx worker 进程里的状态，并不会影响其他的 worker 进程（除非只配置了一个 nginx worker）。 

3. Lua VM 是每一个 nginx worker 进程一份。这些独立的 Lua VM 副本是从 nginx master 进程的 Lua VM 给 fork 出来的。而 init_by_lua 运行在 master 进程的 Lua VM 中，时间点发生在进程 fork 之前。 

4. 在共享内存字典中保存最新的数据，每个 worker 进程里通过 Lua 模块变量或者 init_by_lua 创建的全局变量`追踪`当前 worker 里实际使用的数据(worker需要不断同共享内存的数据进行比较并更新)。

5. 关于上述1、2、3、4点，更多请参考: 
    - [google group-init_by_lua中全局变量的用法](https://groups.google.com/forum/#!msg/openresty/N1GJZyaClkM/4xoNQO1MDhQJ)
    - [google group-worker内多个请求共享全局变量](https://groups.google.com/forum/#!topic/openresty/xd_33CSvU_o)

6. lua_code_cache的使用
    - 关闭lua_code_cache, 则每一个请求都由一个独立的lua VM来处理。因此，通过A请求变更的lua数据(如模块变量)，不会被B请求解析到，即使只配置了一个。
    - 关闭lua_code_cache的好处，对于纯lua文件(不涉及nginx解析的),在不重启nginx的情况下也能立即生效。
    - 启用lua_code_cache, 则同一个worker的所有请求共享一个lua VM的数据。因此，由该worker处理的A请求变更了lua数据(如模块变量)，则会被同一个worker处理的B请求访问到。  
    - 生产环境强烈建议启用lua_code_cache,否则会带来较大的性能损失。
    - 更多参考 [这里](https://github.com/openresty/lua-nginx-module#lua_code_cache)

7. 关于lua变量共享问题
    - 尽量不使用全局变量
    - 如果要使用，使用模块变量
    - 如果模块变量无法满足，使用共享内存或者分布式缓存
    - 更多参考 [lua-variable-scope](https://github.com/openresty/lua-nginx-module#lua-variable-scope)、[变量的共享范围](https://moonbingbing.gitbooks.io/openresty-best-practices/ngx_lua/lua-variable-scope.html)
    - [Data Sharing within an Nginx Worker](https://github.com/openresty/lua-nginx-module#data-sharing-within-an-nginx-worker)
    - [data-sharing-in-openresty](https://idndx.com/2018/01/19/data-sharing-in-openresty/)

8. 不应使用模块级的局部变量以及模块属性，存放任何请求级的数据。否则在 luacodecache 开启时，会造成请求间相互影响和数据竞争，产生不可预知的异常状况。
    - [关于 OPENRESTY 的两三事](http://lua.ren/topic/135/%E5%85%B3%E4%BA%8E-openresty-%E7%9A%84%E4%B8%A4%E4%B8%89%E4%BA%8B)

```
    关于变量共享的一个最小化配置:
    -- share.lua
    local _M={}
    local data = {}

    function _M.get_value(key)
        return data[key]
    end
    function _M.set_value(key,value)
        data[key] = value
    end

    return _M

    ### server.conf
    server {
        listen 8081;
        server_name 127.0.0.1;

        ### 通过请求A设置模块共享变量
        location = /1 {
            content_by_lua_block {
                local share = require('share')
                share.set_value('a','b')
                ngx.say(share.get_value('a'))
            }
        }

        ### 通过请求B读取共享变量
        location = /2 {
            content_by_lua_block {
                local share = require('share')
                ngx.say(share.get_value('a'))
            }
        }
    }

    -- init.lua
    package.path = "/usr/local/Cellar/openresty/1.13.6.1/lualib/?.lua;/usr/local/etc/openresty/lua/?.lua;;";

    ### nginx 主配置文件部分内容
    http {
        include       mime.types;
        default_type  application/octet-stream;

        log_format main '$remote_addr - [$time_local] "$request" $status '
        ' "$http_referer" "$http_user_agent" "$http_x_forwarded_for" ';

        access_log  logs/access.log  main;
        sendfile        on;
        keepalive_timeout  60;
        include server.conf;
        lua_code_cache on;
        init_by_lua_file lua/init.lua;
    }
```
