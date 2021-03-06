---
layout: post
title: lua c api - registry
category: Lua
tags: Lua,C
keywords: Lua,C
description: 
---

## 一 `registry`

`registry` 是一个单独的表，用来保存 `lua` 中的全局变量，同时 `lua` 代码不能直接访问（只能通过 `c` 函数接口访问）。`registry` 通过伪索引（`LUA_REGISTRYINDEX`）访问，**伪索引类似于栈上的索引，但是它的关联数据并未在栈中**。

`registry` 的存在是用来解决一个问题：在 `c` 动态库中如何保存一个 `lua` 状态机的全局变量。如果使用 `c` 的全局变量、静态变量，相同进程内的所有 `lua` 状态机会共享此变量。如果将值保存在 `lua` 中的全局 `table` 中，`lua` 代码是可以直接访问、修改的；没有限制措施，数据被修改后非常容易导致 `c` 动态库崩溃。

使用 `registry` 可以实现针对每个 `state` 保存全局变量，同时不同动态库直接可以通过约定的 `key` 从 `registry` 中获得值，进行交互。

如果动态库（`.so`）不希望其他库修改自己设置的全局变量，一个好的方法是使用全局变量的地址作为 `key`。此时需要取全局变量地址，并转换为 `lightuserdata` 作为 `registry` 的 `key`。

### 1. 示例代码

```c
/*
 * 在 registry 全局表中保存一条记录 
 * registry[key] = value
 */
void 
state_store_global_var(lua_State *state, char *key, char *value)
{
    if (!state||!key||!value) return;

    lua_pushstring(state, key);     // 入栈
    lua_pushstring(state, value);   // 入栈
    // 操作 registy table
    // registy[key] = value
    lua_settable(state, LUA_REGISTRYINDEX);
}
```

## 二 `references`

如果想在 `registry` 中添加记录而又不想费劲考虑如何分配 `key`，可以使用 `reference` 系统。`reference` 系统可以自动生成一个唯一 `key`。

```c
int r = luaL_ref(L, LUA_REGISTRYINDEX);
```

这个函数会从栈顶弹出一个元素，并保存到 `registry` 中，同时返回唯一数值 `r` 作为索引。数值 `r` 就叫做栈顶值的“引用”（reference）。 

从这可以看出，用户不能在 `registry` 中使用数值作为 `key`，避免影响 `reference` 系统。

在 `c` 中无法直接使用指向 `lua` 的 `table`、`function` 类型指针（未提供此类接口），我们可以使用引用实现类似指针的功能。假设上面的例子中栈顶元素为 `table`，我们已经创建了它的一个引用，下面我们将其再次放入栈顶：

```c
lua_rawgeti(L, LUA_REGISTRYINDEX, r);
```

在引用使用结束后，我们需要使用 `luaL_unref` 释放引用值与引用自身：

```c
luaL_unref(L, LUA_REGISTRYINDEX, r);
```

## 三 `Upvalues`

`Upvalue` 存在的目的是为了在 `c` 中实现闭包功能。`Upvalue` 仅在函数内可见，并且在每个函数中是相互独立的。示例代码：

```c
// 定义函数
static int counter(lua_State *L);

int newCounter(lua_State *L) {
  lua_pushnumber(L, 0); // 栈顶增加数字，后续作为闭包的值
  // 创建闭包
  // &counter 是闭包基函数；注意是函数名取地址
  // 1 告诉闭包 upvalue 的数目
  lua_pushcclosure(L, &counter, 1);
  return 1;
}

static int counter(lua_State *L) {
  // lua_upvalueindex 同样适用伪索引技术，取 counter 函数的第一个 upvale
  double val = lua_tonumber(L, lua_upvalueindex(1));
  lua_pushnumber(L, ++val); // 栈顶值为 upvalue 值加 1
  lua_pushvalue(L, -1); // 栈顶增加一个相同值，用来更新 upvalue 值
  lua_replace(L, lua_upvalueindex(1)); // 更新 upvalue 值
  return 1;
}
```

### 1. 闭包示例代码

