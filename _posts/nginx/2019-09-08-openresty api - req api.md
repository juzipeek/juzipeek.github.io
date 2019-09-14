---
layout: post
title: OpenResty Api - req api
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description: 
---

## 一 概述

`ngx.req` 主要围绕对请求头、请求 `uri`、请求参数、请求包体、请求连接 `socket`、请求方法等进行操作。

## 二 请求头相关

包含 `ngx.req.http_version`、 `ngx.req.raw_header`、 `ngx.req.clear_header`、 `ngx.req.set_header`、 `ngx.req.get_headers` 操作函数。

### 1. `ngx.req.http_version`

功能：获得请求使用的 `HTTP` 协议版本号，返回数值类型（`0.9`、`1.0`、`1.1`、`2.0`），客户端发起请求时会将协议版本号放在请求行中。出错会返回 `nil`。

### 2. `ngx.req.raw_header`

```lua
str = ngx.req.raw_header(no_request_line?)
```

功能：用来获取 `NGINX` 收到的原始请求头，返回字符串。可选布尔类型参数 `no_request_line` 用来控制是否返回请求行。

介入阶段: set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua

示例：

```lua
ngx.print(ngx.req.raw_header())
```

输出如下：

```
GET /t HTTP/1.1
Host: localhost
Connection: close
Foo: bar
```

如果 `no_request_line` 为 `true` 时输出如下：

```
Host: localhost
Connection: close
Foo: bar
```

### 3. `ngx.req.clear_header`

```lua
ngx.req.clear_header(header_name)
```

清除请求头中名为 `header_name` 的头。`clear_header` 的实现与 `set_header` 方法实现相同，只不过 `clear_header` 设置的值为 `nil`。`clear_header` 实现：

```c
static int
ngx_http_lua_ngx_req_header_clear(lua_State *L)
{
  	// 参数检查，只有一个参数（header_name）
    if (lua_gettop(L) != 1) {
        return luaL_error(L, "expecting one arguments, but seen %d",
                          lua_gettop(L));
    }

  	// 向栈中压入 nil
    lua_pushnil(L);

    return ngx_http_lua_ngx_req_header_set_helper(L);
}

// 函数主要功能是将 header 的 key、value 获取出来，再调用最终保存函数
static int
ngx_http_lua_ngx_req_header_set_helper(lua_State *L)
{
	  // ... 参数检查
  	// http 0.9 无 header
    if (r->http_version < NGX_HTTP_VERSION_10) {
        return 0;
    }

    // 获得 header 名
    p = (u_char *) luaL_checklstring(L, 1, &len);

    key.data = ngx_palloc(r->pool, len + 1);
    if (key.data == NULL) {
        return luaL_error(L, "no memory");
    }

    ngx_memcpy(key.data, p, len);

    key.data[len] = '\0';

    key.len = len;

    // 获得 header 值
    if (lua_type(L, 2) == LUA_TNIL) {
        ngx_str_null(&value);
    } else if (lua_type(L, 2) == LUA_TTABLE) {
        n = luaL_getn(L, 2);
        if (n == 0) {
            ngx_str_null(&value);
        } else {
            // 使用数组设置 header
            for (i = 1; i <= n; i++) {
                dd("header value table index %d, top: %d", (int) i,
                   lua_gettop(L));

                // 取出数组中的元素放在栈顶
                lua_rawgeti(L, 2, i);
                // 从栈顶取 header 值
                p = (u_char *) luaL_checklstring(L, -1, &len);

                /*
                 * we also copy the trailling '\0' char here because nginx
                 * header values must be null-terminated
                 * */

                value.data = ngx_palloc(r->pool, len + 1);
                if (value.data == NULL) {
                    return luaL_error(L, "no memory");
                }

                ngx_memcpy(value.data, p, len + 1);
                value.len = len;

                rc = ngx_http_lua_set_input_header(r, key, value,
                                                   i == 1 /* override */);

                if (rc == NGX_ERROR) {
                    return luaL_error(L,
                                      "failed to set header %s (error: %d)",
                                      key.data, (int) rc);
                }
            }

            return 0;
        }

    } else {

        /*
         * we also copy the trailling '\0' char here because nginx
         * header values must be null-terminated
         * */

        p = (u_char *) luaL_checklstring(L, 2, &len);
        value.data = ngx_palloc(r->pool, len + 1);
        if (value.data == NULL) {
            return luaL_error(L, "no memory");
        }

        ngx_memcpy(value.data, p, len + 1);
        value.len = len;
    }

    dd("key: %.*s, value: %.*s",
       (int) key.len, key.data, (int) value.len, value.data);

    // 保存 header 到请求中
    rc = ngx_http_lua_set_input_header(r, key, value, 1 /* override */);

    if (rc == NGX_ERROR) {
        return luaL_error(L, "failed to set header %s (error: %d)",
                          key.data, (int) rc);
    }

    return 0;
}
```

上面两个函数做了 `header` 保存的准备工作，将 `header` 的 `key`、`value` 提取出来，调用 `ngx_http_lua_set_input_header` 对 `header` 进行操作。`lua-nginx` 模块对常见或特殊 `header` 有特殊操作（因为这些 `header` 保存位置或者同时需要更新其他信息），将其保存在一张操作表中（`ngx_http_lua_set_handlers`），表中最后一个元素是通用的 `header` 处理。实现：

```c
static ngx_http_lua_set_header_t  ngx_http_lua_set_handlers[] = {
    { ngx_string("Host"),
                 offsetof(ngx_http_headers_in_t, host),
                 ngx_http_set_host_header },
    // ... 省略其他常见或特殊 header
    { ngx_null_string, 0, ngx_http_set_header }
};

ngx_int_t
ngx_http_lua_set_input_header(ngx_http_request_t *r, ngx_str_t key,
    ngx_str_t value, unsigned override)
{
    ngx_http_lua_header_val_t         hv;
    ngx_http_lua_set_header_t        *handlers = ngx_http_lua_set_handlers;

    ngx_uint_t                        i;

    dd("set header value: %.*s", (int) value.len, value.data);

    hv.hash = ngx_hash_key_lc(key.data, key.len);
    hv.key = key;

    hv.offset = 0;
    hv.no_override = !override;
    hv.handler = NULL;

    // 遍历 ngx_http_lua_set_handlers 数组，查找是否是设置预定义 header
    for (i = 0; handlers[i].name.len; i++) {
        if (hv.key.len != handlers[i].name.len
            || ngx_strncasecmp(hv.key.data, handlers[i].name.data,
                               handlers[i].name.len) != 0)
        {
            dd("hv key comparison: %s <> %s", handlers[i].name.data,
               hv.key.data);

            continue;
        }

        dd("Matched handler: %s %s", handlers[i].name.data, hv.key.data);

        hv.offset = handlers[i].offset;
        hv.handler = handlers[i].handler;

        break;
    }

    // 未找到，使用 ngx_http_lua_set_handlers 数组末尾元素进行存储调用
    if (handlers[i].name.len == 0 && handlers[i].handler) {
        hv.offset = handlers[i].offset;   // 0
        hv.handler = handlers[i].handler; // ngx_http_set_header
    }

#if 1
    if (hv.handler == NULL) {
        return NGX_ERROR;
    }
#endif

    return hv.handler(r, &hv, &value);
}
```

