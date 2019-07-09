---
layout: post
title: OpenResty Api 1
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description: 
---

## 一 概述

本文主要介绍 `OpenResty` 中各 `Api` 功能，加深学习印象。

## 二 `ngx.arg`

`ngx.arg` 在 `body filter` 阶段用来读取、更新应答数据。其中 `ngx.arg[1]` 是待发送的 `body`，`ngx.arg[2]` 指示后续是否还有待发送数据。

`ngx.arg` 变量是 `table` 类型，只支持 `__index`（获取）和 `__newindex`（更新）两个元方法。`__index` 方法由 `ngx_http_lua_body_filter_param_get` 函数最终实现，`__newindex` 方法由 `ngx_http_lua_body_filter_param_set` 函数最终实现。

### 1. 获取函数

```c
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

    if (idx != 1 && idx != 2) {
        lua_pushnil(L);
        return 1;
    }
		// ngx_http_lua_chain_key：
    // 从全局获得值并压入栈
    lua_getglobal(L, ngx_http_lua_chain_key);
    // 将栈顶元素出栈转换为 userdata（ngx_chain_t*）类型，
    // 此内容为 body 的 ngx_chain_t
    in = lua_touserdata(L, -1);

    // 如果调用 ngx.arg[2],使用来判断是否发送结束
    if (idx == 2) {
        /* asking for the eof argument */

        for (cl = in; cl; cl = cl->next) {
            // 最后的 buf，说明发送完毕
            if (cl->buf->last_buf || cl->buf->last_in_chain) {
                lua_pushboolean(L, 1);
                return 1;
            }
        }

        lua_pushboolean(L, 0);
        return 1;
    }

    /* idx == 1 */
		// ngx.arg[1] 用来取应答包体（ngx_chain_t）中内容
    size = 0;

    if (in == NULL) {
        /* being a cleared chain on the Lua land */
        lua_pushliteral(L, "");
        return 1;
    }

    // 应答内容在一个 buf 中，直接压入栈顶，返回
    if (in->next == NULL) {

        dd("seen only single buffer");

        b = in->buf;
        lua_pushlstring(L, (char *) b->pos, b->last - b->pos);
        return 1;
    }

    dd("seen multiple buffers");

    // 计算应答内容的大小
    for (cl = in; cl; cl = cl->next) {
        b = cl->buf;

        size += b->last - b->pos;

        if (b->last_buf || b->last_in_chain) {
            break;
        }
    }

  	// 创建 size 大小缓冲区（userdata 类型），并压入栈顶
    data = (u_char *) lua_newuserdata(L, size);

  	// 将应答内容拷贝到缓冲区
    for (p = data, cl = in; cl; cl = cl->next) {
        b = cl->buf;
        p = ngx_copy(p, b->pos, b->last - b->pos);

        if (b->last_buf || b->last_in_chain) {
            break;
        }
    }

    // 将栈上由 data 指向的 size 长度字符压入栈顶
    // Lua 会生成给定字符串的内部副本，在函数返回后可以删除 data 指向的内容
    lua_pushlstring(L, (char *) data, size);
    return 1;
}
```

### 2. 更新函数

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

  	// 检查第二个参数是否是 int，如果是 int 则返回值
    // 其实还是操作的栈，判断栈中从栈底开始第二个元素是否是 int 类型并返回
    idx = luaL_checkint(L, 2);

    dd("index: %d", idx);

    if (idx != 1 && idx != 2) {
        return luaL_error(L, "bad index: %d", idx);
    }

    // 更新发送结束标识
    if (idx == 2) {
        /* overwriting the eof flag */
        last = lua_toboolean(L, 3);
				// 获得应答 body 的 ngx_chain_t 数据
        lua_getglobal(L, ngx_http_lua_chain_key);
        in = lua_touserdata(L, -1);
        // 将从栈顶开始向下的两个元素 pop 出栈(并未使用)
        lua_pop(L, 1);

        if (last) {
            ctx->seen_last_in_filter = 1;

            /* the "in" chain cannot be NULL and we set the "last_buf" or
             * "last_in_chain" flag in the last buf of "in" */
						// 设置应答 body 的 ngx_chain_t 中的标识为结束
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
            /* last == 0 */
						// 设置应答 body 的 ngx_chain_t 中的标识为未结束
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

    /* idx == 1, overwriting the chunk data */
		// 更新发送内容
    // 获得从栈底(1)开始，第三个索引处的数据类型
    type = lua_type(L, 3);

    switch (type) {
    case LUA_TSTRING:
    case LUA_TNUMBER:
        // 直接转换为字符串，方便更新发送缓冲区
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

    // 从 _G 中获取 __ngx_cl 内容到栈顶
    lua_getglobal(L, ngx_http_lua_chain_key);
    in = lua_touserdata(L, -1);
    lua_pop(L, 1); // 清空除栈底两个元素外的其他元素

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

## 三 Lua C API

### 1. 栈结构

在 `C` 与 `Lua` 之间传递数据是通过栈来实现，栈中的内容 `TValue` 类型（同时包含类型与值信息），在 `C`、`Lua` 间可以识别转换。栈顶可以用 -1 索引，栈底可以用 1 索引

调用 `ngx.arg[2] = 1` 时栈结构：
```
---------------------------------------
| 正数索引 |  数据   | 负数索引 | 栈中位置 |
---------------------------------------
|    2    |   2    |   -1    |  栈顶   |
|    1    |  _G    |   -2    |  栈底   |
```

栈中 `_G` 是在 `body filter` 指令处理函数（`ngx_http_lua_body_filter_inline` 或 `ngx_http_lua_body_filter_file`）中设置的栈底环境变量（`_G` 全局 `table`）。

### 2. 函数说明

`lua_isnumber`、`lua_isstring` 系列函数并不会判断参数类型，他们会进行类型转换，如果转换成功返回其值。这样的话，对于 `string` 类型可以调用 `lua_isnumber` 函数。

## 四 参考资料

- [lua_newuserdata](https://www.lua.org/pil/28.1.html)
- [lua_pushlstring](https://www.lua.org/pil/24.2.1.html)
- [lua_pop](https://www.lua.org/pil/24.2.3.html)
