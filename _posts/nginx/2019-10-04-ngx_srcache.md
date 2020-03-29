---
layout: post
title: ngx_srcache
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description:
---

## 一 概述

`ngx_srcache` 是基于子请求的透明缓存层，能够为各种 `location` 提供缓存服务。通常，`memc-nginx-module` 与此模块一起使用以提供具体的缓存服务。 但是任何提供 REST 接口的模块都可以用作此模块使用的 `fetch` 和 `store` 子请求。

## 二 指令

### 1. `srcache_fetch`

```nginx
syntax: srcache_fetch <method> <uri> <args>?
default: no
context: http, server, location, location if
phase: post-access
```

在 `access` 阶段注册处理函数，请求到来时会发出 Nginx 子请求以进行缓存查找。子请求收到非 `200` 应答时触发 `cache-miss`，会进行后续阶段的处理；当子请求收到 `200` 应答时会触发 `cache-hit` 处理，会使用子请求收到内容进行应答。**确定应答头、应答体**

### 2. `srcache_fetch_skip`

```nginx
syntax: srcache_fetch_skip <flag>
default: srcache_fetch_skip 0
context: http, server, location, location if
phase: post-access
```

使用 `flag` 作为条件，用来判断是否需要继续进行缓存查询。`flag` 可以是变量，当 `flag` 不是空字符串并且不为 “0” 时将跳过缓存查找。

### 3. `srcache_store`

```nginx
syntax: srcache_store <method> <uri> <args>?
default: no
context: http, server, location, location if
phase: output-filter
```

该指令注册一个输出过滤器处理函数，该处理函数将发出 Nginx 子请求，将当前主请求的响应保存到缓存中。 子请求的状态代码将被忽略。

默认情况下响应状态码、响应头（部分响应头未存储）、响应包体都会被存储在缓存中。可以通过 `srcache_store_pass_header` 或 `srcache_store_hide_header` 指令控制允许缓存的应答头。原始响应数据会被立即响应到客户端，`srcache` 不会影响主请求。

### 4. `srcache_store_max_size`

```nginx
syntax: srcache_store_max_size <size>
default: srcache_store_max_size 0
context: http, server, location, location if
phase: output-header-filter
```

当响应包体大于 `srcache_store_max_size` 设置，模块不会将当前应答存储缓存中。

### 5. `srcache_store_skip`

```nginx
syntax: srcache_store_skip <flag>
default: srcache_store_skip 0
context: http, server, location, location if
phase: output-header-filter
```

与 `srcache_fetch_skip` 指令类似，只不过 `store_skip` 指令用来控制是否需要进行缓存存储。

### 6. `srcache_store_statuses`

```nginx
syntax: srcache_store_statuses <status1> <status2> ..
default: srcache_store_statuses 200 301 302
context: http, server, location, location if
phase: output-header-filter
```

指令用来实现根据后端状态码来决定是否进行缓存。

### 7. `srcache_store_ranges`

```nginx
syntax: srcache_store_ranges on|off
default: srcache_store_ranges off
context: http, server, location, location if
phase: output-body-filter
```

`store_ranges` 用来控制部分应答的存储，当配置为 `on` 时会对部分内容响应进行缓存存储，此时必须将 `$http_range` 添加到缓存 `key` 中。例如：

```nginx
location / {
    set $key "$uri$args$http_range";
    srcache_fetch GET /memc $key;
    srcache_store PUT /memc $key;
}
```

### 8. `srcache_header_buffer_size`

```nginx
syntax: srcache_header_buffer_size <size>
default: srcache_header_buffer_size 4k/8k
context: http, server, location, location if
phase: output-header-filter
```

控制用来接收单个响应头的缓冲区大小。

### 9. `srcache_store_hide_header`

```nginx
syntax: srcache_store_hide_header <header>
default: no
context: http, server, location, location if
phase: output-header-filter
```

用来设置不需要进行缓存的应答头。在缓存存储时，默认已经将以下响应头过滤：

```toml
Connection
Keep-Alive
Proxy-Authenticate
Proxy-Authorization
TE
Trailers
Transfer-Encoding
Upgrade
Set-Cookie
```

使用以下配置可以额外过滤掉 `X-Foo`、`Last-Modified` 响应头：

```nginx
srcache_store_hide_header X-Foo;
srcache_store_hide_header Last-Modified;
```

