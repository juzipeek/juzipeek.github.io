---
layout: post
title: lua-upstream-nginx-module 模块
category: NGINX
tags: NGINX,C,Lua
keywords: NGINX,C,Lua
description: 
---

## 一 概述

`lua-upstream` 模块提供了对 `upstrem` 配置的查看，查看所有的 `upstream`、`upstream` 内所有的/启用的/备用的 `server`，当前使用的 `upstream` 名。**虽然有 `set_peer_down` 指令，但是模块只修改单个 `worker` 内的 `server` 标记（处理 `set_peer_down` 请求的 `worker`），无法再生产环境使用**。

## 二 示例

```ngin
worker_processes  1;

error_log logs/error.log info;

events {
    worker_connections 1024;
}

http {
    
    lua_code_cache off;

    upstream backend {
        server localhost:8081 down;
        server localhost:8082;
    }

    server {
        listen 8081;
        location / {
            default_type text/plain; 
            content_by_lua_block {
                ngx.say("hello world")
            }
        }
    }
    server {
        listen 8082;
        location / {
            default_type text/plain; 
            content_by_lua_block {

                ngx.say("hello world 8082")
            }
        }
    }

    server {
        listen 8080;

				## 查看当前使用的 upstream
        location /call_upstream {
            header_filter_by_lua_block {
                local upstream = require "ngx.upstream"
                ngx.log(ngx.INFO,"current upstream:", upstream.current_upstream_name())
            }
            proxy_pass http://backend;
        }
        
				## 查看所有 upstream
        location /status_upstream {
            content_by_lua_block {
                local upstream = require "ngx.upstream"
                local us = upstream.get_upstreams()
                for _, v in ipairs(us) do
                    ngx.log(ngx.INFO, "upstream name:", v)
                    local servers, _ = upstream.get_servers(v)

                    if not servers then
                        ngx.log(ngx.ERR, "no server in upstream")
                    else 
                        for idx, s in ipairs (servers) do
                            local msg = "idx:" .. idx
                            
                            for k, v in pairs (s) do
                                msg = msg .. " " .. k .. ":"
                                if type(v) == "table" then
                                    msg = msg .. table.concat(v, ",")
                                else
                                    msg = msg .. tostring(v)
                                end
                            end

                            ngx.log(ngx.INFO, msg)
                        end
                    end
                end
            }
        }
        
        ## 将 backend 中第一个启用 server 设置为 down 状态
        location /set_upstream {
            content_by_lua_block {
                local upstream = require "ngx.upstream"

                upstream.set_peer_down("backend", false, 0, false)
            }
        }
    }
}
```

### 1. 示例运行

在调用 `status_upstream`、`set_upstream`、`status_upstream` 调用顺序中，最终并无法看到 `server` 被标记为 `down` 状态。这是 `set_peer_down` 修改的是 `round_robin` 模块中 `server` 的标记，`get_servers` 读取的是配置结构体（`ngx_http_upstream_server_t`）中的内容。


### 2. `get_servers` 函数

如果 `server` 未指定 `down/backup` 标记，在 `get_servers` 函数返回值中不会包含 `down/backup` 状态。

## 三 后记

在 `lua-upstream` 模块的 `TODO` 中有提及动态添加、删除 `server` 的考虑，到时候应该会更好用。


## 四 链接

- *[项目连接](https://github.com/openresty/lua-upstream-nginx-module)*

