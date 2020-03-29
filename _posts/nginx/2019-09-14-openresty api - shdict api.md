---
layout: post
title: OpenResty Api - shdict api
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description:
---

## 一 概述

`shdict api` 实现了共享内存的操作接口，在 `lua-nginx` 模块中将所有的共享内存（`shm_zone`）存储在一个 `Table` 中，每个 `shm_zone` 使用红黑树存储操作的 `key-value`。**刚才说所有的共享内存，准确的说应该是所有使用 `lua_shared_dict` 指令创建的共享内存**。

`lua-nginx` 模块中每个 `shm_zone` 用 `Table` 表示，在表中使用 `SHDICT_USERDATA_INDEX` 索引每个共享内存的 `zone`。

```c
void
ngx_http_lua_inject_shdict_api(ngx_http_lua_main_conf_t *lmcf, lua_State *L)
{
    ngx_http_lua_shdict_ctx_t   *ctx;
    ngx_uint_t                   i;
    ngx_shm_zone_t             **zone;

    if (lmcf->shm_zones != NULL) {
        // 共享内存 shared 元表
        lua_createtable(L, 0, lmcf->shm_zones->nelts /* nrec */);
                /* ngx.shared */

        // 共享内存（每块 shm_zone）操作元表
        lua_createtable(L, 0 /* narr */, 18 /* nrec */); /* shared mt */

        // ... 省略元方法定义

        lua_pushvalue(L, -1); /* shared mt mt */
        lua_setfield(L, -2, "__index"); /* shared mt */

        // 遍历所有的 shm_zone，设置其元表，并将其保存在 shared 表中
        zone = lmcf->shm_zones->elts;
        for (i = 0; i < lmcf->shm_zones->nelts; i++) {
            ctx = zone[i]->data;

            lua_pushlstring(L, (char *) ctx->name.data, ctx->name.len);/* shared mt key */

            lua_createtable(L, 1 /* narr */, 0 /* nrec */); /* table of zone[i] */
            lua_pushlightuserdata(L, zone[i]); /* shared mt key ud */
            lua_rawseti(L, -2, SHDICT_USERDATA_INDEX); /* {zone[i]} */
            lua_pushvalue(L, -3); /* shared mt key ud mt */
            lua_setmetatable(L, -2); /* shared mt key ud */
            lua_rawset(L, -4); /* shared mt */
        }

        lua_pop(L, 1); /* shared */
    } else {
        lua_newtable(L);    /* ngx.shared */
    }

    lua_setfield(L, -2, "shared");
}
```

## 二 接口

### 1. `get|get_stale`

```lua
value, flags = ngx.shared.DICT:get(key)
value, flags, stale = ngx.shared.DICT:get_stale(key)
```

`get` 函数用来从共享内存中获得 `key` 对应的值，如果 `key` 不存在或者已经过期则返回 `nil`。出错时返回 `nil` 以及错误描述字符串。

`get_stale` 函数与 `get` 相同，唯一不同点在于使用 `get_stale` 函数访问过期的 `key-value` 时返回其原始值，而非 `nil`。同时，返回值中 `stale` 标识 `key` 是否已经过期。

**`key` 必须为字符串类型，`value` 可以为字符串、数值、布尔或者 `NIL`**。

`get`、`get_stale` 函数最终都是有 `ngx_http_lua_shdict_get_helper` 实现，只是 `get` 调用时 `get_stale` 为 0，`get_stale` 调用时值为 1。`ngx_http_lua_shdict_get_helper` 函数实现：

