---
layout: post
title: ngx_http_core_module - error_page
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description:
---

## 一 概述

`error_page` 指令用来控制特定错误码出现时的应答显示，由 `ngx_http_core_module` 模块实现。

## 二 指令

### 1. `error_page`

```nginx
Syntax: 	error_page code ... [=[response]] uri;
Default: 	—
Context: 	http, server, location, if in location
```

定义为特定状态码显示的 `URI`，`URI` 可以包含变量。注意：状态码取值范围是：`[300, 599]`。示例：

```nginx
error_page 404             /404.html;
error_page 500 502 503 504 /50x.html;
```

如果应答状态码为 `404`、`500`、`502`、`503`、`504` 会内部跳转到 `uri`: `/404.html`、`/50x.html`，同时请求方法会修改为 `GET`。
可以使用 `=response` 指定应答状态码，示例（使用 `/404.php` 的状态码作为应答状态码）：

```nginx
error_page 404 = /404.php;
```

也可以使用 `301`、`302`、`303`、`307`、`308` 重定向跳转（默认状态码是 `302`），示例：

```nginx
error_page 403      http://example.com/forbidden.html;
error_page 404 =301 http://example.com/notfound.html;
```

### 2. `no_error_pages`

```nginx
Syntax: 	no_error_page;
Default: 	—
Context: 	http, server, location, if in location
```

关闭 `error_page` 功能，与 `error_page` 指令不能共存。

## 三 实现

`error_page` 指令在 `ngx_http_core_module` 中实现，会在 `ngx_http_core_loc_conf_t::error_pages` 动态数组中添加 `error_page` 元素（每个状态码对应一个动态数组元素）。

```c
static char *
ngx_http_core_error_page(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t *clcf = conf;

    u_char                            *p;
    ngx_int_t                          overwrite;
    ngx_str_t                         *value, uri, args;
    ngx_uint_t                         i, n;
    ngx_http_err_page_t               *err;
    ngx_http_complex_value_t           cv;
    ngx_http_compile_complex_value_t   ccv;

    // 已经配置了 no_error_pages 指令
    if (clcf->error_pages == NULL) {
        return "conflicts with \"no_error_pages\"";
    }

    if (clcf->error_pages == NGX_CONF_UNSET_PTR) {
        clcf->error_pages = ngx_array_create(cf->pool, 4,
                                             sizeof(ngx_http_err_page_t));
        if (clcf->error_pages == NULL) {
            return NGX_CONF_ERROR;
        }
    }

    value = cf->args->elts;

    // 应答状态码取
    i = cf->args->nelts - 2;

    if (value[i].data[0] == '=') {
        if (i == 1) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "invalid value \"%V\"", &value[i]);
            return NGX_CONF_ERROR;
        }

        if (value[i].len > 1) {
            overwrite = ngx_atoi(&value[i].data[1], value[i].len - 1);

            if (overwrite == NGX_ERROR) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid value \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }

        } else {
            overwrite = 0;
        }

        n = 2;

    } else {
        overwrite = -1;
        n = 1;
    }

    // 跳转 uri
    uri = value[cf->args->nelts - 1];

    ngx_memzero(&ccv, sizeof(ngx_http_compile_complex_value_t));

    ccv.cf = cf;
    ccv.value = &uri;
    ccv.complex_value = &cv;

    if (ngx_http_compile_complex_value(&ccv) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    ngx_str_null(&args);

    if (cv.lengths == NULL && uri.len && uri.data[0] == '/') {
        p = (u_char *) ngx_strchr(uri.data, '?');

        if (p) {
            cv.value.len = p - uri.data;
            cv.value.data = uri.data;
            p++;
            args.len = (uri.data + uri.len) - p;
            args.data = p;
        }
    }

    // 为每个错误状态码创建 error_page 配置
    for (i = 1; i < cf->args->nelts - n; i++) {
        err = ngx_array_push(clcf->error_pages);
        if (err == NULL) {
            return NGX_CONF_ERROR;
        }

        // 错误码数值
        err->status = ngx_atoi(value[i].data, value[i].len);

        if (err->status == NGX_ERROR || err->status == 499) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "invalid value \"%V\"", &value[i]);
            return NGX_CONF_ERROR;
        }

        // 错误码有效范围
        if (err->status < 300 || err->status > 599) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "value \"%V\" must be between 300 and 599",
                               &value[i]);
            return NGX_CONF_ERROR;
        }

        // 应答状态码
        err->overwrite = overwrite;

        if (overwrite == -1) {
            switch (err->status) {
                case NGX_HTTP_TO_HTTPS:
                case NGX_HTTPS_CERT_ERROR:
                case NGX_HTTPS_NO_CERT:
                    err->overwrite = NGX_HTTP_BAD_REQUEST;
                default:
                    break;
            }
        }

        err->value = cv;
        err->args = args;
    }

    return NGX_CONF_OK;
}
```

在 `ngx_http_finalize_request` 函数处理中会根据 `location` 配置，使用 `error_page` 进行应答处理。