### 4. `ngx.req.set_headr`

```lua
ngx.req.set_header(header_name, header_value)
```

将当前请求中名为 `header_name` 的头设置为 `header_value`，如果 `header_name` 已经存在则覆盖原有值。如果 `header_value` 为 `nil` 功能与 `clear_header` 相同。`header_value` 可以为 `array table`。

`set_header` 的实现与 `clear_header` 相似，只是其检查函数不同。

### 5. `ngx.req.get_headers`

```lua
headers, err = ngx.req.get_headers(max_headers?, raw?)
```

返回一个包含所有 `header` 的 `table`，可选数值参数 `max_headers` 用来控制获取的 `header` 数量（其实是 `header_value` 的数量，因为在请求中每个 `header_value` 是一 `header`），默认 100。可选布尔类型参数 `raw` 用来控制返回 `table` 是否有元方法，默认 `false`。

实现函数：

```c
static int
ngx_http_lua_ngx_req_get_headers(lua_State *L)
{
    // ... 忽略参数定义
    // 获得调用参数梳理
    n = lua_gettop(L);

    // 取调用参数 max、raw
    if (n >= 1) {
        if (lua_isnil(L, 1)) {
            max = NGX_HTTP_LUA_MAX_HEADERS;

        } else {
            max = luaL_checkinteger(L, 1);
        }

        if (n >= 2) {
            raw = lua_toboolean(L, 2);
        }
    } else {
        max = NGX_HTTP_LUA_MAX_HEADERS;
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ngx_http_lua_check_fake_request(L, r);

    // 计算 header 总数
    part = &r->headers_in.headers.part;
    count = part->nelts;
    while (part->next) {
        part = part->next;
        count += part->nelts;
    }

    if (max > 0 && count > max) {
        count = max;
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua exceeding request header limit %d", max);
    }

    // 创建返回 table，根据 raw 参数设置 table 的元表（有 index 元方法）
    lua_createtable(L, 0, count);
    if (!raw) {
        lua_pushlightuserdata(L, &ngx_http_lua_headers_metatable_key);
        lua_rawget(L, LUA_REGISTRYINDEX);
        lua_setmetatable(L, -2);
    }

    // 遍历 header，将其保存到新创建的 table 中
    part = &r->headers_in.headers.part;
    header = part->elts;
    for (i = 0; /* void */; i++) {

        dd("stack top: %d", lua_gettop(L));

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }

            part = part->next;
            header = part->elts;
            i = 0;
        }

        if (raw) {
            lua_pushlstring(L, (char *) header[i].key.data, header[i].key.len);
        } else {
            lua_pushlstring(L, (char *) header[i].lowcase_key,
                            header[i].key.len);
        }

        /* stack: table key */
        lua_pushlstring(L, (char *) header[i].value.data,
                        header[i].value.len); /* stack: table key value */

        ngx_http_lua_set_multi_value_table(L, -3);

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua request header: \"%V: %V\"",
                       &header[i].key, &header[i].value);

        if (--count == 0) {
            return 1;
        }
    }

    return 1;
}
```

## 三 请求 `uri`

### 1. `ngx.req.set_uri`

```lua
ngx.req.set_uri(uri, jump?)
```

使用字符串参数 `uri` 重写请求的 `uri`，`uri` 如果非字符串或者字符串长度为零会触发错误。可选布尔参数 `jump` 控制是否进行内部跳转（重新在当前 `server{}` 内进行 `location` 查找）。**`set_uri` 无法修改 `uri` 参数，需要使用 `ngx.req.set_uri_args` 修改参数**。

函数 `ngx_http_lua_ngx_req_set_uri` 是 `set_uri` 的实现函数，在其中修改了请求的 `uri` 以及 `uri_changed` 状态变量，实际的跳转功能是在协程处理循环中实现。

```c
static int
ngx_http_lua_ngx_req_set_uri(lua_State *L)
{
    // ... 省略无关代码
    // 参数数量获取
    n = lua_gettop(L);
    if (n != 1 && n != 2) {
        return luaL_error(L, "expecting 1 or 2 arguments but seen %d", n);
    }

    // 获取 uri
    p = (u_char *) luaL_checklstring(L, 1, &len);

    if (len == 0) {
        return luaL_error(L, "attempt to use zero-length uri");
    }

    // 是否跳转标识
    if (n == 2) {
        luaL_checktype(L, 2, LUA_TBOOLEAN);
        jump = lua_toboolean(L, 2);

        if (jump) {
            ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
            if (ctx == NULL) {
                return luaL_error(L, "no ctx found");
            }
						// 阶段检查
            ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE);
            ngx_http_lua_check_if_abortable(L, ctx);
        }
    }
    // uri 更新
    r->uri.data = ngx_palloc(r->pool, len);
    if (r->uri.data == NULL) {
        return luaL_error(L, "no memory");
    }

    ngx_memcpy(r->uri.data, p, len);

    r->uri.len = len;

    r->internal = 1;
    r->valid_unparsed_uri = 0;

    ngx_http_set_exten(r);
    // 跳转标识，在协程处理循环中会使用
    if (jump) {
        r->uri_changed = 1;
        return lua_yield(L, 0);
    }

    r->valid_location = 0;
    r->uri_changed = 0;

    return 0;
}
```

## 四 请求参数

### 1. `ngx.req.set_uri_args`

```lua
ngx.req.set_uri_args(args)
```

使用 `args` 参数重写当前请求的 `uri` 查询参数，`args` 可以是字符串也可以是 `table`。如果使用 `table` 会将参数进行 `uri_encode`（其实是调用 `ngx.encode_args` 将 `args` 编码为字符串）。`set_uri_args` 实现函数：