### 10. `srcache_store_pass_header`

```nginx
syntax: srcache_store_pass_header <header>
default: no
context: http, server, location, location if
phase: output-header-filter
```

与 `store_hide_header` 指令相反，`store_pass_header` 用来控制必须存储的响应头。

### 11. `srcache_methods`

```nginx
syntax: srcache_methods <method>...
default: srcache_methods GET HEAD
context: http, server, location
phase: post-access, output-header-filter
```

`srcache_methods` 用来控制能够进行缓存 `fetch`、`store` 的请求方法。如果请求方法未在此列表中将跳过缓存处理。`method` 取值范围：`GET`, `HEAD`, `POST`, `PUT`, `DELETE`。

### 12. `srcache_ignore_content_encoding`

```nginx
syntax: srcache_ignore_content_encoding on|off
default: srcache_ignore_content_encoding off
context: http, server, location, location if
phase: output-header-filter
```

当关闭此指令（默认设置）时，非空的 `Content-Encoding` 响应标头将导致 `srcache_store` 跳过将整个响应的缓存存储，并向 nginx 的 error.log 文件写入 `warn` 日志，如下所示

```
[warn] 12500#0: *1 srcache_store skipped due to response header "Content-Encoding: gzip"
            (maybe you forgot to disable compression on the backend?)
```

启用此指令将忽略 `Content-Encoding` 响应标头，并将其存储在缓存中（并且也不会写 `warn` 日志）。

### 13. `srcache_request_cache_control`

```nginx
syntax: srcache_request_cache_control on|off
default: srcache_request_cache_control off
context: http, server, location
phase: post-access, output-header-filter
```

`request_cache_control` 指令用来控制是否根据请求头来执行不同的缓存策略，例如是否跳过缓存查询、是否跳过缓存存储。指令开启时，在缓存获取阶段如果请求头中有 `Cache-Control: no-cache` 或者 `Pragma: no-cache`，会跳过缓存处理；在缓存存储阶段，如果请求头中有 `Cache-Control: no-store`，会跳过缓存存储处理。

关闭此指令将不关心请求头中的缓存控制指令。

### 14. `srcache_response_cache_control`

```nginx
syntax: srcache_response_cache_control on|off
default: srcache_response_cache_control on
context: http, server, location
phase: output-header-filter
```

与 `request_cache_control` 类似，`response_cache_control` 用来控制业务端应答头中缓存控制指令是否启用。当指令开启时，如果应答头中有 `Cache-Control: private|no-store|no-cache|max-age=0` 或 `Expires: 小于当前时间` 将跳过缓存处理。

**该指令优先于 `srcache_store_no_store`，`srcache_store_no_cache` 和 `srcache_store_private` 指令。**

### 15. `srcache_store_no_store`

```nginx
syntax: srcache_store_no_store on|off
default: srcache_store_no_store off
context: http, server, location
phase: output-header-filter
```

启用此指令，在满足其他缓存存储条件下，将强制具有 `Cache-Control：no-store` 应答头的响应存储在缓存中。 默认为关闭。

### 16. `srcache_store_no_cache`

```nginx
syntax: srcache_store_no_cache on|off
default: srcache_store_no_cache off
context: http, server, location
phase: output-header-filter
```

启用此指令，在满足其他缓存存储条件下，将强制具有 `Cache-Control：no-cache` 应答头的响应存储在缓存中。 默认为关闭。

### 17. `srcache_store_private`

```nginx
syntax: srcache_store_private on|off
default: srcache_store_private off
context: http, server, location
phase: output-header-filter
```

启用此指令，在满足其他缓存存储条件下，将强制具有 `Cache-Control：private` 应答头的响应存储在缓存中。 默认为关闭。

### 18. `srcache_default_expire`

```nginx
syntax: srcache_default_expire <time>
default: srcache_default_expire 60s
context: http, server, location, location if
phase: output-header-filter
```

设置缓存的默认过期时间，如果响应头中有 `Cache-Control: max-age=N` 或 `Expires` 将根据响应头设置缓存有效期。

### 19. `srcache_max_expire`

```nginx
syntax: srcache_max_expire <time>
default: srcache_max_expire 0
context: http, server, location, location if
phase: output-header-filter
```

