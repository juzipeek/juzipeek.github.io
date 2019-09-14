---
layout: post
title: OpenResty Api - resp_header api
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description:
---

## 一 概述

resp_header api 主要用来读取、设置应答头。

## 二 接口

### 1. `ngx.header`

通过 `ngx.header.HEADER_NAME` 可以获取或设置应答 `HEADER_NAME` 的值，`lua-nginx` 模块对 `ngx.header` 表的 `__index` 和 `__newindex` 方法做了修改实现了应答头的获取和设置功能。在将头设置为 `nil` 时可以实现删除应答头的功能。**`HEADER_NAME` 是大小写无关的，同时默认会将下划线（_）替换为中划线（-）**。

`__index` 元方法：

```c
static int
ngx_http_lua_ngx_header_get(lua_State *L)
{
    // ... omit other code
    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ngx_http_lua_check_fake_request(L, r);

    /* we skip the first argument that is the table */
    p = (u_char *) luaL_checklstring(L, 2, &len);

    dd("key: %.*s, len %d", (int) len, p, (int) len);

    // 将应答头名中 "_" 替换为 "-"
    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);
    if (llcf->transform_underscores_in_resp_headers
        && memchr(p, '_', len) != NULL)
    {
        key.data = (u_char *) lua_newuserdata(L, len);
        if (key.data == NULL) {
            return luaL_error(L, "no memory");
        }

        /* replace "_" with "-" */
        for (i = 0; i < len; i++) {
            c = p[i];
            if (c == '_') {
                c = '-';
            }
            key.data[i] = c;
        }

    } else {
        key.data = p;
    }

    key.len = len;

    return ngx_http_lua_get_output_header(L, r, &key);
}

// 获取应答头
int
ngx_http_lua_get_output_header(lua_State *L, ngx_http_request_t *r,
    ngx_str_t *key)
{
    // ... omit other code
    dd("looking for response header \"%.*s\"", (int) key->len, key->data);

    // 判断是否是内建 header
    switch (key->len) {
    case 14:
        if (r->headers_out.content_length == NULL
            && r->headers_out.content_length_n >= 0
            && ngx_strncasecmp(key->data, (u_char *) "Content-Length", 14) == 0)
        {
            lua_pushinteger(L, (lua_Integer) r->headers_out.content_length_n);
            return 1;
        }

        break;

    case 12:
        if (r->headers_out.content_type.len
            && ngx_strncasecmp(key->data, (u_char *) "Content-Type", 12) == 0)
        {
            lua_pushlstring(L, (char *) r->headers_out.content_type.data,
                            r->headers_out.content_type.len);
            return 1;
        }

        break;

    default:
        break;
    }

    dd("not a built-in output header");

    found = 0;

#if 1
    if (r->headers_out.location
        && r->headers_out.location->value.len
        && r->headers_out.location->value.data[0] == '/')
    {
        /* XXX ngx_http_core_find_config_phase, for example,
         * may not initialize the "key" and "hash" fields
         * for a nasty optimization purpose, and
         * we have to work-around it here */

        r->headers_out.location->hash = ngx_http_lua_location_hash;
        ngx_str_set(&r->headers_out.location->key, "Location");
    }
#endif

    // 遍历 list
    part = &r->headers_out.headers.part;
    h = part->elts;

    for (i = 0; /* void */; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }

            part = part->next;
            h = part->elts;
            i = 0;
        }

        if (h[i].hash == 0) {
            continue;
        }

        // 匹配时进行大小写无关
        if (h[i].hash != 0
            && h[i].key.len == key->len
            && ngx_strncasecmp(key->data, h[i].key.data, h[i].key.len) == 0)
         {
             if (!found) {
                 found = 1;

                 lua_pushlstring(L, (char *) h[i].value.data, h[i].value.len);
                 continue;
             }

             if (found == 1) {
                 lua_createtable(L, 4 /* narr */, 0);
                 lua_insert(L, -2);
                 lua_rawseti(L, -2, found);
             }

             found++;

             lua_pushlstring(L, (char *) h[i].value.data, h[i].value.len);
             lua_rawseti(L, -2, found);
         }
    }

    if (found) {
        return 1;
    }

    lua_pushnil(L);
    return 1;
}
```

`__newindex` 元方法更复杂些，`HEADER_NAME` 处理简单，只需要获取栈上的参数即可。`HEADER_VALUE` 处理复杂些，需要判断类型，支持 `String`、`NIL`、`String Array Table`。`ngx_http_lua_set_output_header` 是应答头更新函数，在其中将应答头的更新抽象为回调函数并存储在数组中。内建的头有独立的回调函数，通用头使用通用头回调函数。

