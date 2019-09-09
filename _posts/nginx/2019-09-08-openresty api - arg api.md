---
layout: post
title: OpenResty Api - arg api
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description: 
---

## 一 `ngx.arg`

`arg api` 只包含一个接口 `ngx.arg`，`ngx.arg` 可以在两个阶段使用：`set_by_lua` 和 `body_filter_by_lua`。

### 1. 在 `set_by_lua` 中使用

在 `set_by_lua` 阶段 `ngx.arg` 是一个**只读表**，保存指令的输入参数。

```lua
location /foo {
		set $a 1;
  	set $b 2;
  	set $c 3;

  	set_by_lua $sum
  			'return tonumber(ngx.arg[1]) + tonumber(ngx.arg[2]) + tonumber(ngx.arg[3])'
  			$a $b $c;

  	echo $sum;
}
```

这是 `lua-nginx-module` 中的一段示例，在 `set_by_lua` 指令中通过 `ngx.arg[1]` 获得 `$a` 变量，`ngx.arg[2]` 获得 `$b` 变量，`ngx.arg[3]` 获得 `$c` 变量。

### 2. 在 `body_filter_by_lua` 中使用

当在 `body_filter_by_lua` 阶段使用 `ngx.arg` 表时，`ngx.arg[1]` 持有调用 `lua-nginx` 模块 `body_filter` 函数传入的数据块，`ngx.arg[2]` 用来标记整个输出流是否结束（布尔类型）。在 `body_filter_by_lua` 阶段，`ngx.arg` 是可以修改的，`ngx.arg[1]` 设置为 `nil` 或者 `""` 后续的 `body_filter` 将不会收的数据块。同时还可以修改 `ngx.arg[2]` 控制下游 `body_filter` 是否已经发送完毕。在 `body_filter_by_lua` 中只能使用 `1` 和 `2` 两个索引。

## 二 实现

`ngx.arg` 是 `table` 类型数据，在 `lua-nginx` 中通过设置其 `__index` 和 `__newindex` 方法来实现获得参数、包体、设置包体和 `eof` 状态的功能。注册 `ngx.arg` 函数：

```c
static void
ngx_http_lua_inject_arg_api(lua_State *L)
{
    lua_pushliteral(L, "arg");
    lua_newtable(L);    /*  .arg table aka {} */

    lua_createtable(L, 0 /* narr */, 2 /* nrec */);    /*  the metatable */

    lua_pushcfunction(L, ngx_http_lua_param_get);
    lua_setfield(L, -2, "__index");

    lua_pushcfunction(L, ngx_http_lua_param_set);
    lua_setfield(L, -2, "__newindex");

    lua_setmetatable(L, -2);    /*  tie the metatable to param table */

    dd("top: %d, type -1: %s", lua_gettop(L), luaL_typename(L, -1));

    lua_rawset(L, -3);    /*  set ngx.arg table */
}
```

`__index` 方法同时支持 `set_by_lua` 和 `body_filter_by_lua` 功能，用来获取参数、应答包体、应答包体 `eof` 状态。`__newindex` 方法仅支持 `body_filter_by_lua` 阶段，用来设置应答包体、应答包体 `eof` 状态。

### 1.  `set_by_lua` 参数获取

在 `set_by_lua` 中参数的获取最终实现函数是 `ngx_http_lua_setby_param_get`。在使用 `ngx.arg[idx]` 进行获取参数时所有参数以及保存在数组中，通过 `ngx_http_lua_args_key` 伪索引获得数组。因为在 `C` 中索引从 `0` 开始，因此传入的索引 `idx` 需要进行减一操作。

```c
int
ngx_http_lua_setby_param_get(lua_State *L)
{
    int         idx;
    int         n;

    ngx_http_variable_value_t       *v;

    idx = luaL_checkint(L, 2);
    idx--;

    /*  get number of args from globals */
    lua_getglobal(L, ngx_http_lua_nargs_key);
    n = (int) lua_tointeger(L, -1);

    // 取出参数数组 v, v 中保存所有的参数，索引从零开始，所以一开始 idx 进行减一操作
    /*  get args from globals */
    lua_getglobal(L, ngx_http_lua_args_key);
    v = lua_touserdata(L, -1);

    if (idx < 0 || idx > n - 1) {
        lua_pushnil(L);
    } else {
        // 取出相应索引的值，放在栈顶
        lua_pushlstring(L, (const char *) (v[idx].data), v[idx].len);
    }

    return 1;
}
```

