---
layout: post
title: lua c api - calling lua function
category: Lua
tags: Lua,C
keywords: Lua,C
description: 
---

## 一 概述

很多情况下有这种需求：在特定的框架中针对不同的业务做少量修改。使用 `C/C++` 开发稳定的框架，调用针对不同业务开发的 `Lua` 函数可以实现需求。其实从 `C/C++` 调用 `Lua` 函数非常简单，**调用时将 `Lua` 函数压入栈、将函数参数压入栈，调用 `lua_pcall` 完成调用；调用后从栈获得调用函数返回结果**。

## 二 示例

### 1. 程序示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

#include "lua.h"
#include "lauxlib.h"
#include "lualib.h"

/******************************************************************************
 * 函数结构体定义
 ******************************************************************************/
typedef struct function_buf_s {
    char *name;  // 函数名
    int nargs;   // 参数数量
    int nret;    // 返回值数量
    char *chunk; // 函数体
} function_buf;

// 此处 function 不应该使用 function f() ... end 方式编写，写出的函数其实不会被执行
static function_buf 
lua_function[] = {
    {"sayhi", 1, 1,
     "local name = ...\n"
     "return string.format('hi %s!', name)\n"
    },
    {"echo", 1, 1,
     "function echo(msg)\n"
     "    return msg\n"
     "end\n"
     "local msg = ...\n"
     "return echo(msg)\n"
    },
    {NULL, 0, 0,NULL}
};

static int 
sayHi(lua_State *L) {

    function_buf *function = &lua_function[0];
    
    int ret = luaL_loadbuffer(L, function->chunk, strlen(function->chunk), 
                              function->name);
    
    if (ret) {
        return luaL_error(L, "call loadbuffer fail ret:%d, code:%s", 
                          ret, function->chunk);
    }
    
    lua_pushstring(L, "lua");
    ret = lua_pcall(L, 1, 1, 0);
    if (ret != 0) {
        return luaL_error(L, "lua_pcall fail ret:%d %s", 
                          ret, lua_tostring(L, -1));
    }

    // 输出类型
    printf("typename %s\n", luaL_typename(L, -1));

    if (!lua_isstring(L, -1)) {
        return luaL_error(L, "lua_pcall ret not string");
    }
    
    // 函数执行结果在栈中，取出来再放入栈
    const char *msg = lua_tostring(L, -1);
    lua_pop(L, -1);
    lua_pushfstring(L, "sayhi say:%s", msg);
    return 1;
}

/******************************************************************************
 * 通用入口函数
 * 调用时首先将函数名压入栈，之后是其他参数
 ******************************************************************************/
static int
common_entry(lua_State *L) {
    if (!lua_isstring(L, 1)) {
        return luaL_error(L, "first arg should be function name, string type");
    }
    const char * function_name = lua_tostring(L, 1);
    
    function_buf *function = lua_function;
    
    for (;function->name; function++) {
        if (!strcmp(function->name, function_name)) break;
    }
    if (!function->name) {
        return luaL_error(L, "not found:%s function in array", function_name);
    }

    int ret = luaL_loadbuffer(L, function->chunk, strlen(function->chunk), 
                              function->name);
    
    if (ret) {
        return luaL_error(L, "call loadbuffer fail ret:%d, code:%s", 
                          ret, function->chunk);
    }
    
    int i;
    for(i=2; i <= function->nargs+1; i++) {
        lua_pushstring(L, lua_tostring(L, i));
    }

    ret = lua_pcall(L, function->nargs, function->nret, 0);
    if (ret != 0) {
        return luaL_error(L, "lua_pcall fail ret:%d %s", 
                          ret, lua_tostring(L, -1));
    }
    
    int nret = function->nret;
    
    for (i=-1; i >= -nret; i--){
        printf("ret:%d is:%s\n", -i, lua_tostring(L, i));
    }
    
    return nret;
}

static const 
struct luaL_reg call_function_lib[] = {
    {"sayHi", sayHi},
    {"common_entry", common_entry},
    {NULL, NULL}
};

/******************************************************************************
* 注册函数
******************************************************************************/
int 
luaopen_call_function(lua_State *l) {
    luaL_openlib(l, "call_function", call_function_lib, 0);
    return 1;
}
```

程序中有两个示例，`sayHi` 函数接收一个 `string` 参数并返回一个 `string` 参数；`common_entry` 函数可以根据函数名选择相应的函数体进行执行，更加灵活。

**在函数定义中需要注意，`luaL_loadbuffer` 系列函数会将字符串当做程序段载入并放在栈顶，但是并未执行函数。如果载入的代码段是 `function f()…end` 格式，使用 `lua_pcall` 调用代码段时函数 `f` 仅做了定义，未执行函数体。**

### 2. 调用示例

```bash
$ /usr/local/lua5.1.5/bin/lua -e "require 'call_function'; ret = call_function.sayHi('lua'); print(ret)"
typename string
sayhi say:hi lua!

$ /usr/local/lua5.1.5/bin/lua -e "require 'call_function'; ret = call_function.common_entry('echo','lua is cool'); print(ret)"
ret:1 is:lua is cool
lua is cool
```

## 三 函数说明

### 1. `luaL_error`

```c
int luaL_error (lua_State *L, const char *fmt, ...);
```

触发错误，并使用 `fmt` 与后续参数格式化输出错误消息字符串。`luaL_error` 函数并不会退出，通常的是否用法为：

```c
return luaL_error(args);
```

## 四 参考资料

-[programming in lua](https://www.lua.org/pil/25.2.html)

-[lua 5.1 手册](https://www.lua.org/manual/5.1/manual.html#luaL_loadbuffer)