```c
static int
ngx_http_lua_ngx_req_set_uri_args(lua_State *L)
{
    // ... 省略其他代码
    // 参数检查
    if (lua_gettop(L) != 1) {
        return luaL_error(L, "expecting 1 argument but seen %d", lua_gettop(L));
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ngx_http_lua_check_fake_request(L, r);

    // 获取 args
    switch (lua_type(L, 1)) {
    case LUA_TNUMBER:
    case LUA_TSTRING:
        p = (u_char *) lua_tolstring(L, 1, &len);

        args.data = ngx_palloc(r->pool, len);
        if (args.data == NULL) {
            return luaL_error(L, "no memory");
        }

        ngx_memcpy(args.data, p, len);

        args.len = len;
        break;

    case LUA_TTABLE:
        ngx_http_lua_process_args_option(r, L, 1, &args);

        dd("args: %.*s", (int) args.len, args.data);

        break;

    default:
        msg = lua_pushfstring(L, "string, number, or table expected, "
                              "but got %s", luaL_typename(L, 2));
        return luaL_argerror(L, 1, msg);
    }

    dd("args: %.*s", (int) args.len, args.data);

    // 更新 args
    r->args.data = args.data;
    r->args.len = args.len;

    r->valid_unparsed_uri = 0;

    return 0;
}
```

### 2. `ngx.req.get_uri_args`

```lua
args, err = ngx.req.get_uri_args(max_args?)
```

返回一个包含当前请求 `URL` 参数的 `table`。可选数值参数 `max_args` 用来控制返回参数个数，默认为 100，修改为 0 则无限制。

当请求行为 `GET /test?foo=bar&bar=baz&bar=blah` 时，获得的 `table` 为：

```lua
{foo="bar", bar={"baz", "blah"}}
```

当请求行为 `GET /test?foo&bar` 时，获得的 `table` 为：

```lua
{foo=true, bar=true}
```

当请求行为 `GET /test?foo=&bar=` 时，获得的 `table` 为：

```lua
{foo="", bar=""}
```

实现函数：

```c
static int
ngx_http_lua_ngx_req_get_uri_args(lua_State *L)
{
    // ... 省略无关代码
    // 获得调用参数数量
    n = lua_gettop(L);
    if (n != 0 && n != 1) {
        return luaL_error(L, "expecting 0 or 1 arguments but seen %d", n);
    }

    if (n == 1) {
        max = luaL_checkinteger(L, 1);
        lua_pop(L, 1);
    } else {
        max = NGX_HTTP_LUA_MAX_ARGS;
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ngx_http_lua_check_fake_request(L, r);

    if (r->args.len == 0) {
        lua_createtable(L, 0, 0);
        return 1;
    }

    /* we copy r->args over to buf to simplify
     * unescaping query arg keys and values */
    // 获得请求参数
    buf = ngx_palloc(r->pool, r->args.len);
    if (buf == NULL) {
        return luaL_error(L, "no memory");
    }
		// 保存 args 的 table
    lua_createtable(L, 0, 4);
    ngx_memcpy(buf, r->args.data, r->args.len);
    last = buf + r->args.len;

    // 将请求参数进行解析并保存到 table 中
    // 会对参数名和参数值进行 url decode
    retval = ngx_http_lua_parse_args(L, buf, last, max);

    ngx_pfree(r->pool, buf);

    return retval;
}
```

### 3. `ngx.req.get_post_args`

```lua
args, err = ngx.req.get_post_args(max_args?)
```

返回包含当前请求 POST 查询参数的 `table`，调用前需要调用 `ngx.req.read_body` 函数或者通过 `lua_need_request_body` 读取请求包体。可选数值参数 `max_args` 用来控制返回参数个数，默认为 100，修改为 0 则无限制。实现：

```c
static int
ngx_http_lua_ngx_req_get_post_args(lua_State *L)
{
		// ... 忽略无关代码
    // 获得调用参数个数
    n = lua_gettop(L);
    if (n != 0 && n != 1) {
        return luaL_error(L, "expecting 0 or 1 arguments but seen %d", n);
    }
		// 参数检查
    if (n == 1) {
        max = luaL_checkinteger(L, 1);
        lua_pop(L, 1);
    } else {
        max = NGX_HTTP_LUA_MAX_ARGS;
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }
    ngx_http_lua_check_fake_request(L, r);

    // 已经丢弃请求包体
    if (r->discard_body) {
        lua_createtable(L, 0, 0);
        return 1;
    }

    if (r->request_body == NULL) {
        return luaL_error(L, "no request body found; "
                          "maybe you should turn on lua_need_request_body?");
    }

    // 请求包体存储在临时文件中，不能使用 get_post_args
    if (r->request_body->temp_file) {
        lua_pushnil(L);
        lua_pushliteral(L, "requesty body in temp file not supported");
        return 2;
    }

    if (r->request_body->bufs == NULL) {
        lua_createtable(L, 0, 0);
        return 1;
    }

    /* we copy r->request_body->bufs over to buf to simplify
     * unescaping query arg keys and values */

    // 读取请求包体
    len = 0;
    for (cl = r->request_body->bufs; cl; cl = cl->next) {
        len += cl->buf->last - cl->buf->pos;
    }

    if (len == 0) {
        lua_createtable(L, 0, 0);
        return 1;
    }

    buf = ngx_palloc(r->pool, len);
    if (buf == NULL) {
        return luaL_error(L, "no memory");
    }
    lua_createtable(L, 0, 4);

    p = buf;
    for (cl = r->request_body->bufs; cl; cl = cl->next) {
        p = ngx_copy(p, cl->buf->pos, cl->buf->last - cl->buf->pos);
    }

    dd("post body: %.*s", (int) len, buf);

    last = buf + len;
    // 解析请求包体
    retval = ngx_http_lua_parse_args(L, buf, last, max);
    ngx_pfree(r->pool, buf);

    return retval;
}
```

如果在 `NGINX` 配置中将请求包体保存在临时文件中，不能使用 `get_post_args` 函数。同时，`get_post_args` 会对参数名、参数值进行 `url` 解码处理。

## 五 请求体

### 1. `ngx.req.read_body`

```lua
ngx.req.read_body()
```

同步非阻塞的读取请求体，同步的意思是请求头读取完毕后返回，非阻塞的意思是不会将整个处理进程 `hang` 住，当无可读数据时会处理其他协程。实现：