```c
// __newindex 入口
static int
ngx_http_lua_ngx_header_set(lua_State *L)
{
    // ... omit other code
    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no ctx");
    }

    ngx_http_lua_check_fake_request(L, r);
    // 头已经发送出去
    if (r->header_sent || ctx->header_sent) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "attempt to "
                      "set ngx.header.HEADER after sending out "
                      "response headers");
        return 0;
    }

    // header name
    /* we skip the first argument that is the table */
    p = (u_char *) luaL_checklstring(L, 2, &len);

    dd("key: %.*s, len %d", (int) len, p, (int) len);

    key.data = ngx_palloc(r->pool, len + 1);
    if (key.data == NULL) {
        return luaL_error(L, "no memory");
    }

    ngx_memcpy(key.data, p, len);
    key.data[len] = '\0';
    key.len = len;

    // 下划线到中划线转换配置
    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);
    if (llcf->transform_underscores_in_resp_headers) {
        /* replace "_" with "-" */
        p = key.data;
        for (i = 0; i < len; i++) {
            if (p[i] == '_') {
                p[i] = '-';
            }
        }
    }

    if (!ctx->headers_set) {
        rc = ngx_http_lua_set_content_type(r);
        if (rc != NGX_OK) {
            return luaL_error(L,
                              "failed to set default content type: %d",
                              (int) rc);
        }

        ctx->headers_set = 1;
    }

    // header value
    if (lua_type(L, 3) == LUA_TNIL) {
        ngx_str_null(&value);
    } else if (lua_type(L, 3) == LUA_TTABLE) {
        n = luaL_getn(L, 3);
        if (n == 0) {
            ngx_str_null(&value);
        } else {
            for (i = 1; i <= n; i++) {
                dd("header value table index %d", (int) i);

                lua_rawgeti(L, 3, i);
                p = (u_char *) luaL_checklstring(L, -1, &len);

                value.data = ngx_palloc(r->pool, len);
                if (value.data == NULL) {
                    return luaL_error(L, "no memory");
                }

                ngx_memcpy(value.data, p, len);
                value.len = len;

                // i == 1 非常重要
                // 如果一直为 true 那么 multi header 表只会有最后一个 值被设置
                rc = ngx_http_lua_set_output_header(r, key, value,
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
        p = (u_char *) luaL_checklstring(L, 3, &len);
        value.data = ngx_palloc(r->pool, len);
        if (value.data == NULL) {
            return luaL_error(L, "no memory");
        }

        ngx_memcpy(value.data, p, len);
        value.len = len;
    }

    dd("key: %.*s, value: %.*s",
       (int) key.len, key.data, (int) value.len, value.data);

    rc = ngx_http_lua_set_output_header(r, key, value, 1 /* override */);

    if (rc == NGX_ERROR) {
        return luaL_error(L, "failed to set header %s (error: %d)",
                          key.data, (int) rc);
    }

    return 0;
}

static ngx_http_lua_set_header_t  ngx_http_lua_set_handlers[] = {
    { ngx_string("Server"),
                 offsetof(ngx_http_headers_out_t, server),
                 ngx_http_set_builtin_header },

    // ... omit other code
    { ngx_null_string, 0, ngx_http_set_header }
};

ngx_int_t
ngx_http_lua_set_output_header(ngx_http_request_t *r, ngx_str_t key,
    ngx_str_t value, unsigned override)
{
    // ... omit other code
    // header 操作表
    ngx_http_lua_set_header_t        *handlers = ngx_http_lua_set_handlers;

    dd("set header value: %.*s", (int) value.len, value.data);

    hv.hash = ngx_hash_key_lc(key.data, key.len);
    hv.key = key;
    hv.offset = 0;
    hv.no_override = !override;
    hv.handler = NULL;

    // 大小写无关匹配
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

    if (handlers[i].name.len == 0 && handlers[i].handler) {
        hv.offset = handlers[i].offset;
        hv.handler = handlers[i].handler;
    }

#if 1
    if (hv.handler == NULL) {
        return NGX_ERROR;
    }
#endif

    return hv.handler(r, &hv, &value);
}
```

### 2. `ngx.resp`

```lua
headers, err = ngx.resp.get_headers(max_headers?, raw?)
```

返回一个包含当前请求所有应答头的 `Table`。可选数值参数 `max_header` 用来控制返回 `header` 的数量，默认为 100,布尔类型参数 `raw` 用来控制返回的 `Table` 是否有 `__index` 元方法（`HEADER_NAME` 大小写无关，并且将下划线转为中划线）。

