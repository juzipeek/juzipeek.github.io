---
layout: post
title: lua c api - array and table operation
category: Lua
tags: Lua,C
keywords: Lua,C
description: 
---

## 一 概述

在 `C` 中使用 `API` 通过栈操作表或数组类型的数据，本篇使用示例来演示如果通过栈来操作表。

## 二 渠道秘钥更新

`set_channel_key` 函数接收一个 `table` 作为参数，从 `table` 中获取 `channel_name` 的值作为 `key`，之后从动态库的全局渠道秘钥表中获取相应渠道的秘钥，并在 `table` 中添加或更新 `channel_key` 条目。

### 1. 示例代码

```c
// gcc -fPIC -I/usr/local/lua5.1.5/include  -g -c private_cfg.c -Wall
// gcc -shared -I/usr/local/lua5.1.5/include  -L/usr/local/lua5.1.5/lib -llua -o private_cfg.so private_cfg.o

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

#include "lua.h"
#include "lauxlib.h"
#include "lualib.h"

typedef struct channel_key_s {
    char *channel;
    char *key;
} channel_key;


static channel_key 
g_channel_keys[] = {
    {"mobile", "mobile key"},
    {"pc", "pc key"},
    {NULL, NULL},
};

/*******************************************************************************
 * 从全局渠道秘钥表获取秘钥
 ******************************************************************************/
static char *
get_key(char *channel_name) {
    char *key = NULL;
    
    channel_key *p = g_channel_keys;
    for (;p->channel;p++) {
        if (!strcmp(p->channel, channel_name)) {
            key = p->key;
            break;
        }
    }

    return key;
}

/*******************************************************************************
 * do_set_channel_key 
 * 函数接收一个 table 作为参数，从 table 中获取 channel_name 的值作为 key，
 * 之后从全局渠道秘钥表中获取相应渠道的秘钥，并在 table 中添加 channel_key 条目。
 ******************************************************************************/
static int
do_set_channel_key(lua_State *L) {
    // 参数必须是 table
    if (!lua_istable(L, -1)) {
        return luaL_error(L, "type error, should use table as param!");
    }
    
    // 将 table 的 key `channel` 压入栈顶
    lua_pushstring(L, "channel_name");
    // 从 table 中获取 channel 字段，并其放到栈顶
    // 注意：此时栈顶是 channel 值，栈顶第二个元素是 table。
    //      原先栈顶的 "channel" 字符串已经被使用并移出栈。
    lua_gettable(L, -2);
    if (!lua_isstring(L, -1)) {
        return luaL_error(L, "channel type error, should use string as param!");
    }
    
    // 取出 channel 名
    char *channel_name = (char *)lua_tostring(L, -1);

    // 将栈顶的 channel 值出栈，此时栈顶元素是 table
    lua_pop(L, 1);
    
    char *key = get_key(channel_name);
    
    // 在表中添加记录
    lua_pushstring(L, "channel_key");
    lua_pushstring(L, key);
    lua_settable(L, -3);
    
    return 0;
}

static const 
struct luaL_reg private_cfg[] = {
    {"set_channel_key", do_set_channel_key},
    {NULL, NULL}
};

/******************************************************************************
* 注册函数
******************************************************************************/
int 
luaopen_private_cfg(lua_State *l) {
    luaL_openlib(l, "private_cfg", private_cfg, 0);
    return 1;
}
```

### 2. 执行

```lua
requrie "private_cfg"
tb = {channel_name = 'mobile', channel_key = 'no'}
private_cfg.set_channel_key(tb)
print(tb['channel_key']) --> mobile key
```

## 三 函数说明

### 1. 访问表

```c
void lua_gettable(lua_State *L, int index);
void lua_rawget(lua_State *L, int index);
void lua_rawgeti(lua_State *L, int index, int n);
```

`lua_gettable` 将 `t[k]` 的值压入栈顶，表是由 `index` 索引确定的栈上元素，`k` 使用栈顶元素。**函数会将栈顶的 `k` 元素 `pop` 出栈**。

`lua_rawget` 与 `lua_gettable` 函数功能相似，只是 `lua_rawget` 不会使用表的元方法（`metamethods`）。

`lua_rawgeti` 将 `t[n]` 值压入栈，`t` 由 `index` 索引指定，`n` 为字面值，不会使用元方法。

### 2. 更新表

```c
void lua_settable(lua_State *L, int index);
void lua_rawset(lua_State *L, int index);
void lua_rawseti(lua_State *L, int index, int n);
```

`lua_settable` 与 `lua` 中的 `t[k] = v` 语义相同，表 `t` 是由 `index` 索引确定的栈上元素，`v` 是当前栈顶元素，`k` 是当前次栈顶元素。**此函数会将栈顶、次栈顶元素 `pop` 出栈**。

`lua_rawset` 与 `lua_settable` 函数功能相似，只是 `lua_rawset` 不会使用表的元方法（`metamethods`）。

`lua_rawseti` 与 `lua` 中的 `t[n] = v` 语义相同，表 `t` 是由 `index` 索引确定的栈上元素，`v` 是当前栈顶元素，`n`  为字面值，该函数不会使用元方法。

### 3. 错误输出

```c
int luaL_error (lua_State *L, const char *fmt, ...);
```

触发错误，并输出 `fmt` 格式的错误信息，如果能获取文件名或行号会添加这些信息。

## 四 参考

- [programming in lua](https://www.lua.org/pil/25.1.html)
- [lua 5.1 c api 手册](https://www.lua.org/manual/5.1/manual.html)