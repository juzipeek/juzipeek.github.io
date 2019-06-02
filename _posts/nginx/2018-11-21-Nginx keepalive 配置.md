---
layout: post
title: Nginx keepalive 配置
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## 使用限制

对于客户端、upstream 都可以设置 keepalive 选项。

## 客户端设置

- keepalive_requests 

  The number of requests a client can make over a single keepalive connection. The default is 100, but a much higher value can be especially useful for testing with a load‑generation tool, which generally sends a large number of requests from a single client.

- keepalive_timeout

  How long an idle keepalive connection remains open.default 75000 ms.

## upstream 设置

- keepalive

  The number of idle keepalive connections to an upstream server that remain open for each worker process. There is no default value.

示例

```nginx
upstream memcached_backend {
    server 127.0.0.1:11211;
    server 10.0.0.2:11211;

    keepalive 32;
}
```

