---
layout: post
title: OpenResty Api - balancer
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description:
---

## 一 概述

如何使用 OpenResty 实现负载均衡模块？使用 `balancer_by_lua_file|block` 指令能够使用 Lua 实现负载均衡模块，在 Lua 代码中必须使用 `ngx.balancer.set_current_peer` 函数设置当前使用 `server` 地址端口。

OpenResty 使用 `balancer_by_lua_file|block` 指令修改 `ngx_http_upstream_module` 模块的 `ngx_http_upstream_srv_conf_t::peer.init_upstream` 回调函数指针，在 upstream 初始化时会再次将当前 upstream 配置的 `ngx_http_upstream_srv_conf_t::peer.init` 回调函数指针修改，实现 `server` 获取释放（get/free）注入功能。

## 二 指令

### 1. balancer_by_lua_block

```NGINX
syntax: balancer_by_lua_block { lua-script }
context: upstream
phase: content
```

注入 Lua 代码块，实现负载均衡功能。
**如果 `lua-script` 处理失败会回退到 `round robin` 模块进行负载均衡处理**。


### 2. balancer_by_lua_file

```NGINX
syntax: balancer_by_lua_file <path-to-lua-file>
context: upstream
phase: content
```

注入 Lua 代码文件，实现负载均衡功能。

## 三 函数

### 1. set_current_peer

```lua
syntax: ok, err = balancer.set_current_peer(host, port)
context: balancer_by_lua*
```

设置当前使用的 `server` 地址和端口。
**支持 `unix` 域地址，IPv4 和 IPv6，但是不支持域名**。

### 2. set_more_tries

```lua
syntax: ok, err = balancer.set_more_tries(count)
context: balancer_by_lua*
```

设置当前请求重试次数（不包含当前尝试）。`count` 不能大于 `proxy_next_upstream_tries` 设置值。

### 3. get_last_failure

```lua
syntax: state_name, status_code = balancer.get_last_failure()
context: balancer_by_lua*
```

在触发重试机制时，查询先前失败的详细信息。如果有失败，则返回状态名称已经状态码。如果当前尝试是首次尝试，返回值为 `nil`。

### 4. set_timeouts

```lua
syntax: ok, err = balancer.set_timeouts(connect_timeout, send_timeout, read_timeout)
context: balancer_by_lua*
```

设置当前和后续 upstream 连接的超时时间（连接超时，发送超时，读超时），以秒为单位精确到毫秒。
**超时时间必须大于零，不能等于零或小于零**。

## 四 实现

### 1. balancer_by_lua_block|file

模块指令
```c
static ngx_command_t ngx_http_lua_cmds[] = {
    // ...
    { ngx_string("balancer_by_lua_block"),
      NGX_HTTP_UPS_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_http_lua_balancer_by_lua_block,
      NGX_HTTP_SRV_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_balancer_handler_inline },

    { ngx_string("balancer_by_lua_file"),
      NGX_HTTP_UPS_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_balancer_by_lua,
      NGX_HTTP_SRV_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_balancer_handler_file },
      // ...
};


// balancer_by_lua 配置解析函数
char *
ngx_http_lua_balancer_by_lua(ngx_conf_t *cf, ngx_command_t *cmd,
    void *conf)
{
    u_char                      *p;
    u_char                      *name;
    ngx_str_t                   *value;
    ngx_http_lua_srv_conf_t     *lscf = conf;

    ngx_http_upstream_srv_conf_t      *uscf;

    dd("enter");

    /*  must specify a content handler */
    if (cmd->post == NULL) {
        return NGX_CONF_ERROR;
    }

    if (lscf->balancer.handler) {
        return "is duplicate";
    }

    value = cf->args->elts;
    // 设置处理函数指针
    // cmd->post 为 ngx_http_lua_balancer_handler_inline
    lscf->balancer.handler = (ngx_http_lua_srv_conf_handler_pt) cmd->post;

    if (cmd->post == ngx_http_lua_balancer_handler_file) {
        /* Lua code in an external file */

        name = ngx_http_lua_rebase_path(cf->pool, value[1].data,
                                        value[1].len);
        if (name == NULL) {
            return NGX_CONF_ERROR;
        }

        lscf->balancer.src.data = name;
        lscf->balancer.src.len = ngx_strlen(name);

        p = ngx_palloc(cf->pool, NGX_HTTP_LUA_FILE_KEY_LEN + 1);
        if (p == NULL) {
            return NGX_CONF_ERROR;
        }

        lscf->balancer.src_key = p;

        p = ngx_copy(p, NGX_HTTP_LUA_FILE_TAG, NGX_HTTP_LUA_FILE_TAG_LEN);
        p = ngx_http_lua_digest_hex(p, value[1].data, value[1].len);
        *p = '\0';

    } else {
        /* inlined Lua code */

        lscf->balancer.src = value[1];

        p = ngx_palloc(cf->pool, NGX_HTTP_LUA_INLINE_KEY_LEN + 1);
        if (p == NULL) {
            return NGX_CONF_ERROR;
        }

        lscf->balancer.src_key = p;

        p = ngx_copy(p, NGX_HTTP_LUA_INLINE_TAG, NGX_HTTP_LUA_INLINE_TAG_LEN);
        p = ngx_http_lua_digest_hex(p, value[1].data, value[1].len);
        *p = '\0';
    }
    // 修改 upstream 模块的 peer.init_upstream 指针，lua_module 介入处理
    // 跟踪代码可以发现，lua-nginx-module 只在请求的 peer.get/peer.free 介入处理
    uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);

    if (uscf->peer.init_upstream) {
        ngx_conf_log_error(NGX_LOG_WARN, cf, 0,
                           "load balancing method redefined");
    }

    uscf->peer.init_upstream = ngx_http_lua_balancer_init;

    uscf->flags = NGX_HTTP_UPSTREAM_CREATE
                  |NGX_HTTP_UPSTREAM_WEIGHT
                  |NGX_HTTP_UPSTREAM_MAX_FAILS
                  |NGX_HTTP_UPSTREAM_FAIL_TIMEOUT
                  |NGX_HTTP_UPSTREAM_DOWN;

    return NGX_CONF_OK;
}


static ngx_int_t
ngx_http_lua_balancer_init(ngx_conf_t *cf,
    ngx_http_upstream_srv_conf_t *us)
{
    if (ngx_http_upstream_init_round_robin(cf, us) != NGX_OK) {
        return NGX_ERROR;
    }

    /* this callback is called upon individual requests */
    us->peer.init = ngx_http_lua_balancer_init_peer;

    return NGX_OK;
}


static ngx_int_t
ngx_http_lua_balancer_init_peer(ngx_http_request_t *r,
    ngx_http_upstream_srv_conf_t *us)
{
    ngx_http_lua_srv_conf_t            *bcf;
    ngx_http_lua_balancer_peer_data_t  *bp;

    bp = ngx_pcalloc(r->pool, sizeof(ngx_http_lua_balancer_peer_data_t));
    if (bp == NULL) {
        return NGX_ERROR;
    }

    r->upstream->peer.data = &bp->rrp;

    if (ngx_http_upstream_init_round_robin_peer(r, us) != NGX_OK) {
        return NGX_ERROR;
    }

    r->upstream->peer.get = ngx_http_lua_balancer_get_peer;
    r->upstream->peer.free = ngx_http_lua_balancer_free_peer;

#if (NGX_HTTP_SSL)
    r->upstream->peer.set_session = ngx_http_lua_balancer_set_session;
    r->upstream->peer.save_session = ngx_http_lua_balancer_save_session;
#endif

    bcf = ngx_http_conf_upstream_srv_conf(us, ngx_http_lua_module);

    bp->conf = bcf;
    bp->request = r;

    return NGX_OK;
}
```