```c
static int
ngx_http_lua_ngx_req_read_body(lua_State *L)
{
    // ... 忽略无关代码
    // 参数检查
    n = lua_gettop(L);

    if (n != 0) {
        return luaL_error(L, "expecting 0 arguments but seen %d", n);
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "request object not found");
    }

    r->request_body_in_single_buf = 1;
    r->request_body_in_persistent_file = 1;
    r->request_body_in_clean_file = 1;

#if 1
    if (r->request_body_in_file_only) {
        r->request_body_file_log_level = 0;
    }
#endif

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no ctx found");
    }

    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT);

    coctx = ctx->cur_co_ctx;
    if (coctx == NULL) {
        return luaL_error(L, "no co ctx found");
    }

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua start to read buffered request body");

    // 读请求包体
    rc = ngx_http_read_client_request_body(r, ngx_http_lua_req_body_post_read);

#if (nginx_version < 1002006) ||                                             \
        (nginx_version >= 1003000 && nginx_version < 1003009)
    r->main->count--;
#endif

    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        ctx->exit_code = rc;
        ctx->exited = 1;

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http read client request body returned error code %i, "
                       "exitting now", rc);

        return lua_yield(L, 0);
    }

#if (nginx_version >= 1002006 && nginx_version < 1003000) ||                 \
        nginx_version >= 1003009
    r->main->count--;
    dd("decrement r->main->count: %d", (int) r->main->count);
#endif

    if (rc == NGX_AGAIN) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua read buffered request body requires I/O "
                       "interruptions");
        ctx->waiting_more_body = 1;
        ctx->downstream = coctx;

        ngx_http_lua_cleanup_pending_operation(coctx);
        coctx->cleanup = ngx_http_lua_req_body_cleanup;
        coctx->data = r;
        return lua_yield(L, 0);
    }

    /* rc == NGX_OK */
    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua has read buffered request body in a single run");

    return 0;
}
```

### 2. `ngx.req.discard_body`

```lua
ngx.req.discard_body()
```

丢弃请求包体，读取连接上的请求数据，并立即丢弃。函数是异步调用，立即返回。如果已经读取请求包体，函数无影响立即返回。实现：

```c
static int
ngx_http_lua_ngx_req_discard_body(lua_State *L)
{
    // ... 忽略无关代码
    // 参数判断
    n = lua_gettop(L);
    if (n != 0) {
        return luaL_error(L, "expecting 0 arguments but seen %d", n);
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "request object not found");
    }

    ngx_http_lua_check_fake_request(L, r);
    // 丢弃处理
    rc = ngx_http_discard_request_body(r);
    if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        return luaL_error(L, "failed to discard request body");
    }

    return 0;
}

ngx_int_t
ngx_http_discard_request_body(ngx_http_request_t *r)
{
    // ... 忽略无关代码

#if (NGX_HTTP_V2)
    if (r->stream) {
        r->stream->skip_data = 1;
        return NGX_OK;
    }
#endif

    if (ngx_http_test_expect(r) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    rev = r->connection->read;
    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, rev->log, 0, "http set discard body");

    if (rev->timer_set) {
        ngx_del_timer(rev);
    }
    if (r->headers_in.content_length_n <= 0 && !r->headers_in.chunked) {
        return NGX_OK;
    }

    size = r->header_in->last - r->header_in->pos;

    if (size || r->headers_in.chunked) {
        rc = ngx_http_discard_request_body_filter(r, r->header_in);

        if (rc != NGX_OK) {
            return rc;
        }

        if (r->headers_in.content_length_n == 0) {
            return NGX_OK;
        }
    }

    rc = ngx_http_read_discarded_request_body(r);
    if (rc == NGX_OK) {
        r->lingering_close = 0;
        return NGX_OK;
    }

    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        return rc;
    }

    /* rc == NGX_AGAIN */
    // 设置处理函数为丢弃请求数据
    r->read_event_handler = ngx_http_discarded_request_body_handler;

    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    r->count++;
    r->discard_body = 1;

    return NGX_OK;
}
```

### 3. `ngx.req.get_body_data`

```lua
data = ngx.req.get_body_data()
```

获取存储在内存中的请求体数据，返回字符串。为了正确使用函数的功能，需要保证调用前已经使用 `ngx.req.read_body` 或者通过配置 `lua_need_request_body` 读取请求包体；同时，请求包体使用内存存储而非存储在临时文件中（通过配置实现）。

在 `NGINX` 配置指令中，`client_body_buffer_size` 配置指令用来控制接收包体内存缓冲区的大小，`client_max_body_size` 用来控制可以接收的最大包体限制。如果两者相同，接收的请求包体必定在内存中，使用 `ngx.req.get_body_data` 可以将请求数据读取出来。

```c
static int
ngx_http_lua_ngx_req_get_body_data(lua_State *L)
{
    // ... 忽略无关代码
    // 参数判断
    n = lua_gettop(L);
    if (n != 0) {
        return luaL_error(L, "expecting 0 arguments but seen %d", n);
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "request object not found");
    }

    ngx_http_lua_check_fake_request(L, r);
    // 无请求包体、请求包体存储在临时文件中，返回 nil
    if (r->request_body == NULL
        || r->request_body->temp_file
        || r->request_body->bufs == NULL)
    {
        lua_pushnil(L);
        return 1;
    }

    // 请求包体在一个 buf 中存储，直接使用 buf 生成 lua string
    cl = r->request_body->bufs;

    if (cl->next == NULL) {
        len = cl->buf->last - cl->buf->pos;
        if (len == 0) {
            lua_pushnil(L);
            return 1;
        }
        lua_pushlstring(L, (char *) cl->buf->pos, len);
        return 1;
    }

    /* found multi-buffer body */
    // 需要遍历 chain，计算长度、并分配缓冲区，将所有的 buf 拷贝在一起，返回 string
    len = 0;
    for (; cl; cl = cl->next) {
        dd("body chunk len: %d", (int) ngx_buf_size(cl->buf));
        len += cl->buf->last - cl->buf->pos;
    }

    if (len == 0) {
        lua_pushnil(L);
        return 1;
    }

    buf = (u_char *) lua_newuserdata(L, len);
    p = buf;
    for (cl = r->request_body->bufs; cl; cl = cl->next) {
        p = ngx_copy(p, cl->buf->pos, cl->buf->last - cl->buf->pos);
    }

    lua_pushlstring(L, (char *) buf, len);
    return 1;
}
```

### 4. `ngx.req.get_body_file`

```lua
file_name = ngx.req.get_body_file()
```

获得使用临时文件存储请求体的文件名。与 `ngx.req.get_body_data` 函数相似，需要使用 `ngx.req.read_body` 或 `lua_need_request_body` 配置读取包体。