```c
// gcc -fPIC -I/usr/local/lua5.1.5/include  -g -c counter.c -Wall
// gcc -shared -I/usr/local/lua5.1.5/include  -L/usr/local/lua5.1.5/lib -llua -o counter.so counter.o

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#include "lua.h"
#include "lauxlib.h"
#include "lualib.h"

static int 
counter(lua_State *L) {
  // lua_upvalueindex 同样适用伪索引技术，取 counter 函数的第一个 upvale
  double val = lua_tonumber(L, lua_upvalueindex(1));
  lua_pushnumber(L, ++val); // 栈顶值为 upvalue 值加 1
  lua_pushvalue(L, -1); // 栈顶增加一个相同值，用来更新 upvalue 值
  lua_replace(L, lua_upvalueindex(1)); // 更新 upvalue 值
  return 1;
}

static int 
newCounter(lua_State *L) {
  lua_pushnumber(L, 0); // 栈顶增加数字，后续作为闭包的值
  // 创建闭包
  // &counter 是闭包基函数；注意是函数名取地址
  // 1 告诉闭包 upvalue 的数目
  lua_pushcclosure(L, &counter, 1);
  return 1;
}

static const 
struct luaL_reg counter_lib[] = {
    {"newCounter", newCounter},
    {NULL, NULL}
};

/******************************************************************************
 * 注册函数
 * 注意，在 lua 中 require 时，必须与 luaopen_xxxx 相同
 ******************************************************************************/
int 
luaopen_counter(lua_State *l) {
    luaL_openlib(l, "counter", counter_lib, 0);
    return 1;
}
```

`newCounter` 在 `C` 中创建一个闭包，返回给 `lua` 使用。

```lua
-- use package.loadlib or require to dynamic library
-- l,e = package.loadlib("./counter.so", "luaopen_counter")
require "counter"
c1 = counter.newCounter()
print(type(c1)) --> function
c2 = counter.newCounter()
print(c1()) --> 1
print(c1()) --> 2
print(c2()) --> 1
```

### 2. 程序说明

```c
#define luaI_openlib luaL_openLib
typedef struct luaL_Reg {
const char *name;
lua_CFunction func;
} luaL_Reg;
/* luaL_register 实际调用的也是 luaI_openlib 函数 */
LUALIB_API void 
(luaL_register) (lua_State *L, const char *libname, const luaL_Reg *l) {
luaI_openlib(L, libname, l, 0);
}
LUALIB_API void 
luaI_openlib (lua_State *L, const char *libname, const luaL_Reg *l, int nup) {
if (libname) {
int size = libsize(l);
/* check whether lib already exists */
luaL_findtable(L, LUA_REGISTRYINDEX, "_LOADED", 1);
lua_getfield(L, -1, libname);  /* get _LOADED[libname] */
if (!lua_istable(L, -1)) {  /* not found? */
lua_pop(L, 1);  /* remove previous result */
/* try global variable (and create one if it does not exist) */
if (luaL_findtable(L, LUA_GLOBALSINDEX, libname, size) != NULL)
luaL_error(L, "name conflict for module " LUA_QS, libname);
lua_pushvalue(L, -1);
lua_setfield(L, -3, libname);  /* _LOADED[libname] = new table */
}
lua_remove(L, -2);  /* remove _LOADED table */
lua_insert(L, -(nup+1));  /* move library table to below upvalues */
}
for (; l->name; l++) {
int i;
for (i=0; i<nup; i++)  /* copy upvalues to the top */
lua_pushvalue(L, -nup);
lua_pushcclosure(L, l->func, nup);
lua_setfield(L, -(nup+2), l->name);
}
lua_pop(L, nup);  /* remove upvalues */
}
```
`luaL_register\luaI_openlib\luaL_openlib` 三个函数都提供了注册函数列表到 `table` 的功能。`luaL_register` 是通过调用 `luaI_openlib` 函数来实现，提供了无闭包函数的功能。`luaI_openlib` 与 `luaL_openlib` 是同一个函数都会将函数注册到 `table` 中，`nup` 指定”上值“数量。
`luaI_openlib` 思路是操作栈，将需要注册的函数保存到栈底的 `table` 中，函数执行完毕后栈底保留原先 `table`。

## 四 参考链接

- [Storing State in C Function](https://www.lua.org/pil/27.3.html)
- [The Registry](https://www.lua.org/pil/27.3.1.html)
- [References](https://www.lua.org/pil/27.3.2.html)
- [Upvalues](https://www.lua.org/pil/27.3.3.html)