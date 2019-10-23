---
layout: post
title: OpenResty Api - socket tcp
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description:
---

## 一 概述

`cosocket` 提供了非阻塞的通讯方式，`cosocket` 分为两类，第一类是与下游通讯的 `cosocket`,第二类是在 `OpenResty` 内主动创建的 `cosocket`。第一类是基于原始请求的连接建立，由 `ngx.req.socket` 函数创建。对于第二类 `cosocket` 分为 TCP 与 UDP 两类，分别由 `ngx.socket.tcp`,`ngx.socket.stream` 和 `ngx.socket.udp` 创建。

## 二 downstream socket

### 1. 指令

```lua
syntax: tcpsock, err = ngx.req.socket()
syntax: tcpsock, err = ngx.req.socket(raw)
context: rewrite_by_lua*, access_by_lua*, content_by_lua*
```

返回只读的 `downstream` 方向连接的 `cosocket` 对象，只支持 `receive`, `receiveuntil` 方法。此方法通常用于以流式读取当前请求的包体，不应该与 `lua_need_request_body` 指令或 `ngx.req.read_body`,`ngx.req.discard_body` 混用。
当使用 `true` 参数调用函数时返回的 `tcpsock` 是全双工的 `cosocket`，除了 `receive`, `receiveuntil` 方法外还可以调用 `send` 方法向下游发送应答。注意，当调用 `ngx.req.socket(true)` 时缓冲区中不能存在之前由于调用 `ngx.say`, `ngx.print`, `ngx.send_headers` 发送的数据。正确的方式是，先调用 `ngx.flush(ture)` 将缓冲区中数据写出然后再调用 `ngx.req.socket(true)`，以保证缓冲区中无待发送数据。

### 2. 实现