### 2. `body_filter_by_lua` 获取数据

在 `body_filter_by_lua` 中获取应答包体、应答 `eof` 状态的实现函数是 `ngx_http_lua_body_filter_param_get`。函数仅允许使用 `1` 或 `2` 作为索引，当索引为 `2` 时仅需要判断输出链 `ngx_chain_t` 类型数据是否为空即可返还布尔类型应答值以标识应答是否结束；当索引为 `1` 时需要遍历输出链，计算长度，并将其拷贝到缓冲区进行返回。

```c
// body filter 阶段获取待输出数据
// ngx.arg[1] ngx.arg[2]
int
ngx_http_lua_body_filter_param_get(lua_State *L)
{
    u_char              *data, *p;
    size_t               size;
    ngx_chain_t         *cl;
    ngx_buf_t           *b;
    int                  idx;
    ngx_chain_t         *in;

    idx = luaL_checkint(L, 2);

    dd("index: %d", idx);

    // 仅允许使用 ngx.arg[1] ngx.arg[2]
    if (idx != 1 && idx != 2) {
        lua_pushnil(L);
        return 1;
    }

    // 取出输出链
    lua_getglobal(L, ngx_http_lua_chain_key);
    in = lua_touserdata(L, -1);

    // 判断是否已经结束（eof）
    if (idx == 2) {
        /* asking for the eof argument */

        for (cl = in; cl; cl = cl->next) {
            if (cl->buf->last_buf || cl->buf->last_in_chain) {
                lua_pushboolean(L, 1);
                return 1;
            }
        }

        lua_pushboolean(L, 0);
        return 1;
    }

    // 访问数据
    /* idx == 1 */

    size = 0;

    if (in == NULL) {
        /* being a cleared chain on the Lua land */
        lua_pushliteral(L, "");
        return 1;
    }

    if (in->next == NULL) {

        dd("seen only single buffer");

        b = in->buf;
        lua_pushlstring(L, (char *) b->pos, b->last - b->pos);
        return 1;
    }

    dd("seen multiple buffers");

    for (cl = in; cl; cl = cl->next) {
        b = cl->buf;

        size += b->last - b->pos;

        if (b->last_buf || b->last_in_chain) {
            break;
        }
    }

    data = (u_char *) lua_newuserdata(L, size);

    for (p = data, cl = in; cl; cl = cl->next) {
        b = cl->buf;
        p = ngx_copy(p, b->pos, b->last - b->pos);

        if (b->last_buf || b->last_in_chain) {
            break;
        }
    }

    lua_pushlstring(L, (char *) data, size);
    return 1;
}
```

### 3. `body_filter_by_lua` 设置数据

在 `body_filter_by_lua` 中设置应答包体、应答 `eof` 状态的实现函数是 `ngx_http_lua_body_filter_param_set`。