```c
static int
ngx_http_lua_shdict_get_helper(lua_State *L, int get_stale)
{
    // ... omit other code
    // 参数判断
    n = lua_gettop(L);
    if (n != 2) {
        return luaL_error(L, "expecting exactly two arguments, "
                          "but only seen %d", n);
    }

    // shm_zone
    if (lua_type(L, 1) != LUA_TTABLE) {
        return luaL_error(L, "bad \"zone\" argument");
    }

    zone = ngx_http_lua_shdict_get_zone(L, 1);
    if (zone == NULL) {
        return luaL_error(L, "bad \"zone\" argument");
    }

    ctx = zone->data;
    name = ctx->name;

    // key 
    if (lua_isnil(L, 2)) {
        lua_pushnil(L);
        lua_pushliteral(L, "nil key");
        return 2;
    }

    key.data = (u_char *) luaL_checklstring(L, 2, &key.len);

    if (key.len == 0) {
        lua_pushnil(L);
        lua_pushliteral(L, "empty key");
        return 2;
    }
    // key 长度判断
    if (key.len > 65535) {
        lua_pushnil(L);
        lua_pushliteral(L, "key too long");
        return 2;
    }

    hash = ngx_crc32_short(key.data, key.len);
    // 上锁
    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
    if (!get_stale) {
        // 过期 key 处理
        // n == 1 会尝试删除一个或两个过期 key
        // n == 0 删除一个最久未被访问的 key
        ngx_http_lua_shdict_expire(ctx, 1);
    }
#endif

    // 查找 key 节点
    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returns %d", (int) rc);

    if (rc == NGX_DECLINED || (rc == NGX_DONE && !get_stale)) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        lua_pushnil(L);
        return 1;
    }

    /* rc == NGX_OK || (rc == NGX_DONE && get_stale) */

    value_type = sd->value_type;

    dd("data: %p", sd->data);
    dd("key len: %d", (int) sd->key_len);

    value.data = sd->data + sd->key_len;
    value.len = (size_t) sd->value_len;

    switch (value_type) {
    case SHDICT_TSTRING:
        lua_pushlstring(L, (char *) value.data, value.len);
        break;

    case SHDICT_TNUMBER:
        if (value.len != sizeof(double)) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);
            return luaL_error(L, "bad lua number value size found for key %s "
                              "in shared_dict %s: %lu", key.data, name.data,
                              (unsigned long) value.len);
        }

        ngx_memcpy(&num, value.data, sizeof(double));
        lua_pushnumber(L, num);
        break;

    case SHDICT_TBOOLEAN:
        if (value.len != sizeof(u_char)) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);
            return luaL_error(L, "bad lua boolean value size found for key %s "
                              "in shared_dict %s: %lu", key.data, name.data,
                              (unsigned long) value.len);
        }

        c = *value.data;
        lua_pushboolean(L, c ? 1 : 0);
        break;

    case SHDICT_TLIST:
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        lua_pushnil(L);
        lua_pushliteral(L, "value is a list");
        return 2;

    default:
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        return luaL_error(L, "bad value type found for key %s in "
                          "shared_dict %s: %d", key.data, name.data,
                          value_type);
    }

    user_flags = sd->user_flags;
    ngx_shmtx_unlock(&ctx->shpool->mutex);

    if (get_stale) {
        /* always return value, flags, stale */
        if (user_flags) {
            lua_pushinteger(L, (lua_Integer) user_flags);
        } else {
            lua_pushnil(L);
        }

        lua_pushboolean(L, rc == NGX_DONE);
        return 3;
    }

    if (user_flags) {
        lua_pushinteger(L, (lua_Integer) user_flags);
        return 2;
    }

    return 1;
}
```



### 2. `set`

```lua
 success, err, forcible = ngx.shared.DICT:set(key, value, exptime?, flags?)
```

无条件的向共享内存中添加一条 `key-value`。返回值 `success` 用以标明添加是否成功，`err` 为错误描述字符串，布尔类型的 `forcible` 用来标明为了保存本条记录是否将其他合法的 `key-value` 删除。

参数 `key`、`value` 是要保存的 `key-value`，可选的数值参数 `exptime` 指定 `key` 的过期时间，单位为秒但是可以使用小数精确到毫秒级别。`exptime` 默认为 0，不过期。可选的参数 `flag` 为 `key` 的“用户”表示，在使用 `get|get_stale` 获取 `key` 时会返回 `flag`。`flag` 是 32 为的整数。

在调用 `set` 时，如果共享内存已经无可用空间，会使用 `LRU` 策略，从 `lru_queue` 中删除 `key-value`（仍然有可能无法满足 `set` 调用使用的空间）。`set` 实现函数为 `ngx_http_lua_shdict_set_helper`，实现函数：