```c
// 注册 ngx.req.socket
void
ngx_http_lua_inject_req_socket_api(lua_State *L)
{
    lua_pushcfunction(L, ngx_http_lua_req_socket);
    lua_setfield(L, -2, "socket");
}

// 创建一个 downstream socket
static int
ngx_http_lua_req_socket(lua_State *L)
{
    // 省略变量定义，参数判断

    r = ngx_http_lua_get_req(L);

    if (r != r->main) {
        return luaL_error(L, "attempt to read the request body in a "
                          "subrequest");
    }

    // 不支持 SPDY, HTTP2 协议
#if (NGX_HTTP_SPDY)
    if (r->spdy_stream) {
        return luaL_error(L, "spdy not supported yet");
    }
#endif

#if (NGX_HTTP_V2)
    if (r->stream) {
        return luaL_error(L, "http v2 not supported yet");
    }
#endif

    // 不支持 CHUNKED 传输编码
#if nginx_version >= 1003009
    if (!raw && r->headers_in.chunked) {
        lua_pushnil(L);
        lua_pushliteral(L, "chunked request bodies not supported yet");
        return 2;
    }
#endif

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no ctx found");
    }

    // 阶段判断
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT);

    // downstream socket 基于原始请求的 connection
    c = r->connection;

    if (raw) {
#if !defined(nginx_version) || nginx_version < 1003013
        lua_pushnil(L);
        lua_pushliteral(L, "nginx version too old");
        return 2;
#else
        // 判断是否已经读入请求包体
        if (r->request_body) {
            if (r->request_body->rest > 0) {
                lua_pushnil(L);
                lua_pushliteral(L, "pending request body reading in some "
                                "other thread");
                return 2;
            }
        } else {
            rb = ngx_pcalloc(r->pool, sizeof(ngx_http_request_body_t));
            if (rb == NULL) {
                return luaL_error(L, "no memory");
            }
            r->request_body = rb;
        }

        if (c->buffered & NGX_HTTP_LOWLEVEL_BUFFERED) {
            lua_pushnil(L);
            lua_pushliteral(L, "pending data to write");
            return 2;
        }

        if (ctx->buffering) {
            lua_pushnil(L);
            lua_pushliteral(L, "http 1.0 buffering");
            return 2;
        }

        if (!r->header_sent) {
            /* prevent other parts of nginx from sending out
             * the response header */
            r->header_sent = 1;
        }

        ctx->header_sent = 1;

        dd("ctx acquired raw req socket: %d", ctx->acquired_raw_req_socket);

        if (ctx->acquired_raw_req_socket) {
            lua_pushnil(L);
            lua_pushliteral(L, "duplicate call");
            return 2;
        }

        ctx->acquired_raw_req_socket = 1;
        r->keepalive = 0;
        r->lingering_close = 1;
#endif

    } else {
        /* request body reader */
        // 已经读入请求包体
        if (r->request_body) {
            lua_pushnil(L);
            lua_pushliteral(L, "request body already exists");
            return 2;
        }

        // 已经设置丢弃请求包体
        if (r->discard_body) {
            lua_pushnil(L);
            lua_pushliteral(L, "request body discarded");
            return 2;
        }

        dd("req content length: %d", (int) r->headers_in.content_length_n);

        // 无请求包体
        if (r->headers_in.content_length_n <= 0) {
            lua_pushnil(L);
            lua_pushliteral(L, "no body");
            return 2;
        }

        // 判断是否是有 expect 100 请求头，并进行应答，允许客户端继续发送包体
        // 与主逻辑无关
        if (ngx_http_lua_test_expect(r) != NGX_OK) {
            lua_pushnil(L);
            lua_pushliteral(L, "test expect failed");
            return 2;
        }

        /* prevent other request body reader from running */

        rb = ngx_pcalloc(r->pool, sizeof(ngx_http_request_body_t));
        if (rb == NULL) {
            return luaL_error(L, "no memory");
        }

        rb->rest = r->headers_in.content_length_n;

        r->request_body = rb;
    }

    // 创建 req_socket 表
    lua_createtable(L, 3 /* narr */, 1 /* nrec */); /* the object */

    // 根据是否为 raw 设置不同的 _index 元表
    if (raw) {
        lua_pushlightuserdata(L, &ngx_http_lua_raw_req_socket_metatable_key);
    } else {
        lua_pushlightuserdata(L, &ngx_http_lua_req_socket_metatable_key);
    }

    // 设置 table 元表为 ngx_http_lua_raw_req_socket_metatable_key|ngx_http_lua_req_socket_metatable_key 伪索引指定的表
    lua_rawget(L, LUA_REGISTRYINDEX);
    lua_setmetatable(L, -2);

    // 创建新的 socket 对象
    u = lua_newuserdata(L, sizeof(ngx_http_lua_socket_tcp_upstream_t));
    if (u == NULL) {
        return luaL_error(L, "no memory");
    }

#if 1
    // 设置 socket 的元表为伪索引中由 ngx_http_lua_downstream_udata_metatable_key 指定的表
    lua_pushlightuserdata(L, &ngx_http_lua_downstream_udata_metatable_key);
    lua_rawget(L, LUA_REGISTRYINDEX);
    lua_setmetatable(L, -2);
#endif

    // 将 socket 保存到 req_socket 表
    // req_socket[SOCKET_CTX_INDEX] = u
    lua_rawseti(L, 1, SOCKET_CTX_INDEX);

    ngx_memzero(u, sizeof(ngx_http_lua_socket_tcp_upstream_t));

    if (raw) {
        u->raw_downstream = 1;
    } else {
        u->body_downstream = 1;
    }

    coctx = ctx->cur_co_ctx;

    u->request = r;

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    u->conf = llcf;
    // 使用 location 默认的读、连接、写超时时间
    u->read_timeout = u->conf->read_timeout;
    u->connect_timeout = u->conf->connect_timeout;
    u->send_timeout = u->conf->send_timeout;

    cln = ngx_http_lua_cleanup_add(r, 0);
    if (cln == NULL) {
        u->ft_type |= NGX_HTTP_LUA_SOCKET_FT_ERROR;
        lua_pushnil(L);
        lua_pushliteral(L, "no memory");
        return 2;
    }

    // 清理函数
    // downstream 不会关闭连接；非 downstream 会将其进行关闭
    cln->handler = ngx_http_lua_socket_tcp_cleanup;
    cln->data = u;
    u->cleanup = &cln->handler;

    pc = &u->peer;

    pc->log = c->log;
    pc->log_error = NGX_ERROR_ERR;

    // c 是原始连接 r->connection;
    pc->connection = c;

    dd("setting data to %p", u);

    coctx->data = u;
    ctx->downstream = u;

    // 删除原先的读超时定时器
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

    if (raw) {
        // 删除原先的写超时定时器
        if (c->write->timer_set) {
            ngx_del_timer(c->write);
        }
    }

    // 返回 req_socket 表
    lua_settop(L, 1);
    return 1;
}
```