该伪指令控制 `$srcache_expire` 变量值所允许的最大到期时间。 此设置优先于其他计算方法。

### 20. `$srcache_expire`

`srcache_expire` 变量是当前响应存储在缓存中的有效时间段（以秒为单位）。 计算值的算法如下：

- 当响应头中有 `Cache-Control: max-age=N` 头时，将使用 `N` 作为缓存有效期；
- 当响应头中有 `Expires` 时，会使用 `Expires` 指定时间与当前时间相减获得缓存有效期；
- 否则使用 `srcache_default_expire` 指令指定的有效期。

在以上三步计算完成后，会将有效期与 `srcache_max_expire` 设置值相比较，如果超过 `srcache_max_expire` 设置值，则设置为 `srcache_max_expire` 值。

## 三 实现

`srcache` 模块同时介入 `NGX_HTTP_ACCESS_PHASE` 、 `HEADER_FILTER`、`BODY_FILTER` 处理阶段。缓存的读取需要 `NGX_HTTP_ACCESS_PHASE` 阶段与 `HEADER|BODY_FILTER`阶段配合实现；缓存的设置在 `BODY_FILTER` 阶段实现。缓存的读取、设置是在一个状态循环中实现（有些过于复杂），`HEADER|BODY_FILTER` 除了给主请求使用外还给子请求使用，在其中判断缓存查找是否成功、将查找缓存拷贝到 `ctx` 中。

### 1. 注册处理函数

```c
// 注册除了函数
static ngx_int_t
ngx_http_srcache_post_config(ngx_conf_t *cf)
{
    int                              multi_http_blocks;
    ngx_int_t                        rc;
    ngx_http_handler_pt             *h;
    ngx_http_core_main_conf_t       *cmcf;
    ngx_http_srcache_main_conf_t    *smcf;

    rc = ngx_http_srcache_add_variables(cf);
    if (rc != NGX_OK) {
        return rc;
    }

    smcf = ngx_http_conf_get_module_main_conf(cf,
                                              ngx_http_srcache_filter_module);

    if (ngx_http_srcache_prev_cycle != ngx_cycle) {
        ngx_http_srcache_prev_cycle = ngx_cycle;
        multi_http_blocks = 0;
    } else {
        multi_http_blocks = 1;
    }

    if (multi_http_blocks || smcf->module_used) {

        dd("using ngx-srcache");

        /* register our output filters */
        rc = ngx_http_srcache_filter_init(cf);
        if (rc != NGX_OK) {
            return rc;
        }

        cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

        /* register our access phase handler */

        h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
        if (h == NULL) {
            return NGX_ERROR;
        }

        *h = ngx_http_srcache_access_handler;
    }

    return NGX_OK;
}
```

### 2. 读缓存上半部

