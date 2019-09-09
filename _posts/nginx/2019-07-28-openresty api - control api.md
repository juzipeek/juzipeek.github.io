---
layout: post
title: OpenResty Api - control api
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description: 
---

## 一 概述

`ngx.redirect` `ngx.exec` `ngx.throw_error` `ngx.exit` `ngx.on_abort`

## 二 函数

### 1. `ngx.redirect`

```lua
ngx.redirect(uri, status?)
```

可介入的阶段：`rewrite_by_lua`, `access_by_lua`, `content_by_lua`

发出重定向应答，跳转到 `uri` 参数指定的 `uri`， `HTTP` 状态码默认为 `302`，可以使用可选参数 `status` 指定状态码。`status` 可选值为：`301`、`302`、`303`、`307`、`308`。**函数调用会终止请求处理**。

函数实现：

```c
static int
ngx_http_lua_ngx_redirect(lua_State *L)
{
    ngx_http_lua_ctx_t          *ctx;
    ngx_int_t                    rc;
    int                          n;
    u_char                      *p;
    u_char                      *uri;
    size_t                       len;
    ngx_table_elt_t             *h;
    ngx_http_request_t          *r;

    n = lua_gettop(L);

    if (n != 1 && n != 2) {
        return luaL_error(L, "expecting one or two arguments");
    }

    p = (u_char *) luaL_checklstring(L, 1, &len);

    if (n == 2) {
        rc = (ngx_int_t) luaL_checknumber(L, 2);

        if (rc != NGX_HTTP_MOVED_TEMPORARILY
            && rc != NGX_HTTP_MOVED_PERMANENTLY
            && rc != NGX_HTTP_SEE_OTHER
            && rc != NGX_HTTP_PERMANENT_REDIRECT
            && rc != NGX_HTTP_TEMPORARY_REDIRECT)
        {
            return luaL_error(L, "only ngx.HTTP_MOVED_TEMPORARILY, "
                              "ngx.HTTP_MOVED_PERMANENTLY, "
                              "ngx.HTTP_PERMANENT_REDIRECT, "
                              "ngx.HTTP_SEE_OTHER, and "
                              "ngx.HTTP_TEMPORARY_REDIRECT are allowed");
        }

    } else {
        rc = NGX_HTTP_MOVED_TEMPORARILY;
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no request ctx found");
    }

    // 阶段检查
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT);

    // 如果子请求未应答则不能调用 redirect 函数
    ngx_http_lua_check_if_abortable(L, ctx);

    if (r->header_sent || ctx->header_sent) {
        return luaL_error(L, "attempt to call ngx.redirect after sending out "
                          "the headers");
    }

    uri = ngx_palloc(r->pool, len);
    if (uri == NULL) {
        return luaL_error(L, "no memory");
    }

    ngx_memcpy(uri, p, len);

    // 添加 Location 头，值为 uri
    h = ngx_list_push(&r->headers_out.headers);
    if (h == NULL) {
        return luaL_error(L, "no memory");
    }

    h->hash = ngx_http_lua_location_hash;

#if 0
    dd("location hash: %lu == %lu",
       (unsigned long) h->hash,
       (unsigned long) ngx_hash_key_lc((u_char *) "Location",
                                       sizeof("Location") - 1));
#endif

    h->value.len = len;
    h->value.data = uri;
    ngx_str_set(&h->key, "Location");

    r->headers_out.status = rc;

    ctx->exit_code = rc;
    ctx->exited = 1;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua redirect to \"%V\" with code %i",
                   &h->value, ctx->exit_code);

    if (len && uri[0] != '/') {
        r->headers_out.location = h;
    }

    /*
     * we do not set r->headers_out.location here to avoid the handling
     * the local redirects without a host name by ngx_http_header_filter()
     */

    return lua_yield(L, 0);
}
```

### 2. `ngx.exec`

```lua
ngx.exec(uri, args?)
```

可介入的阶段：`rewrite_by_lua`, `access_by_lua`, `content_by_lua`

执行内部跳转功能，跳转到 `uri`，使用可选参数 `args` 可以传递 `uri` 参数，`args` 可以是 `string` 也可以说是 `table` 类型。支持使用命名 `location` 作为 `uri` 参数，但是 `args` 参数会被忽略。`ngx.exec` 函数功能由 `ngx_http_lua_ngx_exec` 函数实现，但是在 `ngx_http_lua_ngx_exec` 中处理成功后仅设置了 `ctx[ngx_http_lua_module]->exec_uri`、`ctx[ngx_http_lua_module]->exec_args` 变量，在 `ngx_http_lua_run_thread` 函数中检测 `exec_uri` 被设置会调用 `ngx_http_lua_handle_exec` 函数进行内部跳转。对于命名 `location` 跳转会直接跳转到 `NGX_HTTP_POST_REWRITE_PHASE` 阶段进行处理(设置的索引是 `NGX_HTTP_REWRITE_PHASE` ，会执行下个阶段的处理)，不会经过 `NGX_HTTP_FIND_CONFIG_PHASE` 阶段进行 `location` 查找，因此需要在 `server{}` 内查找好 `location` 并对请求进行更新。对于非命令 `location` 的内部跳转，会更新当前请求的 `uri`，跳转到 `NGX_HTTP_FIND_CONFIG_PHASE` 阶段进行处理。

### 3. `ngx.exit`

```lua
ngx.exit(status)
```

可介入的阶段：`rewrite_by_lua*`, `access_by_lua*`, `content_by_lua*`,`header_filter_by_lua*`, `ngx.timer.*`, `balancer_by_lua*`, `ssl_certificate_by_lua*`, `ssl_session_fetch_by_lua*`, `ssl_session_store_by_lua`

