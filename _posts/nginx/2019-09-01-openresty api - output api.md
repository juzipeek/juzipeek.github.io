---
layout: post
title: OpenResty Api - output api
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description:
---

## 一 概述

`output api` 主要用于向**客户端**发送应答数据，或者对应答进行控制。输出函数：`ngx.send_headers`、`ngx.print`、`ngx.say` ；响应控制函数：`ngx.flush`、`ngx.eof`。

## 二 输出函数

### 1. `ngx.send_headers`

函数原型：

```lua
ok, err = ngx.send_headers()
```

允许介入的阶段：`rewrite_by_lua_*`、`access_by_lua_*`、`content_by_lua_*`。

向客户端发送应答头。在发送响应时需要对 `content type`、`content length` 进行特别关注，如果这两者错误会直接导致客户端解析异常。`lua_nginx_module` 会调用 `ngx_http_lua_set_content_type` 函数对 `content type` 进行特殊处理。在对 `conten_type` 进行特殊处理时最终会调用 `ngx_http_set_content_type` 函数，首先根据请求文件扩展名获得对于的 `content type`，如果失败则使用默认 `content type` 设置。其实现如下：

```c
ngx_int_t
ngx_http_set_content_type(ngx_http_request_t *r)
{
    u_char                     c, *exten;
    ngx_str_t                 *type;
    ngx_uint_t                 i, hash;
    ngx_http_core_loc_conf_t  *clcf;

    if (r->headers_out.content_type.len) {
        return NGX_OK;
    }

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    // 请求文件的扩展名，例如请求行：GET /index.html HTTP/1.1，扩展名为 html
    if (r->exten.len) {

        hash = 0;

        // 将扩展名转换为小写计算 hash key
        for (i = 0; i < r->exten.len; i++) {
            c = r->exten.data[i];

            if (c >= 'A' && c <= 'Z') {

                exten = ngx_pnalloc(r->pool, r->exten.len);
                if (exten == NULL) {
                    return NGX_ERROR;
                }

                hash = ngx_hash_strlow(exten, r->exten.data, r->exten.len);

                r->exten.data = exten;

                break;
            }
            // ngx_hash 会累加 hash 值
            hash = ngx_hash(hash, c);
        }

        // 根据 mime 表查找 conten type
        type = ngx_hash_find(&clcf->types_hash, hash,
                             r->exten.data, r->exten.len);

        if (type) {
            r->headers_out.content_type_len = type->len;
            r->headers_out.content_type = *type;

            return NGX_OK;
        }
    }

    // 设置默认 content type
    r->headers_out.content_type_len = clcf->default_type.len;
    r->headers_out.content_type = clcf->default_type;

    return NGX_OK;
}
```

对于 `content_length` 头，如果未设置过应答 `header`（调用 `ngx.resp.header['new_header'] = 'value'` 函数），需要将 `content_length` 头清除，使用 `chrunked` 编码方式传输。

**调用 `ngx.send_headers` 后会进入 `header filter` 阶段，此函数的实际输出是调用 `ngx_http_top_header_filter`**。

### 2. `ngx.print`

函数原型：

```lua
ok, err = ngx.print(...)
```

允许介入阶段：`rewrite_by_lua_*`、`access_by_lua_*`、`content_by_lua_*`。

`ngx.print` 会调用 `ngx_http_lua_ngx_echo` 函数进行应答 `ngx_http_top_body_filter`