使用 `NGINX` 配置指令 `client_body_in_file_only` 可以将请求包体保存在临时文件中。

```c
static int
ngx_http_lua_ngx_req_get_body_file(lua_State *L)
{
    // 参数判断
    n = lua_gettop(L);
    if (n != 0) {
        return luaL_error(L, "expecting 0 arguments but seen %d", n);
    }
    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "request object not found");
    }
    ngx_http_lua_check_fake_request(L, r);

    if (r->request_body == NULL || r->request_body->temp_file == NULL) {
        lua_pushnil(L);
        return 1;
    }

    // 返回文件名
    lua_pushlstring(L, (char *) r->request_body->temp_file->file.name.data,
                    r->request_body->temp_file->file.name.len);
    return 1;
}
```

### 5. `ngx.req.set_body_data`

```lua
ngx.set_body_data(data)
```

使用 `data` 参数指定当前请求的请求体。函数调用前需要读取请求包体（包体存储在内存或者文件无所谓），函数会将原始请求清理（释放内存、请求临时文件设置清理回调函数）。

```c
static int
ngx_http_lua_ngx_req_set_body_data(lua_State *L)
{
    // 参数判断
    n = lua_gettop(L);

    if (n != 1) {
        return luaL_error(L, "expecting 1 arguments but seen %d", n);
    }

    // 要设置的包体
    body.data = (u_char *) luaL_checklstring(L, 1, &body.len);

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "request object not found");
    }

    ngx_http_lua_check_fake_request(L, r);

    // 已经执行 discard_body
    if (r->discard_body) {
        return luaL_error(L, "request body already discarded asynchronously");
    }

    // 未读入请求体
    if (r->request_body == NULL) {
        return luaL_error(L, "request body not read yet");
    }

    rb = r->request_body;

    tag = (ngx_buf_tag_t) &ngx_http_lua_module;
    // 临时文件清理函数
    tf = rb->temp_file;
    if (tf) {
        if (tf->file.fd != NGX_INVALID_FILE) {

            dd("cleaning temp file %.*s", (int) tf->file.name.len,
               tf->file.name.data);

            ngx_http_lua_pool_cleanup_file(r->pool, tf->file.fd);
            tf->file.fd = NGX_INVALID_FILE;

            dd("temp file cleaned: %.*s", (int) tf->file.name.len,
               tf->file.name.data);
        }

        rb->temp_file = NULL;
    }

    if (body.len == 0) {

        // 释放原始 buf 缓冲区
        if (rb->bufs) {
            for (cl = rb->bufs; cl; cl = cl->next) {
                if (cl->buf->tag == tag && cl->buf->temporary) {

                    dd("free old request body buffer: size:%d",
                       (int) ngx_buf_size(cl->buf));

                    ngx_pfree(r->pool, cl->buf->start);
                    cl->buf->tag = (ngx_buf_tag_t) NULL;
                    cl->buf->temporary = 0;
                }
            }
        }

        rb->bufs = NULL;
        rb->buf = NULL;

        dd("request body is set to empty string");
        goto set_header;
    }

    if (rb->bufs) {
        // 释放原始缓冲区
        for (cl = rb->bufs; cl; cl = cl->next) {
            if (cl->buf->tag == tag && cl->buf->temporary) {
                dd("free old request body buffer: size:%d",
                   (int) ngx_buf_size(cl->buf));

                ngx_pfree(r->pool, cl->buf->start);
                cl->buf->tag = (ngx_buf_tag_t) NULL;
                cl->buf->temporary = 0;
            }
        }

        // 创建一块新的缓冲区，用来存储新设置的包体
        rb->bufs->next = NULL;
        b = rb->bufs->buf;
        ngx_memzero(b, sizeof(ngx_buf_t));
        b->temporary = 1;
        b->tag = tag;

        b->start = ngx_palloc(r->pool, body.len);
        if (b->start == NULL) {
            return luaL_error(L, "no memory");
        }
        b->end = b->start + body.len;

        b->pos = b->start;
        b->last = ngx_copy(b->pos, body.data, body.len);

    } else {
        // 创建缓冲区存储新设置的包体
        rb->bufs = ngx_alloc_chain_link(r->pool);
        if (rb->bufs == NULL) {
            return luaL_error(L, "no memory");
        }
        rb->bufs->next = NULL;

        b = ngx_create_temp_buf(r->pool, body.len);
        if (b == NULL) {
            return luaL_error(L, "no memory");
        }

        b->tag = tag;
        b->last = ngx_copy(b->pos, body.data, body.len);

        rb->bufs->buf = b;
        rb->buf = b;
    }

set_header:
    // 更新 content-length 头
    /* override input header Content-Length (value must be null terminated) */
    value.data = ngx_palloc(r->pool, NGX_SIZE_T_LEN + 1);
    if (value.data == NULL) {
        return luaL_error(L, "no memory");
    }

    value.len = ngx_sprintf(value.data, "%uz", body.len) - value.data;
    value.data[value.len] = '\0';

    dd("setting request Content-Length to %.*s (%d)",
       (int) value.len, value.data, (int) body.len);

    r->headers_in.content_length_n = body.len;
  
    if (r->headers_in.content_length) {
        r->headers_in.content_length->value.data = value.data;
        r->headers_in.content_length->value.len = value.len;
    } else {
        ngx_str_set(&key, "Content-Length");
        rc = ngx_http_lua_set_input_header(r, key, value, 1 /* override */);
        if (rc != NGX_OK) {
            return luaL_error(L, "failed to reset the Content-Length "
                              "input header");
        }
    }

    return 0;
}
```

### 6. `ngx.req.set_body_file`

```lua
ngx.req.set_body_file(file_name, auto_clean?)
```

使用 `file_name` 参数指定的文件作为请求体，可选布尔类型参数 `auto_clean` 用来控制释放需要自动清理文件（默认为 `false`）。函数调用前需要读取请求包体（包体存储在内存或者文件无所谓），函数会将原始请求清理（释放内存、请求临时文件设置清理回调函数）。