`status >= 200` 会中断当前请求的执行，并将 `status` 作为应答状态码返回，会经过后续的 `header_filter`、`body_filter` 阶段。

`status == 0` 将只会跳过当前阶段的处理函数，继续执行后续阶段的处理函数。

`status` 取值范围是[`HTTP` 常量状态码](https://github.com/openresty/lua-nginx-module#http-status-constants)以及 `ngx.OK`、`ngx.ERROR`。

`ngx_http_lua_ngx_exit` 是 `ngx.exit` 的实现函数，在其中仅将 `status` 设置到 `ctx->exit_code` 中，同时设置 `exited` 标记。在 `ngx_http_lua_run_thread` 中检测到 `exited` 标记后会调用 `ngx_http_lua_handle_exit` 进行处理。

**TODO：`ngx_http_lua_handle_exit` 、`ngx_http_lua_run_thread` 继续跟进**

### 4. `ngx.on_abort`

```lua
ok, err = ngx.on_abort(callback)
```

可介入的阶段：`rewrite_by_lua*`、`access_by_lua*`、`content_by_lua*`

将函数 `callback` 注册为回调函数，当客户端过早关闭（下游）连接时，将自动调用该函数。调用成功返回值为 `1` 否则返回 `nil`。回调函数可以决定如何处理客户端的提前关闭事件，可以通过不执行任何操作来简单忽略该事件，并且当前的请求处理程序将继续执行而不会中断。回调函数可以调用 `ngx.exit` 来终止请求处理。

当 `lua_check_client_abort` 指令配置为 `off` 时调用 `ngx.on_abort` 会返回 `lua_check_client_abort is off` 错误提示。

**在一个请求中，此函数只能注册一次**。

在实现函数 `ngx_http_lua_on_abort` 中会创建 `light thread`：

```c
static int
ngx_http_lua_on_abort(lua_State *L)
{
    ngx_http_lua_ctx_t           *ctx;
    ngx_http_lua_co_ctx_t        *coctx = NULL;
		// ... 省略其他代码
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no request ctx found");
    }

    ngx_http_lua_check_fake_request2(L, r, ctx);
		// 已经创建过协程
    if (ctx->on_abort_co_ctx) {
        lua_pushnil(L);
        lua_pushliteral(L, "duplicate call");
        return 2;
    }

		// 关闭 lua_check_client_abort 配置
    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);
    if (!llcf->check_client_abort) {
        lua_pushnil(L);
        lua_pushliteral(L, "lua_check_client_abort is off");
        return 2;
    }

		// 使用回调函数创建协程
    ngx_http_lua_coroutine_create_helper(L, r, ctx, &coctx);

    lua_pushlightuserdata(L, &ngx_http_lua_coroutines_key);
    lua_rawget(L, LUA_REGISTRYINDEX);
    lua_pushvalue(L, -2);

    dd("on_wait thread 1: %p", lua_tothread(L, -1));

    coctx->co_ref = luaL_ref(L, -2);
    lua_pop(L, 1);

    coctx->is_uthread = 1;
    // 保存创建的协程
    ctx->on_abort_co_ctx = coctx;

    dd("on_wait thread 2: %p", coctx->co);
		// 协程状态挂起
    coctx->co_status = NGX_HTTP_LUA_CO_SUSPENDED;
    coctx->parent_co_ctx = ctx->cur_co_ctx;

    lua_pushinteger(L, 1);
    return 1;
}
```

在 `NGINX` 事件处理中，当检测到客户端连接关闭时会调用 `ngx_http_lua_rd_check_broken_connection` 函数，会将 `on_abort` 创建的协程设置为处理协程。

```c
void
ngx_http_lua_rd_check_broken_connection(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_event_t                *rev;
    ngx_http_lua_ctx_t         *ctx;

    if (r->done) {
        return;
    }

    rc = ngx_http_lua_check_broken_connection(r, r->connection->read);

    if (rc == NGX_OK) {
        return;
    }

    /* rc == NGX_ERROR || rc > NGX_OK */

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return;
    }
    // 未设置 on_abort 函数
    if (ctx->on_abort_co_ctx == NULL) {
        r->connection->error = 1;
        ngx_http_lua_request_cleanup(ctx, 0);
        ngx_http_lua_finalize_request(r, rc);
        return;
    }

   // 协程状态必定是 SUSPENDED
    if (ctx->on_abort_co_ctx->co_status != NGX_HTTP_LUA_CO_SUSPENDED) {

        /* on_abort already run for the current request handler */

        rev = r->connection->read;

        if ((ngx_event_flags & NGX_USE_LEVEL_EVENT) && rev->active) {
            if (ngx_del_event(rev, NGX_READ_EVENT, 0) != NGX_OK) {
                ngx_http_lua_request_cleanup(ctx, 0);
                ngx_http_lua_finalize_request(r,
                                              NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }
        }

        return;
    }

    ctx->uthreads++;
    ctx->resume_handler = ngx_http_lua_on_abort_resume;
    ctx->on_abort_co_ctx->co_status = NGX_HTTP_LUA_CO_RUNNING;
    // 处理协程是 on_abort 协程
    ctx->cur_co_ctx = ctx->on_abort_co_ctx;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua waking up the on_abort callback thread");

    if (ctx->entered_content_phase) {
        r->write_event_handler = ngx_http_lua_content_wev_handler;

    } else {
        r->write_event_handler = ngx_http_core_run_phases;
    }
		// 触发写事件处理
    r->write_event_handler(r);
}
```