```c
static int
ngx_http_lua_ngx_echo(lua_State *L, unsigned newline)
{
    ngx_http_request_t          *r;
    ngx_http_lua_ctx_t          *ctx;
    const char                  *p;
    size_t                       len;
    size_t                       size;
    ngx_buf_t                   *b;
    ngx_chain_t                 *cl;
    ngx_int_t                    rc;
    int                          i;
    int                          nargs;
    int                          type;
    const char                  *msg;

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    if (ctx == NULL) {
        return luaL_error(L, "no request ctx found");
    }
		// 当前阶段检测，判断是否在 rewrite、access、content 阶段
    // NGX_HTTP_LUA_CONTEXT_REWRITE 等几个宏是 lua-nginx 模块定义
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT);

    if (ctx->acquired_raw_req_socket) {
        lua_pushnil(L); // 状态标识
        lua_pushliteral(L, "raw request socket acquired"); // 错误信息字符串
        return 2; // 栈中元素数，返回给 lua return value 数量
    }

    if (r->header_only) {
        lua_pushnil(L); // 状态标识
        lua_pushliteral(L, "header only"); // 错误信息字符串
        return 2; // 栈中元素数，返回给 lua return value 数量
    }

    if (ctx->eof) {
        lua_pushnil(L); // 状态标识
        lua_pushliteral(L, "seen eof"); // 错误信息
        return 2;
    }

    nargs = lua_gettop(L);
    size = 0;

    // 计算 ngx.print 或 ngx.say 的参数长度
    for (i = 1; i <= nargs; i++) {

        type = lua_type(L, i);

        switch (type) {
            case LUA_TNUMBER:
            case LUA_TSTRING:

                lua_tolstring(L, i, &len);
                size += len;
                break;

            case LUA_TNIL:

                size += sizeof("nil") - 1;
                break;

            case LUA_TBOOLEAN:

                if (lua_toboolean(L, i)) {
                    size += sizeof("true") - 1;

                } else {
                    size += sizeof("false") - 1;
                }

                break;

            case LUA_TTABLE:

                size += ngx_http_lua_calc_strlen_in_table(L, i, i,
                                                          0 /* strict */);
                break;

            case LUA_TLIGHTUSERDATA:

                dd("userdata: %p", lua_touserdata(L, i));

                if (lua_touserdata(L, i) == NULL) {
                    size += sizeof("null") - 1;
                    break;
                }

                continue;

            default:

                msg = lua_pushfstring(L, "string, number, boolean, nil, "
                                      "ngx.null, or array table expected, "
                                      "but got %s", lua_typename(L, type));

                return luaL_argerror(L, i, msg);
        }
    }
    // ngx.say newline 为 1；ngx.print newline 为 0
    if (newline) {
        size += sizeof("\n") - 1;
    }

    // 无 body，仅发送 header
    if (size == 0) {
        rc = ngx_http_lua_send_header_if_needed(r, ctx);
        if (rc == NGX_ERROR || rc > NGX_OK) {
            lua_pushnil(L);
            lua_pushliteral(L, "nginx output filter error");
            return 2;
        }

        lua_pushinteger(L, 1);
        return 1;
    }

    ctx->seen_body_data = 1;

    // 创建 size 大小的 ngx_chain_t 对象
    cl = ngx_http_lua_chain_get_free_buf(r->connection->log, r->pool,
                                         &ctx->free_bufs, size);

    if (cl == NULL) {
        return luaL_error(L, "no memory");
    }

    b = cl->buf;
    // 将调用参数进行转换，并拷贝的 output_chain
    for (i = 1; i <= nargs; i++) {
        type = lua_type(L, i);
        switch (type) {
            case LUA_TNUMBER:
            case LUA_TSTRING:
                p = lua_tolstring(L, i, &len);
                b->last = ngx_copy(b->last, (u_char *) p, len);
                break;

            case LUA_TNIL:
                *b->last++ = 'n';
                *b->last++ = 'i';
                *b->last++ = 'l';
                break;

            case LUA_TBOOLEAN:
                if (lua_toboolean(L, i)) {
                    *b->last++ = 't';
                    *b->last++ = 'r';
                    *b->last++ = 'u';
                    *b->last++ = 'e';

                } else {
                    *b->last++ = 'f';
                    *b->last++ = 'a';
                    *b->last++ = 'l';
                    *b->last++ = 's';
                    *b->last++ = 'e';
                }

                break;

            case LUA_TTABLE:
                b->last = ngx_http_lua_copy_str_in_table(L, i, b->last);
                break;

            case LUA_TLIGHTUSERDATA:
                *b->last++ = 'n';
                *b->last++ = 'u';
                *b->last++ = 'l';
                *b->last++ = 'l';
                break;

            default:
                return luaL_error(L, "impossible to reach here");
        }
    }

    if (newline) {
        *b->last++ = '\n';
    }

#if 0
    if (b->last != b->end) {
        return luaL_error(L, "buffer error: %p != %p", b->last, b->end);
    }
#endif

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   newline ? "lua say response" : "lua print response");

    // 最终会调用 ngx_http_top_body_filter 进行响应
    rc = ngx_http_lua_send_chain_link(r, ctx, cl);

    if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        lua_pushnil(L);
        lua_pushliteral(L, "nginx output filter error");
        return 2;
    }

    dd("downstream write: %d, buf len: %d", (int) rc,
       (int) (b->last - b->pos));

    lua_pushinteger(L, 1);
    return 1;
}
```

### 3. `ngx.say`

函数原型：

```lua
ok, err = ngx.say(...)
```

与 `ngx.print` 功能相同，最终会调用 `ngx_http_lua_ngx_echo` 函数，只不过 `ngx.say` 调用时 `newline` 参数为 `1`。

### 4. `ngx.flush`

函数原型：

```lua
ok, err = ngx.flush(wait?)
```

允许介入阶段：`rewrite_by_lua_*`、`access_by_lua_*`、`content_by_lua_*`。

如果存在可选 `wait` 参数，并且值为 `true` 函数调用会等待发送结束或超时才返回（HTTP/1.0 buffing 未在此情况），否则立即返回。`ngx.flush` 实现函数 `ngx_http_lua_ngx_flush` 主要实现两个功能：`body filter` 链对应答数据进行处理、将应答数据发送出去，函数实现如下：

