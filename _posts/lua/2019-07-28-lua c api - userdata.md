---
layout: post
title: lua c api - userdata
category: Lua
tags: Lua,C
keywords: Lua,C
description: 
---

## 一 概述

在 `Lua` 中使用 `userdata` 表示 `C` 中的复杂数据类型（其实是一块内存区域），对于 `uerdata` 没有预定义的操作。`light userdata` 表示指针类型数据，并不需要创建（它是指针值），同时，**`light userdata` 不被 `gc` 管理**。`ligth userdata` 的主要用途是用户自己管理内存，避免 `gc` 管理内存。

## 二 使用 `userdata` 实现数组

在 `lua` 中使用 `table` 作为数组，在数组变大时会耗费非常多的内存，使用 `userdata` 能够降低内存使用。`userdata` 可以有 `metatable`，对 `userdata` 类型进行识别（判断 `C` 类型是否正确）增加元方法。

### 1. 示例代码

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

#include "lua.h"
#include "lauxlib.h"
#include "lualib.h"

typedef struct NumArray {
    int size;
    double values[1];  /* variable part */
} NumArray;

/******************************************************************************
* 创建 array
******************************************************************************/
static int 
newarray(lua_State *L) {
    int n = luaL_checkint(L, 1);
    size_t nbytes = sizeof(NumArray) + (n - 1)*sizeof(double);
    NumArray *a = (NumArray *)lua_newuserdata(L, nbytes);
    
    luaL_getmetatable(L, "LuaBook.array");
    lua_setmetatable(L, -2);
    
    a->size = n;
    return 1;  /* new userdatum is already on the stack */
}

static NumArray *
checkarray(lua_State *L) {
    void *ud = luaL_checkudata(L, 1, "LuaBook.array");
    luaL_argcheck(L, ud != NULL, 1, "`array' expected");
    return (NumArray *)ud;
}

/******************************************************************************
* 更新 array 中内容
* @param  userdata    array
* @param  index
* @para   value
******************************************************************************/
static int 
setarray(lua_State *L) {
    //NumArray *a = (NumArray *)lua_touserdata(L, 1);
    NumArray *a = checkarray(L);    
    luaL_argcheck(L, a != NULL, 1, "`array' expected");
    
    int index = luaL_checkint(L, 2);
    luaL_argcheck(L, 1 <= index && index <= a->size, 2,
                  "index out of range");
    
    double value = luaL_checknumber(L, 3);
    
    a->values[index-1] = value;
    return 0;
}

/******************************************************************************
* 获得 array 中内容
* @param  userdata    array
* @param  index
******************************************************************************/
static int 
getarray(lua_State *L) {
    //NumArray *a = (NumArray *)lua_touserdata(L, 1);
    NumArray *a = checkarray(L);
    luaL_argcheck(L, a != NULL, 1, "`array' expected");
    
    int index = luaL_checkint(L, 2);
    luaL_argcheck(L, 1 <= index && index <= a->size, 2,
                  "index out of range");
    
    lua_pushnumber(L, a->values[index-1]);
    return 1;
}

/******************************************************************************
* 获得 array 大小
* @param  userdata    array
******************************************************************************/
static int 
getsize(lua_State *L) {
    // NumArray *a = (NumArray *)lua_touserdata(L, 1);
    NumArray *a = checkarray(L);
    
    luaL_argcheck(L, a != NULL, 1, "`array' expected");
    lua_pushnumber(L, a->size);
    return 1;
}

static const 
struct luaL_reg array_lib[] = {
    {"new", newarray},
    {"set", setarray},
    {"get", getarray},
    {"size", getsize},
    {NULL, NULL}
};

/******************************************************************************
* 注册函数
******************************************************************************/
int 
luaopen_userdata(lua_State *l) {
    // 创建 metatable，使用 array.new 创建的数组元表是同一个元表
    luaL_newmetatable(l, "LuaBook.array");
    luaL_openlib(l, "array", array_lib, 0);
    return 1;
}
```

### 2. 调用示例

```bash
$ /usr/local/lua5.1.5/bin/lua -e "require 'userdata'; arr = array.new(20); print(type(arr)); array.set(arr,1,20);print('arr[1]=',array.get(arr,1));print('size:',array.size(arr))"
userdata
arr[1]=	20
size:	20
```

## 三 添加元方法

在创建元表时可以在元表中添加元方法，方便使用 `Lua` 的 `__index`、`__newindex` 方法。修改 `luaopen_userdata` 方法实现：

```c
int 
luaopen_userdata(lua_State *l) {
    // 创建 metatable，使用 array.new 创建的数组元表是同一个元表
    luaL_newmetatable(l, "LuaBook.array");
    luaL_openlib(l, "array", array_lib, 0);    
    
    /* now the stack has the metatable at index 1 and
       `array' at index 2 */
    lua_pushstring(l, "__index");
    lua_pushstring(l, "get");
    lua_gettable(l, -3);  /* get array.get on stack top*/
    lua_settable(l, -4);  /* metatable.__index = array.get */
    
    lua_pushstring(l, "__newindex");
    lua_pushstring(l, "set");
    lua_gettable(l, -3); /* get array.set */
    lua_settable(l, -4); /* metatable.__newindex = array.set */
    
    return 0;
}
```

**在这里的栈操作要仔细**

### 调用示例

```bash
$ /usr/local/lua5.1.5/bin/lua -e "require 'userdata'; arr = array.new(20); print(type(arr)); arr[1]=20;print('arr[1]=',arr[1])"
userdata
arr[1]=	20
```

## 四 函数说明

### 1. `lua_newuserdata`

```c
void *lua_newuserdata (lua_State *L, size_t size);
```

创建一块 `size` 大小的内存区域，将其压入栈顶并返回内存地址指针。

### 2. `luaL_checkudata`

```c
void *luaL_checkudata (lua_State *L, int narg, const char *tname);
```

检查函数的第 `narg` 参数是否是 `tname` 类型的 `userdata`。`luaL_checkudata` 首先将 `narg` 转换为 `userdata` ，然后获得元表；同时，从注册表中根据 `tname` 获得元表，两者相互比较如果不同触发错误并返回 `NULL`，如果相同则返回 `narg` 指向的 `userdata`。

*可以看 `luaL_checkudata` 代码实现，非常简单。*

### 3. `luaL_newmetatable`

```c
int luaL_newmetatable (lua_State *L, const char *tname);
```

创建一个新的可以作为 `userdata` 类型元表的 `table`，并使用 `tname` 作为 `key` 存储在注册表中。如果注册表中已经存在 `tname` 类型的值，返回值为 `0`。

### 4. `luaL_getmetatable`

```c
void luaL_getmetatable (lua_State *L, const char *tname);
```

将注册表中与名称 `tname` 关联的元表压入栈中。

### 5. `lua_setmetatable`

```c
int lua_setmetatable (lua_State *L, int index);
```

从栈顶弹出一个 `table` 并将其设置为栈中 `index` 索引出的值的元表。

## 五 参考资料

- [programming in lua](https://www.lua.org/pil/28.4.html)

- [lua 5.1 manual](https://www.lua.org/manual/5.1/manual.html)