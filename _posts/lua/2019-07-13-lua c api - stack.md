---
layout: post
title: lua c api - stack
category: Lua
tags: Lua,C
keywords: Lua,C
description: 
---

## 一 概述

《lua c api》系列文章描述的是 `C` 语言作为宿主语言，与 `Lua` 程序通讯的 `API` 接口。

## 二 stack

`Lua` 使用虚拟的栈与宿主 `C` 语言进行通讯，在栈中的每个元素都是 `Lua` 识别的数据类型。**每当 `Lua` 调用 `C` 时，被调用的 `C` 函数都会得到一个新的栈，该栈独立于以前的栈和仍处于活跃状态的 `C` 函数栈**。在此堆栈包含 `lua` 调用 `C` 函数的所有参数，`C` 函数应该将需要返回的结果压入此堆栈中（**是否需要清空栈**）。

为了方便，在 `C` 中操作栈时并未严格遵守栈操作语义，可以使用“索引”来查询栈中的元素。正数索引从栈底开始计数，栈底索引为 `1`；负数索引从栈顶开始计数，栈顶索引为 `-1`。示例结构：

```
---------------------------------------
| 正数索引 |  数据   | 负数索引 | 栈中位置 |
---------------------------------------
|    3    |   nil  |   -1    |  栈顶   |
|    2    |   2    |   -2    |  中间   |
|    1    |  "hi"  |   -3    |  栈底   |
```

注意，使用 `0` 作为索引会出错。

## 三 示例

*使用 `lua5.1.5` 版本*

### 1. 程序

在示例中 `echo` 函数会返回所有调用参数，在返回时栈底元素作为第一个返回值，栈顶元素作为最后一个返回值；`sayhi` 会清空栈顶并返回一个 "hi" 字符串。

```c
// gcc -fPIC -I/usr/local/lua5.1.5/include -g -c stack.c
// gcc -shared -I/usr/local/lua5.1.5/include  -L/usr/local/lua5.1.5/lib -llua -o stack.so stack.o
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#include "lua.h"
#include "lauxlib.h"
#include "lualib.h"

/******************************************************************************
 * sayhi 函数
 * 清空栈，并将 hi 字符串压入栈顶
 ******************************************************************************/
static int 
sayhi(lua_State *L) {
    lua_settop(L, 0);
    lua_pushstring(L, "hi");
    return 1;
}

/******************************************************************************
 * echo 函数会返回所有输入参数
 ******************************************************************************/
static int
echo(lua_State *L) {
    int num = lua_gettop(L);
    return num;
}

static const 
struct luaL_reg stack_lib[] = {
    {"sayhi", sayhi},
    {"echo", echo},
    {NULL, NULL}
};

/******************************************************************************
 * 注册函数
 * 注意 luaopen_xxx 中 xxx 为 lua 中 require 库名称
 ******************************************************************************/
int 
luaopen_stack(lua_State *l) {
    luaL_openlib(l, "stack", stack_lib, 0);
    return 1;
}
```

*编译命令在代码开始处*

### 2. 调用

在解释器中可以调用测试：

```bash
Lua 5.1.5  Copyright (C) 1994-2012 Lua.org, PUC-Rio
> require "stack"
> r1,r2 = stack.echo("a","b")
> print(r1,r2)
a	b
> r = stack.sayhi()
> print(r)
hi
```

## 四 函数说明

### 1. 栈顶索引操作

```c
void lua_settop (lua_State *L, int index);
```

函数功能：
将 `L` 的栈顶设置到 `index` 指定的索引处。
如果 `index` 比现有的栈顶高，会扩充栈，扩充的栈中值为 `nil`。
如果 `index` 比现有的栈顶低，那么 `index` 到历史栈顶之间的元素会被删除。
如果 `index` 为 `0`，那么栈中所有元素都将被删掉。

### 2. 字符串压入栈

```c
void lua_pushstring (lua_State *L, const char *s);
```

将 `s` 指向的字符串压入 `L` 的栈顶。字符串会以 "\0" 结束。
需要注意：**`lua_pushstring` 会做值拷贝，不需要担心调用栈退出后内存回收导致 `coredump` 问题**。

### 3. 获得栈顶索引

```c
int lua_gettop (lua_State *L);
```

获得栈顶元素的索引。因为栈底索引是 `1` 所以栈顶元素索引等价于调用栈中所有元素数量。如果返回值为 `0` 说明栈为空。

## 五 参考

- [lua 5.1 手册](https://www.lua.org/manual/5.1/manual.html#lua_checkstack)