```c
int
ngx_http_lua_body_filter_param_set(lua_State *L, ngx_http_request_t *r,
    ngx_http_lua_ctx_t *ctx)
{
    int                      type;
    int                      idx;
    int                      found;
    u_char                  *data;
    size_t                   size;
    unsigned                 last;
    unsigned                 flush = 0;
    ngx_buf_t               *b;
    ngx_chain_t             *cl;
    ngx_chain_t             *in;

    idx = luaL_checkint(L, 2);

    dd("index: %d", idx);

    if (idx != 1 && idx != 2) {
        return luaL_error(L, "bad index: %d", idx);
    }

    // 设置 eof 状态
    if (idx == 2) {
        /* overwriting the eof flag */
        last = lua_toboolean(L, 3);

        lua_getglobal(L, ngx_http_lua_chain_key);
        in = lua_touserdata(L, -1);
        lua_pop(L, 1);

        // 需要设置 eof == true，标记应答结束
        if (last) {
            ctx->seen_last_in_filter = 1;

            /* the "in" chain cannot be NULL and we set the "last_buf" or
             * "last_in_chain" flag in the last buf of "in" */

            for (cl = in; cl; cl = cl->next) {
                if (cl->next == NULL) {
                    if (r == r->main) {
                        cl->buf->last_buf = 1;

                    } else {
                        cl->buf->last_in_chain = 1;
                    }

                    break;
                }
            }

        } else {
            // 需要设置 eof == false，去除应答结束标记
            /* last == 0 */

            found = 0;

            for (cl = in; cl; cl = cl->next) {
                b = cl->buf;

                if (b->last_buf) {
                    b->last_buf = 0;
                    found = 1;
                }

                if (b->last_in_chain) {
                    b->last_in_chain = 0;
                    found = 1;
                }

                if (found && b->last == b->pos && !ngx_buf_in_memory(b)) {
                    /* make it a special sync buf to make
                     * ngx_http_write_filter_module happy. */
                    b->sync = 1;
                }
            }

            ctx->seen_last_in_filter = 0;
        }

        return 0;
    }

    // 应答包体处理
    /* idx == 1, overwriting the chunk data */

    type = lua_type(L, 3);

    switch (type) {
    case LUA_TSTRING:
    case LUA_TNUMBER:
        data = (u_char *) lua_tolstring(L, 3, &size);
        break;

    case LUA_TNIL:
        /* discard the buffers */
        // 从 _G 中获取 __ngx_cl 到栈顶
        lua_getglobal(L, ngx_http_lua_chain_key); /* key val */
        in = lua_touserdata(L, -1);
        lua_pop(L, 1);

        last = 0;

        for (cl = in; cl; cl = cl->next) {
            b = cl->buf;

            if (b->flush) {
                flush = 1;
            }

            if (b->last_in_chain || b->last_buf) {
                last = 1;
            }

            dd("mark the buf as consumed: %d", (int) ngx_buf_size(b));
            b->pos = b->last;
        }

        /* cl == NULL */

        goto done;

    case LUA_TTABLE:
        size = ngx_http_lua_calc_strlen_in_table(L, 3 /* index */, 3 /* arg */,
                                                 1 /* strict */);
        data = NULL;
        break;

    default:
        return luaL_error(L, "bad chunk data type: %s",
                          lua_typename(L, type));
    }

    lua_getglobal(L, ngx_http_lua_chain_key);
    in = lua_touserdata(L, -1);
    lua_pop(L, 1);

    last = 0;

    for (cl = in; cl; cl = cl->next) {
        b = cl->buf;

        if (b->flush) {
            flush = 1;
        }

        if (b->last_buf || b->last_in_chain) {
            last = 1;
        }

        dd("mark the buf as consumed: %d", (int) ngx_buf_size(cl->buf));
        cl->buf->pos = cl->buf->last;
    }

    /* cl == NULL */

    if (size == 0) {
        goto done;
    }

    cl = ngx_http_lua_chain_get_free_buf(r->connection->log, r->pool,
                                         &ctx->free_bufs, size);
    if (cl == NULL) {
        return luaL_error(L, "no memory");
    }

    // 拷贝应答内容到发送缓冲区
    if (type == LUA_TTABLE) {
        cl->buf->last = ngx_http_lua_copy_str_in_table(L, 3, cl->buf->last);

    } else {
        cl->buf->last = ngx_copy(cl->buf->pos, data, size);
    }

done:
// 更新 last_buf 标记
    if (last || flush) {
        if (cl == NULL) {
            cl = ngx_http_lua_chain_get_free_buf(r->connection->log,
                                                 r->pool,
                                                 &ctx->free_bufs, 0);
            if (cl == NULL) {
                return luaL_error(L, "no memory");
            }
        }

        if (last) {
            ctx->seen_last_in_filter = 1;

            if (r == r->main) {
                cl->buf->last_buf = 1;

            } else {
                cl->buf->last_in_chain = 1;
            }
        }

        if (flush) {
            cl->buf->flush = 1;
        }
    }

    // 更新 _G 中 __ngx_cl 值为最新值
    lua_pushlightuserdata(L, cl);
    lua_setglobal(L, ngx_http_lua_chain_key);
    return 0;
}
```

**在 `body_filter_by_lua` 中通过 `ngx.arg[1]` 设置应答包体时可以使用 `table` 作为值，但是 `table` 必须是数组类型，非哈希表。**