```c
// flag 控制 set 的行为，以实现 set、safe_set 功能
static int
ngx_http_lua_shdict_set_helper(lua_State *L, int flags)
{
    // ... omit other code
    // 参数
    n = lua_gettop(L);
    if (n != 3 && n != 4 && n != 5) {
        return luaL_error(L, "expecting 3, 4 or 5 arguments, but only seen %d", n);
    }

    // shm_zone table
    if (lua_type(L, 1) != LUA_TTABLE) {
        return luaL_error(L, "bad \"zone\" argument");
    }
    zone = ngx_http_lua_shdict_get_zone(L, 1);
    if (zone == NULL) {
        return luaL_error(L, "bad \"zone\" argument");
    }
    ctx = zone->data;

    // key
    if (lua_isnil(L, 2)) {
        lua_pushnil(L);
        lua_pushliteral(L, "nil key");
        return 2;
    }

    key.data = (u_char *) luaL_checklstring(L, 2, &key.len);

    // ... key 长度判断，省略其他代码

    hash = ngx_crc32_short(key.data, key.len);

    // value
    value_type = lua_type(L, 3);
    switch (value_type) {
    case SHDICT_TSTRING:
        value.data = (u_char *) lua_tolstring(L, 3, &value.len);
        break;

    case SHDICT_TNUMBER:
        value.len = sizeof(double);
        num = lua_tonumber(L, 3);
        value.data = (u_char *) &num;
        break;

    case SHDICT_TBOOLEAN:
        value.len = sizeof(u_char);
        c = lua_toboolean(L, 3) ? 1 : 0;
        value.data = &c;
        break;

    case LUA_TNIL:
        if (flags & (NGX_HTTP_LUA_SHDICT_ADD|NGX_HTTP_LUA_SHDICT_REPLACE)) {
            lua_pushnil(L);
            lua_pushliteral(L, "attempt to add or replace nil values");
            return 2;
        }

        ngx_str_null(&value);
        break;

    default:
        lua_pushnil(L);
        lua_pushliteral(L, "bad value type");
        return 2;
    }

    // 超时时间
    if (n >= 4) {
        exptime = luaL_checknumber(L, 4);
        if (exptime < 0) {
            return luaL_error(L, "bad \"exptime\" argument");
        }
    }

    // 用户标识
    if (n == 5) {
        user_flags = (uint32_t) luaL_checkinteger(L, 5);
    }

    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
    // 尝试删除一个或两个过期 key-value
    ngx_http_lua_shdict_expire(ctx, 1);
#endif

    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returned %d", (int) rc);

    if (flags & NGX_HTTP_LUA_SHDICT_REPLACE) {

        if (rc == NGX_DECLINED || rc == NGX_DONE) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "not found");
            lua_pushboolean(L, forcible);
            return 3;
        }
        /* rc == NGX_OK */
        goto replace;
    }

    if (flags & NGX_HTTP_LUA_SHDICT_ADD) {
        // 已经存在
        if (rc == NGX_OK) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);
            lua_pushboolean(L, 0);
            lua_pushliteral(L, "exists");
            lua_pushboolean(L, forcible);
            return 3;
        }

        if (rc == NGX_DONE) {
            /* exists but expired */
            dd("go to replace");
            goto replace;
        }

        /* rc == NGX_DECLINED */
        dd("go to insert");
        goto insert;
    }

    if (rc == NGX_OK || rc == NGX_DONE) {
        if (value_type == LUA_TNIL) {
            goto remove;
        }
replace:

        if (value.data
            && value.len == (size_t) sd->value_len
            && sd->value_type != SHDICT_TLIST)
        {
            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                           "lua shared dict set: found old entry and value "
                           "size matched, reusing it");
            // 插入 lru_queue 队首
            ngx_queue_remove(&sd->queue);
            ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

            sd->key_len = (u_short) key.len;

            if (exptime > 0) {
                tp = ngx_timeofday();
                sd->expires = (uint64_t) tp->sec * 1000 + tp->msec
                              + (uint64_t) (exptime * 1000);
            } else {
                sd->expires = 0;
            }

            sd->user_flags = user_flags;
            sd->value_len = (uint32_t) value.len;
            dd("setting value type to %d", value_type);
            sd->value_type = (uint8_t) value_type;
            p = ngx_copy(sd->data, key.data, key.len);
            ngx_memcpy(p, value.data, value.len);

            ngx_shmtx_unlock(&ctx->shpool->mutex);
            lua_pushboolean(L, 1);
            lua_pushnil(L);
            lua_pushboolean(L, forcible);
            return 3;
        }

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict set: found old entry but value size "
                       "NOT matched, removing it first");

remove:

        if (sd->value_type == SHDICT_TLIST) {
            queue = ngx_http_lua_shdict_get_list_head(sd, key.len);
            for (q = ngx_queue_head(queue);
                 q != ngx_queue_sentinel(queue);
                 q = ngx_queue_next(q))
            {
                p = (u_char *) ngx_queue_data(q,
                                              ngx_http_lua_shdict_list_node_t,
                                              queue);

                ngx_slab_free_locked(ctx->shpool, p);
            }
        }

        ngx_queue_remove(&sd->queue);
        node = (ngx_rbtree_node_t *)
                   ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

        ngx_rbtree_delete(&ctx->sh->rbtree, node);
        ngx_slab_free_locked(ctx->shpool, node);
    }

insert:
    /* rc == NGX_DECLINED or value size unmatch */
    if (value.data == NULL) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        lua_pushboolean(L, 1);
        lua_pushnil(L);
        lua_pushboolean(L, 0);
        return 3;
    }

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                   "lua shared dict set: creating a new entry");

    n = offsetof(ngx_rbtree_node_t, color)
        + offsetof(ngx_http_lua_shdict_node_t, data)
        + key.len
        + value.len;
    node = ngx_slab_alloc_locked(ctx->shpool, n);
    if (node == NULL) {
        if (flags & NGX_HTTP_LUA_SHDICT_SAFE_STORE) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);
            lua_pushboolean(L, 0);
            lua_pushliteral(L, "no memory");
            return 2;
        }

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict set: overriding non-expired items "
                       "due to memory shortage for entry \"%V\"", &key);
        // 尝试强制删除访问时间最远的 30 个 key-value 以满足 新增 key-value
        for (i = 0; i < 30; i++) {
            if (ngx_http_lua_shdict_expire(ctx, 0) == 0) {
                break;
            }

            forcible = 1;

            node = ngx_slab_alloc_locked(ctx->shpool, n);
            if (node != NULL) {
                goto allocated;
            }
        }

        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 0);
        lua_pushliteral(L, "no memory");
        lua_pushboolean(L, forcible);
        return 3;
    }

allocated:

    sd = (ngx_http_lua_shdict_node_t *) &node->color;
    node->key = hash;
    sd->key_len = (u_short) key.len;

    if (exptime > 0) {
        tp = ngx_timeofday();
        sd->expires = (uint64_t) tp->sec * 1000 + tp->msec
                      + (uint64_t) (exptime * 1000);
    } else {
        sd->expires = 0;
    }

    sd->user_flags = user_flags;
    sd->value_len = (uint32_t) value.len;
    dd("setting value type to %d", value_type);
    sd->value_type = (uint8_t) value_type;
    p = ngx_copy(sd->data, key.data, key.len);
    ngx_memcpy(p, value.data, value.len);

    ngx_rbtree_insert(&ctx->sh->rbtree, node);
    // 插入 lru_queue 队首
    ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushboolean(L, 1);
    lua_pushnil(L);
    lua_pushboolean(L, forcible);
    return 3;
}
```

