---
layout: post
title: ngx_http_mirror_module
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description:
---

## 一 概述

`ngx_http_mirror_module` 模块使用**子请求**方式创建原始请求的镜像，子请求接收到的应答会被忽略。使用 `ngx_http_mirror_module` 可以方便的实现流量镜像，以供测试使用。

*在 `nginx-1.13.4` 版本加入，`ngx_http_mirror_module` 在 `NGX_HTTP_PRECONTENT_PHASE`阶段介入处理，虽然 HTTP 请求处理还是由 11 阶段构成，但是，`NGX_HTTP_TRY_FILES_PHASE` 阶段被修改为 `NGX_HTTP_PRECONTENT_PHASE` 阶段，允许模块介入处理。*

## 二 指令

### 1. `mirror`

```nginx
Syntax: 	mirror uri | off;
Default: 	mirror off;
Context: 	http, server, location
```

设置原始请求被镜像到的目的 `uri`（当前 `server{}` 内的 `location`），使用多个 `mirror` 指令可以复制多份。

### 2. `mirror_request_body`

```nginx
Syntax: 	mirror_request_body on | off;
Default: 	mirror_request_body on;
Context: 	http, server, location
```

控制是否转发原始请求包体。启用后，将在创建子请求前读取客户端请求包体。

## 三 实现

`mirror` 功能是通过子请求来实现，模块简单。对于 `mirrro` 指令处理，只是在模块中注册镜像 `uri`，在请求处理中会依次创建子请求。请求处理实现：

```c
// 在 NGX_HTTP_PRECONTENT_PHASE 阶段进行调用
static ngx_int_t
ngx_http_mirror_handler(ngx_http_request_t *r)
{
    // ... 省略代码
    if (r != r->main) {
        return NGX_DECLINED;
    }

    mlcf = ngx_http_get_module_loc_conf(r, ngx_http_mirror_module);

    if (mlcf->mirror == NULL) {
        return NGX_DECLINED;
    }

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0, "mirror handler");
    // 需要镜像请求包体
    if (mlcf->request_body) {
        ctx = ngx_http_get_module_ctx(r, ngx_http_mirror_module);

        if (ctx) {
            return ctx->status;
        }

        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_mirror_ctx_t));
        if (ctx == NULL) {
            return NGX_ERROR;
        }

        ctx->status = NGX_DONE;

        ngx_http_set_ctx(r, ctx, ngx_http_mirror_module);
        // 先读取请求包体，再调用 ngx_http_mirror_body_handler 处理
        rc = ngx_http_read_client_request_body(r, ngx_http_mirror_body_handler);
        if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
            return rc;
        }

        ngx_http_finalize_request(r, NGX_DONE);
        return NGX_DONE;
    }
    // 创建子请求
    return ngx_http_mirror_handler_internal(r);
}
```

