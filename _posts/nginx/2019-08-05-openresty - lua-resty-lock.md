---
layout: post
title: OpenResty lib - lua-resty-lock
category: NGINX
tags: NGINX,OpenResty,Lua
keywords: NGINX,OpenResty,Lua
description: 
---

## 一 概述

`lua-resty-lock` 使用共享内存构建的非阻塞锁，锁等待非阻塞是通过 `ngx.sleep` 函数实现的，当无法获取锁时会不断调用 `ngx.sleep`，直至成功获取锁或超时失败。*ngx.sleep 是非阻塞的，会将当前协程放回事件循环中。*

## 二 API 接口

### 1. `new`

```lua
syntax: obj, err = lock:new(dict_name)
syntax: obj, err = lock:new(dict_name, opts)
```

通过 `dict_name` 指定的共享内存名称创建锁对象实例。可选的 `opts` 是一个 `table` 类型参数，可以定义锁的行为：

| 参数       | 含义                                                | 备注                |
| ---------- | --------------------------------------------------- | ------------------- |
| `exptime`  | 共享内存中锁过期时间；超过时间后锁会被删除          | 单位秒；默认 30s    |
| `timeout`  | 锁的 `lock` 方法最大等待事件；如果为零则不等待      | 单位秒；默认 5s     |
| `step`     | 指定等待锁时休眠的初始步长，它会按 `ratio` 逐步增大 | 单位秒；默认 0.001s |
| `ratio`    | 指定步长递增比例，默认是 2，步长加倍                |                     |
| `max_step` | 最大步长                                            | 单位秒；默认 0.5s   |

### 2. `lock`

```lua
syntax: elapsed, err = obj:lock(key)
```

尝试给 `key` 加锁，`obj` 是 `new` 函数返回的对象，不同的 `key` 代表不同的锁。成功时 `elapsed` 代表获取锁需要耗费时间；否则为 `nil`，`err` 为错误字符串。

`elapsed` 代表的并不是墙上时间，而是所有等待 `step` 的累加和。如果 `elapsed` 大于零，说明锁之前被其他人持有，但是 `elapsed` 为零并不代表没有其他人获取过锁。

常见错误信息：

| 错误提示  | 含义                                                         |      |
| --------- | ------------------------------------------------------------ | ---- |
| "timeout" | 等待超过了创建锁时指定的 `timeout` 时间                      |      |
| "locked"  | 当前 `resty.lock` 对象实例已经持有一个锁（不一定是同一个 `key`） |      |
|           |                                                              |      |

**每个锁对象实例只能维护一个锁，如果需要操作锁需要多次调用 `new` 创建多个锁对象实例进行维护，不然会出现 `locked` 错误。**错误示例：

```lua
local resty_lock = require "resty.lock"

local dict = "my_locks"
local key1 = "key1"
local key2 = "key2"
local lock, err  = resty_lock:new(dict)
if not lock then
  ngx.log(ngx.ERR, "failed to create lock: ", err)
  return
end

local elapsed, err = lock:lock(key1)
if not elapsed then
  ngx.log(ngx.ERR, "failed to acquire the lock: ", err)
  return
end

-- 此处调用会出现 err: locked
local elapsed, err = lock:lock(key2)
if not elapsed then
  ngx.log(ngx.ERR, "failed to acquire the lock: ", err) 
  return
end
```

### 3. `unlock`

```lua
syntax: ok, err = obj:unlock()
```

释放锁，`obj` 为 `new` 返回值。成功返回 `1`，否则返回 `nil`，`err` 标明错误信息。

### 4. `expire`

```lua
syntax: ok, err = obj:expire(timeout)
```

设置当前 `resty.lock` 对象实例持有的锁的 `TTL`，这将把锁实例的超时时间设置为 `timeout` 参数指定的值（秒）。`expire` 设置的超时时间与调用 `new` 指定的超时时间是独立的（不会覆盖 `new` 指定的超时时间），调用 `expire(nil)` 会使用 `new` 函数指定的超时时间。

成功返回 `true`； 否则返回 `nil` ，错误信息由 `err` 描述。

## 三 使用示例

使用锁的常见场景是避免缓存雪崩：当缓存丢失时，限制对同一个 `key` 的并发后端查询。缓存锁的基本工作流程如下：

1. 使用 `key` 从缓存查询，缓存未命中，跳到第 2 步执行；
2. 实例化 `resty.lock` 对象，并使用 `lock` 函数给 `key` 加锁，成功则跳转到第 3 步；
3. 再次使用 `key` 从缓存查询。如果命中则释放锁并返回 `key` 的命中值；否则进行第 4 步；
4. 从后端进行查询，将查询结果放入缓存（如果未查询到应该存储一个未查询到特殊值），释放锁；

示例代码：

```lua
    local resty_lock = require "resty.lock"
    local cache = ngx.shared.my_cache

    -- step 1:
    local val, err = cache:get(key)
    if val then
        ngx.say("result: ", val)
        return
    end

    if err then
        return fail("failed to get key from shm: ", err)
    end

    -- cache miss!
    -- step 2:
    local lock, err = resty_lock:new("my_locks")
    if not lock then
        return fail("failed to create lock: ", err)
    end

    local elapsed, err = lock:lock(key)
    if not elapsed then
        return fail("failed to acquire the lock: ", err)
    end

    -- lock successfully acquired!

    -- step 3:
    -- someone might have already put the value into the cache
    -- so we check it here again:
    val, err = cache:get(key)
    if val then
        local ok, err = lock:unlock()
        if not ok then
            return fail("failed to unlock: ", err)
        end

        ngx.say("result: ", val)
        return
    end

    --- step 4:
    local val = fetch_redis(key)
    if not val then
        local ok, err = lock:unlock()
        if not ok then
            return fail("failed to unlock: ", err)
        end

        -- FIXME: we should handle the backend miss more carefully
        -- here, like inserting a stub value into the cache.

        ngx.say("no value found")
        return
    end

    -- update the shm cache with the newly fetched value
    local ok, err = cache:set(key, val, 1)
    if not ok then
        local ok, err = lock:unlock()
        if not ok then
            return fail("failed to unlock: ", err)
        end

        return fail("failed to update shm cache: ", err)
    end

    local ok, err = lock:unlock()
    if not ok then
        return fail("failed to unlock: ", err)
    end

    ngx.say("result: ", val)
```