`ngx_http_lua_shdict_set_helper` 函数实现了新增、更新、删除功能，逻辑非常复杂。

### 3. `safe_set`

```lua
ok, err = ngx.shared.DICT:safe_set(key, value, exptime?, flags?)
```

与 `set` 方法相同，只不过不会删除未过期 `key-value` 以满足保存操作。

### 4. `add`

```lua
success, err, forcible = ngx.shared.DICT:add(key, value, exptime?, flags?)
```

与 `set` 方法相似，不过只会在共享内存中不存在 `key` 时才会保存设置的 `key-value`。如果 `key` 已经在共享内存中，`success` 为 `false`，`err` 为 `exists`。有可能会删除未过期 `key-value`，以满足存储需求。

### 5. `safe_add`

```lua
ok, err = ngx.shared.DICT:safe_add(key, value, exptime?, flags?)
```

与 `add` 方法类似，不过不会删除未过期的 `key-value` 以满足存储需求。

### 6. `replace`

```lua
success, err, forcible = ngx.shared.DICT:replace(key, value, exptime?, flags?)
```

与 `set` 方法类似，不过仅当 `key` 已经存在于共享内存时才会进行保存操作。如果 `key` 不存在时会返回 `not found` 错误描述字符串。

### 7. `incr`

```lua
newval, err, forcible? = ngx.shared.DICT:incr(key, value, init?, init_ttl?)
```