```c
static int
ngx_http_lua_ngx_req_set_body_file(lua_State *L)
{
    // ... 省略无关代码
    // 参数检查
    n = lua_gettop(L);
    if (n != 1 && n != 2) {
        return luaL_error(L, "expecting 1 or 2 arguments but seen %d", n);
    }
    // 文件名
    p = (u_char *) luaL_checklstring(L, 1, &name.len);
    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request found");
    }

    ngx_http_lua_check_fake_request(L, r);

    // 是否丢弃了请求体
    if (r->discard_body) {
        return luaL_error(L, "request body already discarded asynchronously");
    }

    // 是否未读取请求体
    if (r->request_body == NULL) {
        return luaL_error(L, "request body not read yet");
    }

    // 文件名
    name.data = ngx_palloc(r->pool, name.len + 1);
    if (name.data == NULL) {
        return luaL_error(L, "no memory");
    }

    ngx_memcpy(name.data, p, name.len);
    name.data[name.len] = '\0';

    // 自动清理
    if (n == 2) {
        luaL_checktype(L, 2, LUA_TBOOLEAN);
        clean = lua_toboolean(L, 2);
    } else {
        clean = 0;
    }

    dd("clean: %d", (int) clean);

    rb = r->request_body;

    /* clean up existing r->request_body->bufs (if any) */

    tag = (ngx_buf_tag_t) &ngx_http_lua_module;

    if (rb->bufs) {
        dd("XXX reusing buf");

        // 释放缓冲区
        for (cl = rb->bufs; cl; cl = cl->next) {
            if (cl->buf->tag == tag && cl->buf->temporary) {
                dd("free old request body buffer: size:%d",
                   (int) ngx_buf_size(cl->buf));

                ngx_pfree(r->pool, cl->buf->start);
                cl->buf->tag = (ngx_buf_tag_t) NULL;
                cl->buf->temporary = 0;
            }
        }

        rb->bufs->next = NULL;
        b = rb->bufs->buf;

        ngx_memzero(b, sizeof(ngx_buf_t));

        b->tag = tag;
        rb->buf = NULL;

    } else {

        dd("XXX creating new buf");

        rb->bufs = ngx_alloc_chain_link(r->pool);
        if (rb->bufs == NULL) {
            return luaL_error(L, "no memory");
        }
        rb->bufs->next = NULL;

        b = ngx_calloc_buf(r->pool);
        if (b == NULL) {
            return luaL_error(L, "no memory");
        }

        b->tag = tag;

        rb->bufs->buf = b;
        rb->buf = NULL;
    }

    b->last_in_chain = 1;

    /* just make r->request_body->temp_file a bare stub */

    tf = rb->temp_file;

    if (tf) {
        // 原始的临时文件清理
        if (tf->file.fd != NGX_INVALID_FILE) {

            dd("cleaning temp file %.*s", (int) tf->file.name.len,
               tf->file.name.data);

            ngx_http_lua_pool_cleanup_file(r->pool, tf->file.fd);

            ngx_memzero(tf, sizeof(ngx_temp_file_t));

            tf->file.fd = NGX_INVALID_FILE;

            dd("temp file cleaned: %.*s", (int) tf->file.name.len,
               tf->file.name.data);
        }
    } else {
        tf = ngx_pcalloc(r->pool, sizeof(ngx_temp_file_t));
        if (tf == NULL) {
            return luaL_error(L, "no memory");
        }

        tf->file.fd = NGX_INVALID_FILE;
        rb->temp_file = tf;
    }

    /* read the file info and construct an in-file buf */
    // 打开新的文件
    ngx_memzero(&of, sizeof(ngx_open_file_info_t));

    of.directio = NGX_OPEN_FILE_DIRECTIO_OFF;

    if (ngx_http_lua_open_and_stat_file(name.data, &of, r->connection->log)
        != NGX_OK)
    {
        return luaL_error(L, "%s \"%s\" failed", of.failed, name.data);
    }

    dd("XXX new body file fd: %d", of.fd);

    tf->file.fd = of.fd;
    tf->file.name = name;
    tf->file.log = r->connection->log;
    tf->file.directio = 0;

    if (of.size == 0) {
        // 设置自动清理，在此处已经删除了文件，
        // 只不过进程持有文件描述符，文件的 inode 被删除，但是文件内容还在。
        if (clean) {
            if (ngx_delete_file(name.data) == NGX_FILE_ERROR) {
                err = ngx_errno;

                if (err != NGX_ENOENT) {
                    ngx_log_error(NGX_LOG_CRIT, r->connection->log, err,
                                  ngx_delete_file_n " \"%s\" failed",
                                  name.data);
                }
            }
        }

        if (ngx_close_file(of.fd) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_ALERT, r->connection->log, ngx_errno,
                          ngx_close_file_n " \"%s\" failed", name.data);
        }

        r->request_body->bufs = NULL;
        r->request_body->buf = NULL;

        goto set_header;
    }

    /* register file cleanup hook */

    cln = ngx_pool_cleanup_add(r->pool,
                               sizeof(ngx_pool_cleanup_file_t));

    if (cln == NULL) {
        return luaL_error(L, "no memory");
    }

    cln->handler = clean ? ngx_pool_delete_file : ngx_pool_cleanup_file;
    clnf = cln->data;

    clnf->fd = of.fd;
    clnf->name = name.data;
    clnf->log = r->pool->log;

    b->file = &tf->file;
    if (b->file == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    dd("XXX file size: %d", (int) of.size);

    b->file_pos = 0;
    b->file_last = of.size;

    b->in_file = 1;

    dd("buf file: %p, f:%u", b->file, b->in_file);

set_header:

    /* override input header Content-Length (value must be null terminated) */
    // 设置 content-length 头
    value.data = ngx_palloc(r->pool, NGX_OFF_T_LEN + 1);
    if (value.data == NULL) {
        return luaL_error(L, "no memory");
    }

    value.len = ngx_sprintf(value.data, "%O", of.size) - value.data;
    value.data[value.len] = '\0';

    r->headers_in.content_length_n = of.size;

    if (r->headers_in.content_length) {
        r->headers_in.content_length->value.data = value.data;
        r->headers_in.content_length->value.len = value.len;
    } else {

        ngx_str_set(&key, "Content-Length");

        rc = ngx_http_lua_set_input_header(r, key, value, 1 /* override */);
        if (rc != NGX_OK) {
            return luaL_error(L, "failed to reset the Content-Length "
                              "input header");
        }
    }

    return 0;
}
```

### 7. `ngx.req.init_body`

```lua
ngx.req.init_body(buffer_size?)
```

为当前请求创建一个新的空白请求体，后续可以通过 `ngx.req.append_body`、 `ngx.req.finish_body` 向缓冲区写数据。可选参数 `buffer_size` 用来控制初始化缓冲区的大小，其默认值为 `client_body_buffer_size` 指令设置的大小。