```c
static int
ngx_http_lua_ngx_flush(lua_State *L)
{
    ngx_http_request_t          *r;
    ngx_http_lua_ctx_t          *ctx;
    ngx_chain_t                 *cl;
    ngx_int_t                    rc;
    int                          n;
    unsigned                     wait = 0;
    ngx_event_t                 *wev;
    ngx_http_core_loc_conf_t    *clcf;
    ngx_http_lua_co_ctx_t       *coctx;

    n = lua_gettop(L);
    if (n > 1) {
        return luaL_error(L, "attempt to pass %d arguments, but accepted 0 "
                          "or 1", n);
    }

    r = ngx_http_lua_get_req(L);

    if (n == 1 && r == r->main) {
        luaL_checktype(L, 1, LUA_TBOOLEAN);
        wait = lua_toboolean(L, 1);
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no request ctx found");
    }

    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT);

    // 
    if (ctx->acquired_raw_req_socket) {
        lua_pushnil(L);
        lua_pushliteral(L, "raw request socket acquired");
        return 2;
    }

    coctx = ctx->cur_co_ctx;
    if (coctx == NULL) {
        return luaL_error(L, "no co ctx found");
    }

    if (r->header_only) {
        lua_pushnil(L);
        lua_pushliteral(L, "header only");
        return 2;
    }

    // 请求已经关闭
    if (ctx->eof) {
        lua_pushnil(L);
        lua_pushliteral(L, "seen eof");
        return 2;
    }

    // 如果是 buffing 则 flush 无效
    if (ctx->buffering) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua http 1.0 buffering makes ngx.flush() a no-op");

        lua_pushnil(L);
        lua_pushliteral(L, "buffering");
        return 2;
    }

#if 1
    // header 已经发出、无待发送包体
    if ((!r->header_sent && !ctx->header_sent)
        || (!ctx->seen_body_data && !wait))
    {
        lua_pushnil(L);
        lua_pushliteral(L, "nothing to flush");
        return 2;
    }
#endif

    // 使用空闲或新建 chain，并设置 flush 标记位；其中无待发送数据
    cl = ngx_http_lua_get_flush_chain(r, ctx);
    if (cl == NULL) {
        return luaL_error(L, "no memory");
    }

    // 发送应答动作，经过 body filter 处理
    // 此时数据并未发送到客户端，仍然在 nginx 中，保存在 cl 中
    rc = ngx_http_lua_send_chain_link(r, ctx, cl);

    dd("send chain: %d", (int) rc);

    if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        lua_pushnil(L);
        lua_pushliteral(L, "nginx output filter error");
        return 2;
    }

    dd("wait:%d, rc:%d, buffered:0x%x", wait, (int) rc,
       r->connection->buffered);

    wev = r->connection->write;

    // 同步写操作
    // 等待所有的数据写到客户端后才返回，并不会阻塞请求
    if (wait && (r->connection->buffered & NGX_HTTP_LOWLEVEL_BUFFERED
                 || wev->delayed))
    {
        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua flush requires waiting: buffered 0x%uxd, "
                       "delayed:%d", (unsigned) r->connection->buffered,
                       wev->delayed);

        coctx->flushing = 1;
        ctx->flushing_coros++;

        if (ctx->entered_content_phase) {
            /* mimic ngx_http_set_write_handler */
            r->write_event_handler = ngx_http_lua_content_wev_handler;

        } else {
            r->write_event_handler = ngx_http_core_run_phases;
        }

        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

        if (!wev->delayed) {
            ngx_add_timer(wev, clcf->send_timeout);
        }

        if (ngx_handle_write_event(wev, clcf->send_lowat) != NGX_OK) {
            if (wev->timer_set) {
                wev->delayed = 0;
                ngx_del_timer(wev);
            }

            lua_pushnil(L);
            lua_pushliteral(L, "connection broken");
            return 2;
        }

        ngx_http_lua_cleanup_pending_operation(ctx->cur_co_ctx);
        coctx->cleanup = ngx_http_lua_flush_cleanup;
        coctx->data = r;

        return lua_yield(L, 0);
    }

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua flush asynchronously");
    
    // 异步发送，不等待发送结束，仅设置 chain 的 flush 标记位
    lua_pushinteger(L, 1);
    return 1;
}
```

在 `ngx.print`、`ngx.say`、`ngx.flush` 函数实现中都会调用 `ngx_http_lua_send_chain_link` 函数，用于将应答数据经过 `header filter`、`body filter` 处理。

