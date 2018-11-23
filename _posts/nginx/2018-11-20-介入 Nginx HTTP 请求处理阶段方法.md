---
layout: post
title: 介入 Nginx HTTP 请求处理阶段方法
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## NGINX 处理阶段说明

在 NGINX 中 HTTP 请求分为 11 个处理阶段，其中只有 7 个阶段用户可以介入。除了 `NGX_HTTP_CONTENT_PHASE` 阶段外，其余阶段只能通过向全局的 `ngx_http_core_main_conf_t` 结构体的 `phases` 数组添加 `ngx_http_handler_pt` 处理方法实现。在 `NGX_HTTP_CONTENT_PHASE` 阶段，除了上述方法外还可以通过将 `ngx_http_handler_pt` 设置到 `ngx_http_core_loc_conf_t` 结构体的 `handler` 指针中。

## 使用 `ngx_http_core_loc_conf_t` 介入 NGX_HTTP_CONTENT_PHASE 处理

如果使用将 `ngx_http_handler_pt` 设置到 `ngx_http_core_loc_conf_t->handler` 的方法来处理 HTTP 请求，那么在 `NGX_HTTP_CONTENT_PHASE` 阶段只有 `handler` 指针指向的处理函数被调用。并且无论处理函数返回任何值，都会直接调用 `ngx_http_finalize_request` 方法结束请求。

示例代码：

```c
static char *
ngx_http_empty_gif(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_empty_gif_handler;

    return NGX_CONF_OK;
}

```



## 向 `ngx_http_core_main_conf_t->phases` 添加处理方法

这种方法通过向 `NGX_HTTP_CONTENT_PHASE` 阶段的动态数组添加处理函数来实现，此时在 `NGX_HTTP_CONTENT_PHASE` 会调用本阶段的多个处理函数。如果处理函数返回 `NGX_DECLINED` 会调用下一个处理函数，返回其他值则会调用 `ngx_http_finalize_request` 方法结束请求。

示例代码：

```c
static ngx_int_t
ngx_http_autoindex_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_autoindex_handler;

    return NGX_OK;
}

```



## 介入其他处理阶段

示例代码：

```c
static ngx_int_t
ngx_http_limit_conn_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_limit_conn_handler;

    return NGX_OK;
}

```