在使用 `ngx.req.append_body` 写请求数据上，如果缓冲区超过了 `buffer_size` 大小，会将请求数据完整的写到临时文件中。

```c
static int
ngx_http_lua_ngx_req_init_body(lua_State *L)
{
    // check arg number
    n = lua_gettop(L);
    if (n != 1 && n != 0) {
        return luaL_error(L, "expecting 0 or 1 argument but seen %d", n);
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request found");
    }

    ngx_http_lua_check_fake_request(L, r);

    // should not discard body
    if (r->discard_body) {
        return luaL_error(L, "request body already discarded asynchronously");
    }

    // should read in body data
    if (r->request_body == NULL) {
        return luaL_error(L, "request body not read yet");
    }

    // has buffer_size param
    if (n == 1) {
        num = luaL_checkinteger(L, 1);
        if (num < 0) {
            return luaL_error(L, "bad size argument: %d", (int) num);
        }

        size = (size_t) num;

    } else {
        // get location's client_body_buffer_size conf size
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
        size = clcf->client_body_buffer_size;
    }

    // save request data in temp file
    if (size == 0) {
        r->request_body_in_file_only = 1;
    }

    rb = r->request_body;

#if 1
    tf = rb->temp_file;

    if (tf) {
        // add indenpent clean up handler original request body temp file
        if (tf->file.fd != NGX_INVALID_FILE) {

            dd("cleaning temp file %.*s", (int) tf->file.name.len,
               tf->file.name.data);

            ngx_http_lua_pool_cleanup_file(r->pool, tf->file.fd);

            ngx_memzero(tf, sizeof(ngx_temp_file_t));

            tf->file.fd = NGX_INVALID_FILE;

            dd("temp file cleaned: %.*s", (int) tf->file.name.len,
               tf->file.name.data);
        }

        rb->temp_file = NULL;
    }
#endif

    r->request_body_in_clean_file = 1;

    r->headers_in.content_length_n = 0;

    // alloc new body data buf size
    rb->buf = ngx_create_temp_buf(r->pool, size);
    if (rb->buf == NULL) {
        return luaL_error(L, "no memory");
    }

    rb->bufs = ngx_alloc_chain_link(r->pool);
    if (rb->bufs == NULL) {
        return luaL_error(L, "no memory");
    }

    rb->bufs->buf = rb->buf;
    rb->bufs->next = NULL;

    return 0;
}
```

### 8. `ngx.req.append_body`

```lua
ngx.req.append_body(data_chunk)
```

将 `data_chunk` 参数指定的新数据块添加到当前请求的请求包体中，请求包体必须要由 `ngx.req.init_body` 创建（使用 `ngx.req.get_body_data` 虽然获取了请求包体，但是包体缓冲区链表最后指向的是 `NULL` 无法直接使用）。

```c
static int
ngx_http_lua_ngx_req_append_body(lua_State *L)
{
    ngx_http_request_t          *r;
    int                          n;
    ngx_http_request_body_t     *rb;
    ngx_str_t                    body;
    size_t                       size, rest;
    size_t                       offset = 0;
    ngx_chain_t                  chain;
    ngx_buf_t                    buf;

    // check arg number
    n = lua_gettop(L);

    if (n != 1) {
        return luaL_error(L, "expecting 1 arguments but seen %d", n);
    }

    // data chunk param
    body.data = (u_char *) luaL_checklstring(L, 1, &body.len);

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request found");
    }

    ngx_http_lua_check_fake_request(L, r);

    // must have request_body
    // and requst_body_buf has buf
    if (r->request_body == NULL
        || r->request_body->buf == NULL
        || r->request_body->bufs == NULL)
    {
        return luaL_error(L, "request_body not initialized");
    }

    // write into temp file
    if (r->request_body_in_file_only) {
        buf.start = body.data;
        buf.pos = buf.start;
        buf.last = buf.start + body.len;
        buf.end = buf.last;
        buf.temporary = 1;

        chain.buf = &buf;
        chain.next = NULL;

        if (ngx_http_lua_write_request_body(r, &chain) != NGX_OK) {
            return luaL_error(L, "fail to write file");
        }

        r->headers_in.content_length_n += body.len;
        return 0;
    }

    // write into memory buf
    rb = r->request_body;

    rest = body.len;

    while (rest > 0) {
        if (rb->buf->last == rb->buf->end) {
            if (ngx_http_lua_write_request_body(r, rb->bufs) != NGX_OK) {
                return luaL_error(L, "fail to write file");
            }

            rb->buf->last = rb->buf->start;
        }

        size = rb->buf->end - rb->buf->last;

        if (size > rest) {
            size = rest;
        }

        ngx_memcpy(rb->buf->last, body.data + offset, size);

        rb->buf->last += size;
        rest -= size;
        offset += size;
        r->headers_in.content_length_n += size;
    }

    return 0;
}
```

### 9. `ngx.req.finish_body`

```lua
ngx.req.finish_body()
```

构建请求包体完成操作，如果使用临时文件存储请求数据，将缓冲区中数据写入临时文件，更新 `content-length` 头。

```c
static int
ngx_http_lua_ngx_req_body_finish(lua_State *L)
{
    // ... omit
    n = lua_gettop(L);
    if (n != 0) {
        return luaL_error(L, "expecting 0 argument but seen %d", n);
    }
    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request found");
    }

    ngx_http_lua_check_fake_request(L, r);

    if (r->request_body == NULL
        || r->request_body->buf == NULL
        || r->request_body->bufs == NULL)
    {
        return luaL_error(L, "request_body not initialized");
    }

    rb = r->request_body;

    if (rb->temp_file) {

        /* save the last part */

        if (ngx_http_lua_write_request_body(r, rb->bufs) != NGX_OK) {
            return luaL_error(L, "fail to write file");
        }

        b = ngx_calloc_buf(r->pool);
        if (b == NULL) {
            return luaL_error(L, "no memory");
        }

        b->in_file = 1;
        b->file_pos = 0;
        b->file_last = rb->temp_file->file.offset;
        b->file = &rb->temp_file->file;

        if (rb->bufs->next) {
            rb->bufs->next->buf = b;

        } else {
            rb->bufs->buf = b;
        }
    }

    /* override input header Content-Length (value must be null terminated) */
    // update content-length header
    value.data = ngx_palloc(r->pool, NGX_SIZE_T_LEN + 1);
    if (value.data == NULL) {
        return luaL_error(L, "no memory");
    }

    size = (size_t) r->headers_in.content_length_n;

    value.len = ngx_sprintf(value.data, "%uz", size) - value.data;
    value.data[value.len] = '\0';

    dd("setting request Content-Length to %.*s (%d)", (int) value.len,
       value.data, (int) size);

    if (r->headers_in.content_length) {
        r->headers_in.content_length->value.data = value.data;
        r->headers_in.content_length->value.len = value.len;
    } else {
        ngx_str_set(&key, "Content-Length");
        rc = ngx_http_lua_set_input_header(r, key, value, 1 /* override */);
        if (rc != NGX_OK) {
            return luaL_error(L, "failed to reset the Content-Length "
                              "input header");
        }
    }

    return 0;
}
```