```c
// 将应答数据经过 body filter 处理
ngx_int_t
ngx_http_lua_send_chain_link(ngx_http_request_t *r, ngx_http_lua_ctx_t *ctx,
    ngx_chain_t *in)
{
    ngx_int_t                     rc;
    ngx_chain_t                  *cl;
    ngx_chain_t                 **ll;
    ngx_http_lua_loc_conf_t      *llcf;

#if 1
    if (ctx->acquired_raw_req_socket || ctx->eof) {
        dd("ctx->eof already set or raw req socket already acquired");
        return NGX_OK;
    }
#endif

    if ((r->method & NGX_HTTP_HEAD) && !r->header_only) {
        r->header_only = 1;
    }

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    if (llcf->http10_buffering
        && !ctx->buffering
        && !r->header_sent
        && !ctx->header_sent
        && r->http_version < NGX_HTTP_VERSION_11
        && r->headers_out.content_length_n < 0)
    {
        ctx->buffering = 1;
    }

    rc = ngx_http_lua_send_header_if_needed(r, ctx);

    if (rc == NGX_ERROR || rc > NGX_OK) {
        return rc;
    }

    // 仅需要发送 header，无 body
    if (r->header_only) {
        ctx->eof = 1;

        if (ctx->buffering) {
            return ngx_http_lua_send_http10_headers(r, ctx);
        }

        return rc;
    }

    if (in == NULL) {
        dd("last buf to be sent");

#if 1
        if (!r->request_body && r == r->main) {
            if (ngx_http_discard_request_body(r) != NGX_OK) {
                return NGX_ERROR;
            }
        }
#endif

        if (ctx->buffering) {
            rc = ngx_http_lua_send_http10_headers(r, ctx);
            if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
                return rc;
            }

            if (ctx->out) {

                rc = ngx_http_lua_output_filter(r, ctx->out);

                if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
                    return rc;
                }

                ctx->out = NULL;
            }
        }

#if defined(nginx_version) && nginx_version <= 8004

        /* earlier versions of nginx does not allow subrequests
           to send last_buf themselves */
        if (r != r->main) {
            return NGX_OK;
        }

#endif

        ctx->eof = 1;

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua sending last buf of the response body");

        rc = ngx_http_lua_send_special(r, NGX_HTTP_LAST);

        if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
            return rc;
        }

        return NGX_OK;
    }

    /* in != NULL */
    // 存在 buffing 数据，将待发送数据放在最后
    if (ctx->buffering) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua buffering output bufs for the HTTP 1.0 request");

        // 将 ctx 中待发送数据保存到 ngx_chain_t 中
        for (cl = ctx->out, ll = &ctx->out; cl; cl = cl->next) {
            ll = &cl->next;
        }

        *ll = in;

        return NGX_OK;
    }
		// 调用 ngx_http_output_filter 进行 body filter 处理
    return ngx_http_lua_output_filter(r, in);
}
```

### 5. `ngx.eof`

函数原型：

```lua
ok, err = ngx.eof()
```

允许介入阶段：`rewrite_by_lua_*`、`access_by_lua_*`、`content_by_lua_*`。

指示响应流结束，如果是 `HTTP/1.1` 版本 `chunked` 编码格式，将触发发送最后一个 `chunk`。如果未启用 `HTTP/1.1` 的 `keep-alive` 特性，调用 `ngx.eof` 后客户端应该关闭连接。**`ngx.eof` 不会关闭连接，只会调用 `header filter`、`body filter` 进行应答处理**。

在 `ngx.eof` 实现中同样会调用 `ngx_http_lua_send_chain_link`。



## 三 HTTP/1.0 buffing

`HTTP/1.0` 不支持 `chunked` 编码，当响应 `body` 不为空时需要 `content-length` 头以便支持 `HTTP/1.0` 的 `keep-alive` 特性。因此当发起 `HTTP/1.0` 协议请求并且打开了 `lua_http10_buffing` 配置时，`lua-nginx` 模块会将 `ngx.say`、`ngx.print` 的输出内容缓存（同时会将应答 `header` 延时发送），直到收到所有的应答 `body`。此时，`lua-nginx` 模块就能计算出 `content-length` 并发送给客户端。如果在应答头中有 `content-length` 头，即使打开了 `lua_http10_buffing` 也不会缓存应答信息。

对于大的响应流，要注意关闭 `lua_http10_buffing` 指令，避免内存占用过高。

### 1. `lua_http10_buffering` 指令

```
syntax: lua_http10_buffering on|off
default: lua_http10_buffering on
context: http, server, location, location-if
```

为 `HTTP/1.0` (或更老的)请求启用或禁用自动响应缓存，这种缓冲机制主要用于低于 `HTTP/1.0` 版本的 `keep-alive` 特性，它依赖于正确的 `content-length` 响应头。

如果 `lua` 代码在发送响应头（通过调用 `ngx.send_headers`、`ngx.say`、`ngx.print` 触发发送响应头）之前已经显示的设置 `content-length` 头，`http10_buffering` 特性将被关闭。