### 2. ngx.balancer.*

`ngx.balancer.set_current_peer`,`ngx.balancer.set_more_tries`,`ngx.balancer.get_last_failure`,`ngx.balancer.set_timeouts` 是通过使用 FFI 技术设置 bp->sockaddr,bp->sockaddrlen,bp->host 等变量值（`bp` 是请求相关对象），在 lvm 执行完 lua 代码后，lua-nginx-module 会从 bp 中取出其值，作为 server 地址配置。

```c
int
ngx_http_lua_ffi_balancer_set_current_peer(ngx_http_request_t *r,
    const u_char *addr, size_t addr_len, int port, char **err)
{
    ngx_url_t              url;
    ngx_http_lua_ctx_t    *ctx;
    ngx_http_upstream_t   *u;

    ngx_http_lua_main_conf_t           *lmcf;
    ngx_http_lua_balancer_peer_data_t  *bp;

    if (r == NULL) {
        *err = "no request found";
        return NGX_ERROR;
    }

    u = r->upstream;

    if (u == NULL) {
        *err = "no upstream found";
        return NGX_ERROR;
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        *err = "no ctx found";
        return NGX_ERROR;
    }

    if ((ctx->context & NGX_HTTP_LUA_CONTEXT_BALANCER) == 0) {
        *err = "API disabled in the current context";
        return NGX_ERROR;
    }

    lmcf = ngx_http_get_module_main_conf(r, ngx_http_lua_module);

    /* we cannot read r->upstream->peer.data here directly because
     * it could be overridden by other modules like
     * ngx_http_upstream_keepalive_module.
     */
    bp = lmcf->balancer_peer_data;
    if (bp == NULL) {
        *err = "no upstream peer data found";
        return NGX_ERROR;
    }

    ngx_memzero(&url, sizeof(ngx_url_t));

    url.url.data = ngx_palloc(r->pool, addr_len);
    if (url.url.data == NULL) {
        *err = "no memory";
        return NGX_ERROR;
    }

    ngx_memcpy(url.url.data, addr, addr_len);

    url.url.len = addr_len;
    url.default_port = (in_port_t) port;
    url.uri_part = 0;
    url.no_resolve = 1;

    if (ngx_parse_url(r->pool, &url) != NGX_OK) {
        if (url.err) {
            *err = url.err;
        }

        return NGX_ERROR;
    }

    if (url.addrs && url.addrs[0].sockaddr) {
        bp->sockaddr = url.addrs[0].sockaddr;
        bp->socklen = url.addrs[0].socklen;
        bp->host = &url.addrs[0].name;

    } else {
        *err = "no host allowed";
        return NGX_ERROR;
    }

    return NGX_OK;
}
```