## 三 TCP

### 1. 指令

```lua
syntax: tcpsock = ngx.socket.tcp()
syntax: tcpsock = ngx.socket.stream()
context: rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*
```

创建并返回一个 `TCP` 或流式 `UNINX` 域 `socket` 对象，支持：`connect`, `sslhandshake`, `send`, `receive`, `close`, `settimeout`, `settimeouts`, `setoption`, `receiveany`, `receiveuntil`, `setkeepalive`, `getreusedtimes` 方法，这些方法都是非阻塞的。`tcpsock` 与创建它的 `lua-handler` 有相同生命周期，为避免 `panic` 不能将 `tcpsock` 传递给其他 `lua-handler` (包括 `ngx.timer`)，当然也不能共享给其他请求。
对于每个 `cosocket` 对象底层的连接，如果没有显示调用 `close` 进行关闭，或者使用 `setkeepalive` 放入连接池，那么当当前请求结束或者 `cosocket` 被 `gc` 时会关闭连接。当操作 `cosocket` 对象出现致命错误（读超时不被当做致命错误）时会关闭 `cosocket`，如果再次调用 `close` 进行关闭会返回 "closed" 错误信息。

### 2. 实现

向 `ngx` 注入函数：

```c
void
ngx_http_lua_inject_socket_tcp_api(ngx_log_t *log, lua_State *L)
{
    ngx_int_t         rc;

    // 创建一个 table 用来存储 ngx.socket 操作函数
    lua_createtable(L, 0, 4 /* nrec */);    /* ngx.socket */

    // ngx_http_lua_socket_tcp
    lua_pushcfunction(L, ngx_http_lua_socket_tcp);
    // 复制 ngx_http_lua_socket_tcp 并压入栈顶
    lua_pushvalue(L, -1);

    // ngx.socket['tcp'] = ngx_http_lua_socket_tcp
    // 向表添加 tcp 函数索引
    lua_setfield(L, -3, "tcp");
    // ngx.socket[stream] = ngx_http_lua_socket_tcp
    // 向表添加 stream 函数索引
    lua_setfield(L, -2, "stream");

    {
        const char  buf[] = "local sock = ngx.socket.tcp()"
                            " local ok, err = sock:connect(...)"
                            " if ok then return sock else return nil, err end";
        // 会将函数压入栈顶
        rc = luaL_loadbuffer(L, buf, sizeof(buf) - 1, "=ngx.socket.connect");
    }

    if (rc != NGX_OK) {
        ngx_log_error(NGX_LOG_CRIT, log, 0,
                      "failed to load Lua code for ngx.socket.connect(): %i",
                      rc);

    } else {
        // 向表添加 connect 函数索引
        // ngx.socket[connect] = func
        lua_setfield(L, -2, "connect");
    }

    // 向 ngx 表添加 socket 表索引
    // ngx[socket] = { tcp = func, stream = func, connect = func}
    lua_setfield(L, -2, "socket");


    // 以下为注册 req_socket 的元表

    /* {{{req socket object metatable */
    // 将 ngx_http_lua_req_socket_metatable_key 的地址作为 key 压入栈
    lua_pushlightuserdata(L, &ngx_http_lua_req_socket_metatable_key);
    // 创建 table_0，并压入栈顶
    lua_createtable(L, 0 /* narr */, 4 /* nrec */);

    // 在 table_0 添加函数索引：table_0['receive'] = ngx_http_lua_socket_tcp_receive
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_receive);
    lua_setfield(L, -2, "receive");

    // 在 table_0 添加函数索引：table_0['receiveuntil'] = ngx_http_lua_socket_tcp_receiveuntil
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_receiveuntil);
    lua_setfield(L, -2, "receiveuntil");

    // 在 table_0 添加函数索引：table_0['settimeout'] = ngx_http_lua_socket_tcp_settimeout
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_settimeout);
    lua_setfield(L, -2, "settimeout"); /* ngx socket mt */

    // 创建 table_0 拷贝，并压入栈顶，此时栈结构：
    // 索引 索引 值
    //   1  -4  ngx['socket'] = { tcp=func, stream=func, connect = func }
    //   2  -3  key (address of ngx_http_lua_req_socket_metatable_key)
    //   3  -2  table_0 = {receive=func, receiveuntil=func, settimeout=func}
    //   4  -1  table_0_copy
    lua_pushvalue(L, -1);
    // 给 table_0 添加 __index 元表
    lua_setfield(L, -2, "__index");

    // 向虚拟注册表注册值
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */

    /* {{{raw req socket object metatable */
    // 将 ngx_http_lua_raw_req_socket_metatable_key 的地址作为 key 压入栈
    lua_pushlightuserdata(L, &ngx_http_lua_raw_req_socket_metatable_key);
    // 创建新表 table_1，并压入栈顶
    lua_createtable(L, 0 /* narr */, 5 /* nrec */);

    // 在 table_1 中插入值，table_1['receive'] = ngx_http_lua_socket_tcp_receive
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_receive);
    lua_setfield(L, -2, "receive");

    // 在 table_1 中插入值，table_1['receiveuntil'] = ngx_http_lua_socket_tcp_receiveuntil
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_receiveuntil);
    lua_setfield(L, -2, "receiveuntil");

    // 在 table_1 中插入值，table_1['send'] = ngx_http_lua_socket_tcp_send
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_send);
    lua_setfield(L, -2, "send");

    // 在 table_1 中插入值，table_1['settimeout'] = ngx_http_lua_socket_tcp_settimeout
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_settimeout);
    lua_setfield(L, -2, "settimeout"); /* ngx socket mt */

    // 创建 table_1 拷贝，并压入栈顶
    lua_pushvalue(L, -1);
    // 给 table_0 添加 __index 元表
    lua_setfield(L, -2, "__index");

    // 向虚拟注册表注册值
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */

    /* {{{tcp object metatable */
    lua_pushlightuserdata(L, &ngx_http_lua_tcp_socket_metatable_key);
    lua_createtable(L, 0 /* narr */, 11 /* nrec */);

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_connect);
    lua_setfield(L, -2, "connect");

#if (NGX_HTTP_SSL)

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_sslhandshake);
    lua_setfield(L, -2, "sslhandshake");

#endif

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_receive);
    lua_setfield(L, -2, "receive");

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_receiveuntil);
    lua_setfield(L, -2, "receiveuntil");

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_send);
    lua_setfield(L, -2, "send");

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_close);
    lua_setfield(L, -2, "close");

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_setoption);
    lua_setfield(L, -2, "setoption");

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_settimeout);
    lua_setfield(L, -2, "settimeout"); /* ngx socket mt */

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_getreusedtimes);
    lua_setfield(L, -2, "getreusedtimes");

    lua_pushcfunction(L, ngx_http_lua_socket_tcp_setkeepalive);
    lua_setfield(L, -2, "setkeepalive");

    lua_pushvalue(L, -1);
    lua_setfield(L, -2, "__index");
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */

    /* {{{upstream userdata metatable */
    lua_pushlightuserdata(L, &ngx_http_lua_upstream_udata_metatable_key);
    lua_createtable(L, 0 /* narr */, 1 /* nrec */); /* metatable */
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_upstream_destroy);
    lua_setfield(L, -2, "__gc");
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */

    /* {{{downstream userdata metatable */
    lua_pushlightuserdata(L, &ngx_http_lua_downstream_udata_metatable_key);
    lua_createtable(L, 0 /* narr */, 1 /* nrec */); /* metatable */
    lua_pushcfunction(L, ngx_http_lua_socket_downstream_destroy);
    lua_setfield(L, -2, "__gc");
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */

    /* {{{socket pool userdata metatable */
    lua_pushlightuserdata(L, &ngx_http_lua_pool_udata_metatable_key);
    lua_createtable(L, 0, 1); /* metatable */
    lua_pushcfunction(L, ngx_http_lua_socket_shutdown_pool);
    lua_setfield(L, -2, "__gc");
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */

    /* {{{socket compiled pattern userdata metatable */
    lua_pushlightuserdata(L, &ngx_http_lua_pattern_udata_metatable_key);
    lua_createtable(L, 0 /* narr */, 1 /* nrec */); /* metatable */
    lua_pushcfunction(L, ngx_http_lua_socket_cleanup_compiled_pattern);
    lua_setfield(L, -2, "__gc");
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */

#if (NGX_HTTP_SSL)

    /* {{{ssl session userdata metatable */
    lua_pushlightuserdata(L, &ngx_http_lua_ssl_session_metatable_key);
    lua_createtable(L, 0 /* narr */, 1 /* nrec */); /* metatable */
    lua_pushcfunction(L, ngx_http_lua_ssl_free_session);
    lua_setfield(L, -2, "__gc");
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */

#endif
}
```