## 四 使用限制

因为 `resty.lock` 使用 `ngx.sleep` 不断进行轮询，`ngx.sleep` 会 `yield` 当前轻量级线程，所以在所有禁止 `yield` 操作的阶段（`init/init_worker/header_filter/body_filter/balancer_by/log_by`）不能使用 `resty.lock` 库。

## 五 库实现

在 `lua-resty-lock` 库实现中锁存储在 `shared dict` 中，使用 `shared dict` 实现了跨 `worker` 间的锁。当 `key` 已经在 `shared dict` 中时会不断进行重试，尝试获得锁。

### 1. 创建锁实例

```lua

function _M.new(_, dict_name, opts)
    local dict = shared[dict_name]
    if not dict then
        return nil, "dictionary not found"
    end
  	-- 创建 cdata 类型数据
    -- 主要利用 cdata 的 _gc 元方法（在 ctype 定义中有设置）
    local cdata = ffi_new(ctype)
    cdata.key_id = 0
    -- 将 cdata 存储在 memo 表中，并返回其索引
    cdata.dict_id = ref_obj(dict)

    local timeout, exptime, step, ratio, max_step
    if opts then
        timeout = opts.timeout
        exptime = opts.exptime
        step = opts.step
        ratio = opts.ratio
        max_step = opts.max_step
    end

    if not exptime then
        exptime = 30
    end

    if timeout and timeout > exptime then
        timeout = exptime
    end

    local self = {
        cdata = cdata,
        dict = dict,
        timeout = timeout or 5,
        exptime = exptime,
        step = step or 0.001,
        ratio = ratio or 2,
        max_step = max_step or 0.5,
    }
    -- 设置元表
    return setmetatable(self, mt)
end
```

`ref_obj` 与 `unref_obj` 使用类似 `NGINX` 中 `connections` 数组的方法进行操作。`memo` 表用来进行存储，当表中数据删除时，将其索引保存到 `FREE_LIST_REF` 变量中，下次进行 `ref_obj` 操作时直接使用，当通过 `FREE_LIST_REF` 索引获得的索引为 `0` 时，计算 `memo` 表的大小，取下一个索引存储数据。

```lua
local FREE_LIST_REF = 0

-- FIXME: we don't need this when we have __gc metamethod support on Lua
--        tables.
local memo = {}
if debug then _M.memo = memo end


local function ref_obj(key)
    if key == nil then
        return -1
    end
    local ref = memo[FREE_LIST_REF]
    if ref and ref ~= 0 then
         memo[FREE_LIST_REF] = memo[ref]

    else
        ref = #memo + 1
    end
    memo[ref] = key

    -- print("ref key_id returned ", ref)
    return ref
end
if debug then _M.ref_obj = ref_obj end


local function unref_obj(ref)
    if ref >= 0 then
        memo[ref] = memo[FREE_LIST_REF]
        memo[FREE_LIST_REF] = ref
    end
end
```

### 2. 加锁操作

```lua
// 加锁操作
function _M.lock(self, key)
    if not key then
        return nil, "nil key"
    end

    local dict = self.dict
    local cdata = self.cdata
    -- cdata 已经存在锁
    if cdata.key_id > 0 then 
        return nil, "locked"
    end
  
  	-- 在 shared dict 添加 key
    local exptime = self.exptime
    local ok, err = dict:add(key, true, exptime)
    if ok then
    	  -- 加锁成功，增加 key 的引用
    		-- 将 key 存储在 memo table 中，避免被 GC 掉
        cdata.key_id = ref_obj(key)
        if not shdict_mt then
            shdict_mt = getmetatable(dict)
        end
        return 0
    end
  	-- 出错，直接返回
    if err ~= "exists" then
        return nil, err
    end
    -- 不断重试，尝试获得锁
    -- lock held by others
    local step = self.step
    local ratio = self.ratio
    local timeout = self.timeout
    local max_step = self.max_step
    local elapsed = 0
    while timeout > 0 do
        if step > timeout then
            step = timeout
        end

        sleep(step)
        elapsed = elapsed + step
        timeout = timeout - step

        local ok, err = dict:add(key, true, exptime)
        if ok then
            cdata.key_id = ref_obj(key)
            if not shdict_mt then
                shdict_mt = getmetatable(dict)
            end
            return elapsed
        end

        if err ~= "exists" then
            return nil, err
        end

        if timeout <= 0 then
            break
        end

        step = step * ratio
        if step <= 0 then
            step = 0.001
        end
        if step > max_step then
            step = max_step
        end
    end

    return nil, "timeout"
end
```

### 3. 去锁操作

```lua
function _M.unlock(self)
    local dict = self.dict
    local cdata = self.cdata
    local key_id = tonumber(cdata.key_id)
    if key_id <= 0 then
        return nil, "unlocked"
    end

  	-- 从 memo 表根据 key_id 获得 key 值
    local key = memo[key_id]
    unref_obj(key_id)

    -- 解锁操作：从 shared dict 删除 key
    local ok, err = dict:delete(key)
    if not ok then
        return nil, err
    end
    cdata.key_id = 0

    return 1
end
```