将 `key` 指定的数值类型值增加 `value` 步长，返回增加后的 `value`。如果 `key` 在共享内存中未找到，`init` 可选参数影响其行为：

- `init` 为 `nil`，函数返回 `nil`，并且错误描述为 `not found`。

- `init` 为数值参数，函数返回 `init + value`。

与 `add` 函数类似，如果存储空间不足，会删除未过期 `key-value`。

可选参数 `init_ttl` 控制 `key` 的超时时间（仅在初始化时参数有效），以秒为单位，支持浮点数可以精确到毫秒。

### 8. `delete`

```lua
ngx.shared.DICT:delete(key)
```

无条件的删除 `key` 指定的 `key-value`，等价于调用 `set(key, nil)` 函数。

### 9. `lpush`

```lua
length, err = ngx.shared.DICT:lpush(key, value)
```

将指定的数值或字符串 `value` 插入基于共享内存的列表头，返回 `list` 中的元素数量。如果 `key` 不存在，则在执行当前函后会创建一个新的列表，当 `key` 已经存在但是非列表类型，错误提示为 `value not a list`。

**`lpush` 不会删除未过期 `key-value`**。

### 10. `rpush`

```lua
length, err = ngx.shared.DICT:rpush(key, value)
```

与 `lpush` 相似，只不过 `rpush` 在队列尾插入值。

### 11. `lpop`

```lua
val, err = ngx.shared.DICT:lpop(key)
```

从 `list` 中返回并删除首个元素，如果 `key` 不存在返回 `nil`，如果 `key` 相应的 `value` 非 `list` 类型，返回 `value not a list`。

### 12. `rpop`

```lua
val, err = ngx.shared.DICT:rpop(key)
```

从 `list` 中返回并删除末尾元素，如果 `key` 不存在返回 `nil`，如果 `key` 相应的 `value` 非 `list` 类型，返回 `value not a list`。

### 13. `llen`

```lua
len, err = ngx.shared.DICT:llen(key)
```

计算 `list` 中元素数量，如果 `key` 不存在代表是空列表，返回 0。如果 `key` 的值非列表类型，返回 `nil` 以及相应错误描述。

### 14. `ttl`

```lua
ttl, err = ngx.shared.DICT:ttl(key)
```

返回 `key` 剩余的 `TTL` 时间，以秒为单位，精确到毫秒。失败返回 `nil` 以及错误描述字符串。如果返回 `ttl` 为零，说明未设置超时时间。

### 15. `expire`

```lua
success, err = ngx.shared.DICT:expire(key, exptime)
```

更新 `key` 的超时时间，返回布尔类型状态信息以及错误描述。参数 `exptime` 以秒为单位精确到毫秒。

### 16. `flush_all|flush_expired`

```lua
ngx.shared.DICT:flush_all()
flushed = ngx.shared.DICT:flush_expired(max_count?)
```

删除所有的 `key-value` 或者过期的 `key-value`。`flush_expired` 函数接受一个可选的 `max_count` 参数，用来控制删除过期 `key` 数量。默认为 0，会删除所有的过期 `key`。同时，`flush_expired` 函数会返回删除的 `key` 数量。

### 17. `get_keys`

```lua
keys = ngx.shared.DICT:get_keys(max_count?)
```

返回一个包含共享内存（shm_zone）所有 `keys` 的 `Array Table`。可选数值参数 `max_count` 可以控制返回的 `key` 数量，默认为 1024 个。