```c
ngx_int_t
ngx_http_srcache_access_handler(ngx_http_request_t *r)
{
    ngx_str_t                       skip;
    ngx_int_t                       rc;
    ngx_http_srcache_loc_conf_t    *conf;
    ngx_http_srcache_main_conf_t   *smcf;
    ngx_http_srcache_ctx_t         *ctx;
    ngx_chain_t                    *cl;
    size_t                          len;
    unsigned                        no_store;

    /* access phase handlers are skipped in subrequests,
     * so the current request must be a main request */

    conf = ngx_http_get_module_loc_conf(r, ngx_http_srcache_filter_module);
    // 未配置 fetch、store 指令，不需要进行缓存处理
    if (conf->fetch == NULL && conf->store == NULL) {
        dd("bypass: %.*s", (int) r->uri.len, r->uri.data);
        return NGX_DECLINED;
    }

    dd("store defined? %p", conf->store);

    dd("req method: %lu", (unsigned long) r->method);
    dd("cache methods: %lu", (unsigned long) conf->cache_methods);

    // 请求方法与缓存允许方法不匹配，跳过缓存处理
    if (!(r->method & conf->cache_methods)) {
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "srcache_fetch and srcache_store skipped due to request "
                       "method %V", &r->method_name);

        return NGX_DECLINED;
    }

    // 启用 cache-control 指令，并且不需要进行 cache 处理
    if (conf->req_cache_control
        && ngx_http_srcache_request_no_cache(r, &no_store) == NGX_OK)
    {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "srcache_fetch skipped due to request headers "
                       "\"Cache-Control: no-cache\" or \"Pragma: no-cache\"");

        if (!no_store) {
            /* register a ctx to give a chance to srcache_store to run */

            ctx = ngx_pcalloc(r->pool,
                              sizeof(ngx_http_srcache_filter_module));

            if (ctx == NULL) {
                return NGX_ERROR;
            }

            ngx_http_set_ctx(r, ctx, ngx_http_srcache_filter_module);

        } else {
            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "srcache_store skipped due to request header "
                           "\"Cache-Control: no-store\"");
        }

        return NGX_DECLINED;
    }

    // 配置了 fetch_skip 指令，并且满足，跳过 cache lookup
    if (conf->fetch_skip != NULL
        && ngx_http_complex_value(r, conf->fetch_skip, &skip) == NGX_OK
        && skip.len
        && (skip.len != 1 || skip.data[0] != '0'))
    {
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "srcache_fetch skipped due to the true value fed into "
                       "srcache_fetch_skip: \"%V\"", &skip);

        /* register a ctx to give a chance to srcache_store to run */

        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_srcache_filter_module));

        if (ctx == NULL) {
            return NGX_ERROR;
        }

        ngx_http_set_ctx(r, ctx, ngx_http_srcache_filter_module);

        return NGX_DECLINED;
    }

    // 模块上下文
    ctx = ngx_http_get_module_ctx(r, ngx_http_srcache_filter_module);

    if (ctx != NULL) {
        /*
        if (ctx->fetch_error) {
            return NGX_DECLINED;
        }
        */

        if (ctx->waiting_subrequest) {
            dd("waiting subrequest");
            return NGX_AGAIN;
        }

        if (ctx->waiting_request_body) {
            return NGX_AGAIN;
        }

        // 使用 subrequest 进行 cache lookup
        if (ctx->request_body_done == 1) {
            ctx->request_body_done = 0;
            goto do_fetch_subrequest;
        }

        // 读取子请求应答
        if (ctx->request_done) {
            dd("request done");

            if (ngx_http_post_request(r, NULL) != NGX_OK) {
                return NGX_ERROR;
            }

            if (!ctx->from_cache) {
                return NGX_DECLINED;
            }

            dd("sending header");
            // cache hit
            // send out cached data
            if (ctx->body_from_cache) {
                len = 0;

                for (cl = ctx->body_from_cache; cl->next; cl = cl->next) {
                    len += ngx_buf_size(cl->buf);
                }

                len += ngx_buf_size(cl->buf);

                cl->buf->last_buf = 1;

                r->headers_out.content_length_n = len;

                rc = ngx_http_send_header(r);

                dd("srcache fetch header returned %d", (int) rc);

                if (rc == NGX_ERROR || rc > NGX_OK) {
                    return rc;
                }

#if 1
                if (r->header_only) {
                    return NGX_HTTP_OK;
                }
#endif

                if (!r->filter_finalize) {
                    rc = ngx_http_output_filter(r, ctx->body_from_cache);
                    if (rc == NGX_ERROR || rc > NGX_OK) {
                        return rc;
                    }
                }

                dd("sent body from cache: %d", (int) rc);
                dd("finalize from here...");

                ngx_http_finalize_request(r, rc);

                /* dd("r->main->count (post): %d", (int) r->main->count); */
                return NGX_DONE;
            }

            // cache miss
            return NGX_DECLINED;
        }

    } else {
        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_srcache_filter_module));

        if (ctx == NULL) {
            return NGX_ERROR;
        }

        ngx_http_set_ctx(r, ctx, ngx_http_srcache_filter_module);
    }

    smcf = ngx_http_get_module_main_conf(r, ngx_http_srcache_filter_module);

    if (!smcf->postponed_to_access_phase_end) {
        ngx_http_core_main_conf_t       *cmcf;
        ngx_http_phase_handler_t         tmp;
        ngx_http_phase_handler_t        *ph;
        ngx_http_phase_handler_t        *cur_ph;
        ngx_http_phase_handler_t        *last_ph;

        smcf->postponed_to_access_phase_end = 1;

        cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

        ph = cmcf->phase_engine.handlers;
        cur_ph = &ph[r->phase_handler];

        /* we should skip the post_access phase handler here too */
        last_ph = &ph[cur_ph->next - 2];

        if (cur_ph < last_ph) {
            dd("swaping the contents of cur_ph and last_ph...");

            tmp = *cur_ph;

            memmove(cur_ph, cur_ph + 1,
                    (last_ph - cur_ph) * sizeof (ngx_http_phase_handler_t));

            *last_ph = tmp;

            r->phase_handler--; /* redo the current ph */

            return NGX_DECLINED;
        }
    }

    // 未定义 fetch 操作，跳过 cache lookup
    if (conf->fetch == NULL) {
        dd("fetch is not defined");
        return NGX_DECLINED;
    }

    dd("running phase handler...");
    // 读入请求包体
    if (!r->request_body) {
        dd("reading request body: ctx = %p", ctx);

        rc = ngx_http_read_client_request_body(r,
                                               ngx_http_srcache_post_read_body);
        if (rc == NGX_ERROR || rc > NGX_OK) {
#if (nginx_version < 1002006)                                               \
    || (nginx_version >= 1003000 && nginx_version < 1003009)
            r->main->count--;
#endif
            return rc;
        }

        // 设置等待读入请求包体标记
        if (rc == NGX_AGAIN) {
            ctx->waiting_request_body = 1;
            return NGX_AGAIN;
        }

        /* rc == NGX_OK */
    }

do_fetch_subrequest:

    /* issue a subrequest to fetch cached stuff (if any) */
    // 发起缓存查询子请求
    rc = ngx_http_srcache_fetch_subrequest(r, conf, ctx);

    if (rc != NGX_OK) {
        return rc;
    }

    ctx->waiting_subrequest = 1;

    dd("quit");

    return NGX_AGAIN;
}
```