## 六 请求套接字

### 1. `ngx.req.socket`

```lua
 tcpsock, err = ngx.req.socket(raw)
```

返回与客户端之间只读的 `cosocket`，只支持 `receive` 和 `receiveuntil` 函数。出错返回 `nil`，以及错误描述字符串。

使用 `cosocket` 可以使用流方式读取请求包体，不过不能与 `ngx.req.read_body`、`ngx.req.discard_body` 混用（`lua_need_request_body` 指令同样不能使用）。**不支持 `chunked` 传输编码**。

可选布尔参数 `raw` 用来控制是否返回原始套接字，如果为 `true` 将返回全双工的套接字，支持 `receive`、`receiveuntil`、`send` 函数。全双工的套接字需要保证无输出缓存数据（由 `ngx.say`、`ngx.print` 产生），可以在调用 `ngx.req.socket` 前调用 `ngx.flush(true)`  刷出待输出数据。

**不支持 `HTTP2`、`SPDY` 协议**。

```c
static int
ngx_http_lua_req_socket(lua_State *L)
{
    // ... omit other code
    n = lua_gettop(L);
    if (n == 0) {
        raw = 0;
    } else if (n == 1) {
        raw = lua_toboolean(L, 1);
        lua_pop(L, 1);
    } else {
        return luaL_error(L, "expecting zero arguments, but got %d",
                          lua_gettop(L));
    }

    r = ngx_http_lua_get_req(L);

    if (r != r->main) {
        return luaL_error(L, "attempt to read the request body in a "
                          "subrequest");
    }

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

    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT);

    c = r->connection;

    if (raw) {
#if !defined(nginx_version) || nginx_version < 1003013
        lua_pushnil(L);
        lua_pushliteral(L, "nginx version too old");
        return 2;
#else
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
        if (r->request_body) {
            lua_pushnil(L);
            lua_pushliteral(L, "request body already exists");
            return 2;
        }

        if (r->discard_body) {
            lua_pushnil(L);
            lua_pushliteral(L, "request body discarded");
            return 2;
        }

        dd("req content length: %d", (int) r->headers_in.content_length_n);

        if (r->headers_in.content_length_n <= 0) {
            lua_pushnil(L);
            lua_pushliteral(L, "no body");
            return 2;
        }

        // 对 except 请求头处理
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

    // 元表: receive、receiveuntil、send、send 元表
    lua_createtable(L, 3 /* narr */, 1 /* nrec */); /* the object */

    if (raw) {
        lua_pushlightuserdata(L, &ngx_http_lua_raw_req_socket_metatable_key);
    } else {
        lua_pushlightuserdata(L, &ngx_http_lua_req_socket_metatable_key);
    }

    lua_rawget(L, LUA_REGISTRYINDEX);
    lua_setmetatable(L, -2);

    // 创建 cosocket
    u = lua_newuserdata(L, sizeof(ngx_http_lua_socket_tcp_upstream_t));
    if (u == NULL) {
        return luaL_error(L, "no memory");
    }

#if 1
    lua_pushlightuserdata(L, &ngx_http_lua_downstream_udata_metatable_key);
    lua_rawget(L, LUA_REGISTRYINDEX);
    lua_setmetatable(L, -2);
#endif

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
    // 超时时间
    u->read_timeout = u->conf->read_timeout;
    u->connect_timeout = u->conf->connect_timeout;
    u->send_timeout = u->conf->send_timeout;

    // 清理函数
    cln = ngx_http_lua_cleanup_add(r, 0);
    if (cln == NULL) {
        u->ft_type |= NGX_HTTP_LUA_SOCKET_FT_ERROR;
        lua_pushnil(L);
        lua_pushliteral(L, "no memory");
        return 2;
    }

    cln->handler = ngx_http_lua_socket_tcp_cleanup;
    cln->data = u;
    u->cleanup = &cln->handler;

    // 与客户端的网络连接
    pc = &u->peer;

    pc->log = c->log;
    pc->log_error = NGX_ERROR_ERR;

    pc->connection = c;

    dd("setting data to %p", u);

    coctx->data = u;
    ctx->downstream = u;

    // 删除读超时定时器
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

    if (raw) {
        if (c->write->timer_set) {
            // 删除写超时定时器
            ngx_del_timer(c->write);
        }
    }

    lua_settop(L, 1);
    return 1;
}
```

## 七 请求方法

### 1. `ngx.req.get_method`

```lua
method_name = ngx.req.get_method()
```

获得当前请求的请求方法，返回值为字符串类型，子请求会返回相应子请求的方法。

### 2. `ngx.req.set_method`

```lua
ngx.req.set_method(method_id)
```

设置当前请求的请求方法，参数为数值类型。可选参数：

```lua
ngx.HTTP_GET
ngx.HTTP_HEAD
ngx.HTTP_PUT
ngx.HTTP_POST
ngx.HTTP_DELETE
ngx.HTTP_OPTIONS   (added in the v0.5.0rc24 release)
ngx.HTTP_MKCOL     (added in the v0.8.2 release)
ngx.HTTP_COPY      (added in the v0.8.2 release)
ngx.HTTP_MOVE      (added in the v0.8.2 release)
ngx.HTTP_PROPFIND  (added in the v0.8.2 release)
ngx.HTTP_PROPPATCH (added in the v0.8.2 release)
ngx.HTTP_LOCK      (added in the v0.8.2 release)
ngx.HTTP_UNLOCK    (added in the v0.8.2 release)
ngx.HTTP_PATCH     (added in the v0.8.2 release)
ngx.HTTP_TRACE     (added in the v0.8.2 release)
```

## 八 请求时间

### 1. `ngx.req.start_time`

```lua
secs = ngx.req.start_time()
```

返回当前请求创建的时间戳，浮点类型数值，精确到毫秒级。

## 九 内部请求判断

### 1. `ngx.req.is_internal`

```lua
is_internal = ngx.req.is_internal()
```

判断当前请求是否是内部请求。