调用 `ngx.socket.tcp` 创建 `tcpsocket` 对象，在此过程中并未创建 socket 对象，仅创建了 cosocket table，在调用其 `connect` 方法时才会进行创建：

```c
// 创建 cosocket
static int
ngx_http_lua_socket_tcp(lua_State *L)
{
    ngx_http_request_t      *r;
    ngx_http_lua_ctx_t      *ctx;

    if (lua_gettop(L) != 0) {
        return luaL_error(L, "expecting zero arguments, but got %d",
                          lua_gettop(L));
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request found");
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no ctx found");
    }

    // 阶段检查
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER
                               | NGX_HTTP_LUA_CONTEXT_SSL_CERT
                               | NGX_HTTP_LUA_CONTEXT_SSL_SESS_FETCH);

    // 创建 tcp socket table 并设置元表
    lua_createtable(L, 3 /* narr */, 1 /* nrec */);
    lua_pushlightuserdata(L, &ngx_http_lua_tcp_socket_metatable_key);
    lua_rawget(L, LUA_REGISTRYINDEX);
    lua_setmetatable(L, -2);

    dd("top: %d", lua_gettop(L));

    return 1;
}
```

`tcpsock:connect(host, port, options_table?)` 实现：
```c
// 创建并建立连接
// connect 签名：tcpsock:connect(host, port, options_table?)
static int
ngx_http_lua_socket_tcp_connect(lua_State *L)
{
    ngx_http_request_t          *r;
    ngx_http_lua_ctx_t          *ctx;
    ngx_str_t                    host;
    int                          port;
    ngx_resolver_ctx_t          *rctx, temp;
    ngx_http_core_loc_conf_t    *clcf;
    int                          saved_top;
    int                          n;
    u_char                      *p;
    size_t                       len;
    ngx_url_t                    url;
    ngx_int_t                    rc;
    ngx_http_lua_loc_conf_t     *llcf;
    ngx_peer_connection_t       *pc;
    int                          timeout;
    unsigned                     custom_pool;
    int                          key_index;
    const char                  *msg;
    ngx_http_lua_co_ctx_t       *coctx;

    ngx_http_lua_socket_tcp_upstream_t      *u;

    n = lua_gettop(L);
    if (n != 2 && n != 3 && n != 4) {
        return luaL_error(L, "ngx.socket connect: expecting 2, 3, or 4 "
                          "arguments (including the object), but seen %d", n);
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request found");
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no ctx found");
    }

    // 介入阶段判断
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER
                               | NGX_HTTP_LUA_CONTEXT_SSL_CERT
                               | NGX_HTTP_LUA_CONTEXT_SSL_SESS_FETCH);
    // 检查栈底参数类型
    luaL_checktype(L, 1, LUA_TTABLE);

    p = (u_char *) luaL_checklstring(L, 2, &len);

    key_index = 2;
    custom_pool = 0;

    // 如果最后一个参数时 table 类型
    // 解析调用 connect 函数的 options_table 参数
    if (lua_type(L, n) == LUA_TTABLE) {

        /* found the last optional option table */

        lua_getfield(L, n, "pool");

        switch (lua_type(L, -1)) {
        case LUA_TNUMBER:
            lua_tostring(L, -1);

        case LUA_TSTRING:
            custom_pool = 1;

            lua_pushvalue(L, -1);
            lua_rawseti(L, 1, SOCKET_KEY_INDEX);

            key_index = n + 1;

            break;

        case LUA_TNIL:
            lua_pop(L, 2);
            break;

        default:
            msg = lua_pushfstring(L, "bad \"pool\" option type: %s",
                                  luaL_typename(L, -1));
            luaL_argerror(L, n, msg);
            break;
        }

        n--;
    }

    if (n == 3) {
        port = luaL_checkinteger(L, 3);

        if (port < 0 || port > 65536) {
            lua_pushnil(L);
            lua_pushfstring(L, "bad port number: %d", port);
            return 2;
        }

        if (!custom_pool) {
            lua_pushliteral(L, ":");
            lua_insert(L, 3);
            lua_concat(L, 3);
        }

        dd("socket key: %s", lua_tostring(L, -1));

    } else { /* n == 2 */
        port = 0;
    }

    if (!custom_pool) {
        /* the key's index is 2 */

        lua_pushvalue(L, 2);
        lua_rawseti(L, 1, SOCKET_KEY_INDEX);
    }

    lua_rawgeti(L, 1, SOCKET_CTX_INDEX);
    u = lua_touserdata(L, -1);
    lua_pop(L, 1);

    if (u) {
        if (u->request && u->request != r) {
            return luaL_error(L, "bad request");
        }

        ngx_http_lua_socket_check_busy_connecting(r, u, L);
        ngx_http_lua_socket_check_busy_reading(r, u, L);
        ngx_http_lua_socket_check_busy_writing(r, u, L);

        if (u->body_downstream || u->raw_downstream) {
            return luaL_error(L, "attempt to re-connect a request socket");
        }

        if (u->peer.connection) {
            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "lua tcp socket reconnect without shutting down");

            ngx_http_lua_socket_tcp_finalize(r, u);
        }

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua reuse socket upstream ctx");

    } else {
        // 调用 ngx.socket.tcp 时并没有创建 socket 对象，在第一次调用 connect 时创建
        u = lua_newuserdata(L, sizeof(ngx_http_lua_socket_tcp_upstream_t));
        if (u == NULL) {
            return luaL_error(L, "no memory");
        }

#if 1
        lua_pushlightuserdata(L, &ngx_http_lua_upstream_udata_metatable_key);
        lua_rawget(L, LUA_REGISTRYINDEX);
        lua_setmetatable(L, -2);
#endif

        lua_rawseti(L, 1, SOCKET_CTX_INDEX);
    }

    ngx_memzero(u, sizeof(ngx_http_lua_socket_tcp_upstream_t));

    coctx = ctx->cur_co_ctx;

    u->request = r; /* set the controlling request */

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    u->conf = llcf;

    pc = &u->peer;

    pc->log = r->connection->log;
    pc->log_error = NGX_ERROR_ERR;

    dd("lua peer connection log: %p", pc->log);

    lua_rawgeti(L, 1, SOCKET_TIMEOUT_INDEX);
    timeout = (ngx_int_t) lua_tointeger(L, -1);
    lua_pop(L, 1);

    if (timeout > 0) {
        u->send_timeout = (ngx_msec_t) timeout;
        u->read_timeout = (ngx_msec_t) timeout;
        u->connect_timeout = (ngx_msec_t) timeout;

    } else {
        u->read_timeout = u->conf->read_timeout;
        u->send_timeout = u->conf->send_timeout;
        u->connect_timeout = u->conf->connect_timeout;
    }

    // 尝试从连接池中查找
    rc = ngx_http_lua_get_keepalive_peer(r, L, key_index, u);

    if (rc == NGX_OK) {
        lua_pushinteger(L, 1);
        return 1;
    }

    if (rc == NGX_ERROR) {
        lua_pushnil(L);
        lua_pushliteral(L, "error in get keepalive peer");
        return 2;
    }

    /* rc == NGX_DECLINED */

    /* TODO: we should avoid this in-pool allocation */
    // connect 函数调用中 host 参数解析
    host.data = ngx_palloc(r->pool, len + 1);
    if (host.data == NULL) {
        return luaL_error(L, "no memory");
    }

    host.len = len;

    ngx_memcpy(host.data, p, len);
    host.data[len] = '\0';

    ngx_memzero(&url, sizeof(ngx_url_t));

    url.url.len = host.len;
    url.url.data = host.data;
    url.default_port = (in_port_t) port;
    url.no_resolve = 1;

    // 地址解析获得地址
    if (ngx_parse_url(r->pool, &url) != NGX_OK) {
        lua_pushnil(L);
        if (url.err) {
            lua_pushfstring(L, "failed to parse host name \"%s\": %s",
                            host.data, url.err);
        } else {
            lua_pushfstring(L, "failed to parse host name \"%s\"", host.data);
        }

        return 2;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua tcp socket connect timeout: %M", u->connect_timeout);

    u->resolved = ngx_pcalloc(r->pool, sizeof(ngx_http_upstream_resolved_t));
    if (u->resolved == NULL) {
        return luaL_error(L, "no memory");
    }

    if (url.addrs && url.addrs[0].sockaddr) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua tcp socket network address given directly");

        u->resolved->sockaddr = url.addrs[0].sockaddr;
        u->resolved->socklen = url.addrs[0].socklen;
        u->resolved->naddrs = 1;
        u->resolved->host = url.addrs[0].name;
    } else {
        u->resolved->host = host;
        u->resolved->port = (in_port_t) port;
    }

    // ngx_parse_url 成功，已经有地址
    if (u->resolved->sockaddr) {
        // 直接进行连接
        rc = ngx_http_lua_socket_resolve_retval_handler(r, u, L);
        if (rc == NGX_AGAIN) {
            return lua_yield(L, 0);
        }
        return rc;
    }

    // 需要进行 dns 解析
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    temp.name = host;
    rctx = ngx_resolve_start(clcf->resolver, &temp);
    if (rctx == NULL) {
        u->ft_type |= NGX_HTTP_LUA_SOCKET_FT_RESOLVER;
        lua_pushnil(L);
        lua_pushliteral(L, "failed to start the resolver");
        return 2;
    }

    if (rctx == NGX_NO_RESOLVER) {
        u->ft_type |= NGX_HTTP_LUA_SOCKET_FT_RESOLVER;
        lua_pushnil(L);
        lua_pushfstring(L, "no resolver defined to resolve \"%s\"", host.data);
        return 2;
    }

    rctx->name = host;
#if !defined(nginx_version) || nginx_version < 1005008
    rctx->type = NGX_RESOLVE_A;
#endif
    rctx->handler = ngx_http_lua_socket_resolve_handler;
    rctx->data = u;
    rctx->timeout = clcf->resolver_timeout;

    u->resolved->ctx = rctx;
    u->write_co_ctx = ctx->cur_co_ctx;

    ngx_http_lua_cleanup_pending_operation(coctx);
    coctx->cleanup = ngx_http_lua_tcp_resolve_cleanup;
    coctx->data = u;

    saved_top = lua_gettop(L);

    if (ngx_resolve_name(rctx) != NGX_OK) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua tcp socket fail to run resolver immediately");

        u->ft_type |= NGX_HTTP_LUA_SOCKET_FT_RESOLVER;

        u->resolved->ctx = NULL;
        lua_pushnil(L);
        lua_pushfstring(L, "%s could not be resolved", host.data);

        return 2;
    }

    if (u->conn_waiting) {
        dd("resolved and already connecting");
        return lua_yield(L, 0);
    }

    n = lua_gettop(L) - saved_top;
    if (n) {
        dd("errors occurred during resolving or connecting"
           "or already connected");
        return n;
    }

    /* still resolving */

    u->conn_waiting = 1;
    u->write_prepare_retvals = ngx_http_lua_socket_resolve_retval_handler;

    dd("setting data to %p", u);

    if (ctx->entered_content_phase) {
        r->write_event_handler = ngx_http_lua_content_wev_handler;

    } else {
        r->write_event_handler = ngx_http_core_run_phases;
    }

    return lua_yield(L, 0);
}
```

## 四 UDP

### 1. 指令
```lua
syntax: udpsock = ngx.socket.udp()
context: rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*
```

创建并返回一个 `UDP` 或数据包式 `UNINX` 域 `socket` 对象（aka cosocket），支持：`setpeername`, `send`, `receive`, `close`, `settimeout` 方法，这些方法都是非阻塞方法。