### 3. 读缓存下半部与设置缓存

```c
static ngx_int_t
ngx_http_srcache_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_http_srcache_ctx_t      *ctx, *pr_ctx;
    ngx_int_t                    rc;
    ngx_str_t                    skip;
    ngx_chain_t                 *cl;
    ngx_http_srcache_loc_conf_t *slcf;
    size_t                       len;
    unsigned                     last;

    dd_enter();

    if (in == NULL) {
        return ngx_http_srcache_next_body_filter(r, NULL);
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_srcache_filter_module);

    if (ctx == NULL || ctx->from_cache || ctx->store_skip) {
        dd("bypass: %.*s", (int) r->uri.len, r->uri.data);
        return ngx_http_srcache_next_body_filter(r, in);
    }

    if (ctx->ignore_body || ctx->in_store_subrequest/* || ctx->fetch_error */) {
        dd("ignore body: ignore body %d, in store sr %d",
           (int) ctx->ignore_body, (int) ctx->in_store_subrequest);
        ngx_http_srcache_discard_bufs(r->pool, in);
        return NGX_OK;
    }

    // 缓存读取操作
    // body_filter 处理
    if (ctx->in_fetch_subrequest) {
        
        // header 处理
        if (ctx->parsing_cached_headers) {

            /* parse the cached response's headers and
             * set r->parent->headers_out */

            if (ctx->process_header == NULL) {
                dd("restore parent request header");
                ctx->process_header = ngx_http_srcache_process_status_line;
                r->state = 0; /* sw_start */
            }

            for (cl = in; cl; cl = cl->next) {
                if (ngx_buf_in_memory(cl->buf)) {
                    dd("old pos %p, last %p", cl->buf->pos, cl->buf->last);

                    rc = ctx->process_header(r, cl->buf);

                    if (rc == NGX_AGAIN) {
                        dd("AGAIN/OK: new pos %p, last %p",
                           cl->buf->pos, cl->buf->last);

                        continue;
                    }

                    // 处理 header 出错，忽略包体处理，设置缓存未空
                    if (rc == NGX_ERROR) {
                        r->state = 0; /* sw_start */
                        ctx->parsing_cached_headers = 0;
                        ctx->ignore_body = 1;
                        ngx_http_srcache_discard_bufs(r->pool, cl);
                        pr_ctx = ngx_http_get_module_ctx(r->parent,
                                              ngx_http_srcache_filter_module);

                        if (pr_ctx == NULL) {
                            return NGX_ERROR;
                        }

                        pr_ctx->from_cache = 0;

                        return NGX_OK;
                    }

                    /* rc == NGX_OK */

                    dd("OK: new pos %p, last %p", cl->buf->pos, cl->buf->last);
                    dd("buf left: %.*s", (int) (cl->buf->last - cl->buf->pos),
                       cl->buf->pos);

                    ctx->parsing_cached_headers = 0;

                    break;
                }
            }

            if (cl == NULL) {
                return NGX_OK;
            }

            if (cl->buf->pos == cl->buf->last) {
                cl = cl->next;
            }

            if (cl == NULL) {
                return NGX_OK;
            }

            // header 处理结束，更新 in 指向 body
            in = cl;
        }

        dd("save the cached response body for parent");

        pr_ctx = ngx_http_get_module_ctx(r->parent,
                                         ngx_http_srcache_filter_module);

        if (pr_ctx == NULL) {
            return NGX_ERROR;
        }

        // 将 body 拷贝到 body_from_cache
        rc = ngx_http_srcache_add_copy_chain(r->pool,
                                             &pr_ctx->body_from_cache, in,
                                             &last);

        if (rc != NGX_OK) {
            return NGX_ERROR;
        }

        if (last) {
            ctx->seen_subreq_eof = 1;
        }

        // 删除 in
        ngx_http_srcache_discard_bufs(r->pool, in);

        return NGX_OK;
    }

    // 缓存存储操作
    if (ctx->store_response) {
        dd("storing the response: %p", in);

        slcf = ngx_http_get_module_loc_conf(r, ngx_http_srcache_filter_module);

        if (r->headers_out.status == NGX_HTTP_PARTIAL_CONTENT
            && ctx->http_status == NGX_HTTP_OK)
        {
            u_char *p;

            if (!slcf->store_ranges) {
                ctx->store_response = 0;
                goto done;
            }

            dd("fix 206 status code");

            /* handle 206 Partial Content generated by the range filter */
            cl = ctx->body_to_cache;
            assert(cl && cl->buf && cl->buf->last - cl->buf->pos > 12);
            p = cl->buf->pos + sizeof("HTTP/1.x 20") - 1;
            *p = '6';

            ctx->http_status = NGX_HTTP_PARTIAL_CONTENT;
        }

        for (cl = in; cl; cl = cl->next) {
            if (ngx_buf_in_memory(cl->buf)) {
                len = ngx_buf_size(cl->buf);
                ctx->response_length += len;
                ctx->response_body_length += len;
            }
        }

        if (slcf->store_max_size != 0
            && ctx->response_length > slcf->store_max_size)
        {
            ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "srcache_store bypassed because response body "
                           "exceeded maximum size: %z (limit is: %z)",
                           ctx->response_length, slcf->store_max_size);

            ctx->store_response = 0;

            goto done;
        }

        rc = ngx_http_srcache_add_copy_chain(r->pool, &ctx->body_to_cache,
                                             in, &last);

        if (rc != NGX_OK) {
            ctx->store_response = 0;
            goto done;
        }

        if (last && r == r->main) {

#if 1
            if (r->headers_out.content_length_n >
                (off_t) ctx->response_body_length)
            {
                ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                              "srcache_store: skipped because response body "
                              "truncated: %O > %uz",
                              r->headers_out.content_length_n,
                              ctx->response_body_length);

                ctx->store_response = 0;
                goto done;
            }

            if (r->headers_out.status >= NGX_HTTP_SPECIAL_RESPONSE
                && r->headers_out.status != ctx->http_status)
            {
                /* data truncation or body receive timeout */

                ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                              "srcache_store: skipped due to new error status "
                              "code %ui (old: %ui)",
                              r->headers_out.status, ctx->http_status);

                ctx->store_response = 0;
                goto done;
            }
#endif

            if (slcf->store_skip != NULL
                && ngx_http_complex_value(r, slcf->store_skip, &skip) == NGX_OK
                && skip.len
                && (skip.len != 1 || skip.data[0] != '0'))
            {
                ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                               "srcache_store skipped due to the true value in "
                               "srcache_store_skip: \"%V\"", &skip);

                ctx->store_response = 0;
                goto done;
            }

            rc = ngx_http_srcache_store_subrequest(r, ctx);

            if (rc != NGX_OK) {
                ctx->store_response = 0;
                goto done;
            }
        }

    } else {
        dd("NO store response");
    }

done:

    return ngx_http_srcache_next_body_filter(r, in);
}
```

