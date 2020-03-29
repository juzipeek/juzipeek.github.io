---
layout: post
title: OpenResty Api - socket tcp
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description:
---

## 一 概述

`ngx.thread` 提供了以 Lua 协程为基础构建的轻量级线程，在 OpenResty 中有两类轻量级线程：普通轻量级线程和主轻量级线程（entry thread）。主轻量级线程是由 `access_by_lua`,`content_by_lua`,`rewrite_by_lua` 指令创建的轻量级线程，普通轻量级线程是在 entry thread 内动态创建的，两者皆由 Openresty 调度。与轻量级线程不同，Lua 原生语义创建的协程并非轻量级线程，两者的区别在于：轻量级线程在遇到 IO 调用时会自动 yield 让出，而原生协程需要协程自己 yield。不过两者（原生协程、轻量级线程）都会被 Openresty 协程调度框架调度。

## 指令

### 1. `spawn`

```
syntax: co = ngx.thread.spawn(func, arg1, arg2, ...)
context: rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*
```

使用 func 以及可选的参数 `arg1`,`arg2`... 创建协程作为轻量级线程返回，`ngx.thread.spawn` 创建的线程由 `ngx_lua` 模块调度。新创建的协程会先运行（在 `spawn` 调用前已经运行，由 `ngx_lua` 调度实现），直到出错或进行 IO 操作而让出执行。`spawn` 返回后，新建的轻量级协程会在其“关注”的 IO 事件触发后异步执行（由 `ngx_lua` 调度实现）。

其实，由 `rewrite_by_lua`,`access_by_lua`,`content_by_lua` 运行的 Lua 代码块会由 `ngx_lua` 自动创建轻量级线程，这些轻量级线程被称为入口线程。这些入口线程会在以下三种情况下终止：
- 1. 入口线程和所有的用户轻量级线程已经终止（意味着请求正常处理完毕）；
- 2. 入口线程或用户轻量级线程调用 `ngx.exit`, `ngx.exec`, `ngx.redirect`, `ngx.req.set_uri(uri, true)`;
- 3. 入口线程因为 Lua 异常而终止（用户轻量级线程出错不会导致其他轻量级线程终止）；

在 Nginx 实现中子请求不能被终止，如果一个线程正在执行子请求，那么该线程不允许被终止。对于具有子请求的线程，应该使用 `ngx.thread.wait` 等待其终止，异常情况下可以使用 ngx.ERROR, 408, 444, 499 调用 ngx.exit 来终止子请求。

轻量级线程不会以抢占方式进行调度，线程会一直运行直至以下三种情况才会让出执行：
- 1. 非阻塞 IO 无法在一次运行中完成；
- 2. 线程调用 coroutine.yield 主动让出；
- 3. 触发 Lua 异常而退出或者调用 ngx.exit, ngx.exec, ngx.redirect, ngx.req.set_uri(uri, true) 而退出；

线程有父子关系，只有父线程能调用 `ngx.thread.wait`, `ngx.thread.kill` 去等待或杀掉子线程。轻量级线程有“僵尸”状态：当前线程已经结束（正常或错误退出），它的父线程还存活但是父线程未使用 `ngx.thread.wait` 等待线程结束。

### 2. `wait`

```
syntax: ok, res1  = ngx.thread.wait(thread1, thread2, ...)
context: rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*
```

等待一个或多个之前通过 `ngx.thread.spawn` 创建的线程结束，当其中一个线程结束或退出时函数返回。返回值与 `coroutine.resume` 返回值相同。`ok` 表示线程是否成功终止，而后续返回值是用户函数的返回值或其执行错误。

### 3. `kill`

```
syntax: ok, err = ngx.thread.kill(thread)
context: rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*
```

关闭之前由 `spawn` 创建的线程，当成功时 `ok` 为 `true`，失败时返回 `nil` 以及错误描述。

## 实现

### 1. spawn

```c
// 创建新的请轻量级线程
// 在此次并未执行新创建的协程，本函数结束后，会处罚协程调度，在协程调度中执行新创建协程
static int
ngx_http_lua_uthread_spawn(lua_State *L)
{
    // ...
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    // 创建协程
    ngx_http_lua_coroutine_create_helper(L, r, ctx, &coctx);

    /* anchor the newly created coroutine into the Lua registry */

    lua_pushlightuserdata(L, &ngx_http_lua_coroutines_key);
    // 从 LUA_REGISTRYINDEX 伪索引中获得 ngx_http_lua_coroutines_key 指向的表，并将其压入栈顶
    lua_rawget(L, LUA_REGISTRYINDEX);
    // 此时 -2 索引指向新创建的协程 coctx->co，创建其拷贝并压入栈顶
    lua_pushvalue(L, -2);
    // 将新创建的协程从栈顶移除，并在 ngx_http_lua_coroutines_key 指定表中存储
    coctx->co_ref = luaL_ref(L, -2);
    lua_pop(L, 1); // 将新创建协程出栈

    // 将新创建的协程运行参数从其父协程移动到新创建的协程中，会保持原先顺序
    if (n > 1) {
        lua_replace(L, 1);
        lua_xmove(L, coctx->co, n - 1);
    }

    coctx->is_uthread = 1;
    ctx->uthreads++;

    // 设置新建协程状态
    coctx->co_status = NGX_HTTP_LUA_CO_RUNNING;
    // 设置当前协程状态
    ctx->co_op = NGX_HTTP_LUA_USER_THREAD_RESUME;

    ctx->cur_co_ctx->thread_spawn_yielded = 1;

    // 将新创建的协程存储在请求 ctx 中的 posted_threads
    if (ngx_http_lua_post_thread(r, ctx, ctx->cur_co_ctx) != NGX_OK) {
        return luaL_error(L, "no memory");
    }

    coctx->parent_co_ctx = ctx->cur_co_ctx;
    ctx->cur_co_ctx = coctx;

    // dtrace 使用，忽略
    ngx_http_lua_probe_user_thread_spawn(r, L, coctx->co);

    dd("yielding with arg %s, top=%d, index-1:%s", luaL_typename(L, -1),
       (int) lua_gettop(L), luaL_typename(L, 1));
    // 调用协程让出，以便新创建的协程运行，返回一个协程结果
    return lua_yield(L, 1);
}

// 创建协程
int
ngx_http_lua_coroutine_create_helper(lua_State *L, ngx_http_request_t *r,
    ngx_http_lua_ctx_t *ctx, ngx_http_lua_co_ctx_t **pcoctx)
{
    // ... omit

    vm = ngx_http_lua_get_lua_vm(r, ctx);

    /* create new coroutine on root Lua state, so it always yields
     * to main Lua thread
     */
    co = lua_newthread(vm);

    // dtrace 检测使用，忽略
    ngx_http_lua_probe_user_coroutine_create(r, L, co);

    // 创建 coctx
    coctx = ngx_http_lua_get_co_ctx(co, ctx);
    if (coctx == NULL) {
        coctx = ngx_http_lua_create_co_ctx(r, ctx);
        if (coctx == NULL) {
            return luaL_error(L, "no memory");
        }
    } else {
        ngx_memzero(coctx, sizeof(ngx_http_lua_co_ctx_t));
        coctx->co_ref = LUA_NOREF;
    }

    coctx->co = co;
    coctx->co_status = NGX_HTTP_LUA_CO_SUSPENDED;

    /* make new coroutine share globals of the parent coroutine.
     * NOTE: globals don't have to be separated! */
    // 将协程环境的全局变量压入栈顶
    // 协程环境的全局变量都在 LUA_GLOBALSINDEX 伪索引指向的表中
    // 运行的 C 函数的全局变量都在 LUA_ENVIRONINDEX 伪索引指向的表中
    ngx_http_lua_get_globals_table(L);
    // 将 L 协程栈中 1 索引处值出栈，压入 co 协程的栈顶
    lua_xmove(L, co, 1);
    // 将 co 协程栈顶的全局环境表移动到 LUA_GLOBALSINDEX 处，实现继承父协程的全局环境
    ngx_http_lua_set_globals_table(co);

    // 将主协程 vm 栈底的元素 co 协程移动到 L 协程栈顶
    lua_xmove(vm, L, 1);    /* move coroutine from main thread to L */

    // 将栈底的函数拷贝并压入栈顶
    lua_pushvalue(L, 1);    /* copy entry function to top of L*/
    // 从 L 协程栈底移动到 co 协程栈顶
    lua_xmove(L, co, 1);    /* move entry function from L to co */

    if (pcoctx) {
        *pcoctx = coctx;
    }

#ifdef NGX_LUA_USE_ASSERT
    coctx->co_top = 1;
#endif

    return 1;    /* return new coroutine to Lua */
}
```

### 2. wait

```c
// syntax: ok, res1, res2, ... = ngx.thread.wait(thread1, thread2, ...)
static int
ngx_http_lua_uthread_wait(lua_State *L)
{
    // ... 忽略代码
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no request ctx found");
    }

    // 允许介入阶段
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER
                               | NGX_HTTP_LUA_CONTEXT_SSL_CERT);

    coctx = ctx->cur_co_ctx;

    nargs = lua_gettop(L);

    for (i = 1; i <= nargs; i++) {
        sub_co = lua_tothread(L, i);

        luaL_argcheck(L, sub_co, i, "lua thread expected");

        sub_coctx = ngx_http_lua_get_co_ctx(sub_co, ctx);
        if (sub_coctx == NULL) {
            return luaL_error(L, "no co ctx found");
        }

        if (!sub_coctx->is_uthread) {
            return luaL_error(L, "attempt to wait on a coroutine that is "
                              "not a user thread");
        }

        if (sub_coctx->parent_co_ctx != coctx) {
            return luaL_error(L, "only the parent coroutine can wait on the "
                              "thread");
        }

        switch (sub_coctx->co_status) {
        case NGX_HTTP_LUA_CO_ZOMBIE:

            ngx_http_lua_probe_info("found zombie child");

            nrets = lua_gettop(sub_coctx->co);

            dd("child retval count: %d, %s: %s", (int) nrets,
               luaL_typename(sub_coctx->co, -1),
               lua_tostring(sub_coctx->co, -1));

            if (nrets) {
                lua_xmove(sub_coctx->co, L, nrets);
            }

#if 1
            ngx_http_lua_del_thread(r, L, ctx, sub_coctx);
            ctx->uthreads--;
#endif

            return nrets;

        case NGX_HTTP_LUA_CO_DEAD:
            dd("uthread already waited: %p (parent %p)", sub_coctx,
               coctx);

            if (i < nargs) {
                /* just ignore it if it is not the last one */
                continue;
            }

            /* being the last one */
            lua_pushnil(L);
            lua_pushliteral(L, "already waited or killed");
            return 2;

        default:
            dd("uthread %p still alive, status: %d, parent %p", sub_coctx,
               sub_coctx->co_status, coctx);
            break;
        }

        ngx_http_lua_probe_user_thread_wait(L, sub_coctx->co);
        sub_coctx->waited_by_parent = 1;
    }
    // 调用协程让出，返回 0 个参数
    return lua_yield(L, 0);
}
```

### 3. kill

```c
// syntax: ok, err = ngx.thread.kill(thread)
static int
ngx_http_lua_uthread_kill(lua_State *L)
{
    // ... 忽略参数判断
    // 阶段判断
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER
                               | NGX_HTTP_LUA_CONTEXT_SSL_CERT);

    coctx = ctx->cur_co_ctx;

    // 栈底元素转换为协程对象
    sub_co = lua_tothread(L, 1);
    luaL_argcheck(L, sub_co, 1, "lua thread expected");

    sub_coctx = ngx_http_lua_get_co_ctx(sub_co, ctx);

    if (sub_coctx == NULL) {
        return luaL_error(L, "no co ctx found");
    }

    if (!sub_coctx->is_uthread) {
        lua_pushnil(L);
        lua_pushliteral(L, "not user thread");
        return 2;
    }

    if (sub_coctx->parent_co_ctx != coctx) {
        lua_pushnil(L);
        lua_pushliteral(L, "killer not parent");
        return 2;
    }

    if (sub_coctx->pending_subreqs > 0) {
        lua_pushnil(L);
        lua_pushliteral(L, "pending subrequests");
        return 2;
    }

    switch (sub_coctx->co_status) {
    case NGX_HTTP_LUA_CO_ZOMBIE:
        // 从索引表中删除协程
        ngx_http_lua_del_thread(r, L, ctx, sub_coctx);
        ctx->uthreads--;

        lua_pushnil(L);
        lua_pushliteral(L, "already terminated");
        return 2;

    case NGX_HTTP_LUA_CO_DEAD:
        lua_pushnil(L);
        lua_pushliteral(L, "already waited or killed");
        return 2;

    default:
        // 调用 clenup 回调函数进行清理
        ngx_http_lua_cleanup_pending_operation(sub_coctx);
        ngx_http_lua_del_thread(r, L, ctx, sub_coctx);
        ctx->uthreads--;

        lua_pushinteger(L, 1);
        return 1;
    }

    /* not reacheable */
}
```

## 协程调度

以 `access_by_lua` 指令为例，查看 thread 调度。

### 1. access_by_lua

```c
// entry thread
static ngx_int_t
ngx_http_lua_access_by_chunk(lua_State *L, ngx_http_request_t *r)
{
    int                  co_ref;
    ngx_int_t            rc;
    lua_State           *co;
    ngx_event_t         *rev;
    ngx_connection_t    *c;
    ngx_http_lua_ctx_t  *ctx;
    ngx_http_cleanup_t  *cln;

    ngx_http_lua_loc_conf_t     *llcf;

    /*  {{{ new coroutine to handle request */
    co = ngx_http_lua_new_thread(r, L, &co_ref);

    if (co == NULL) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "lua: failed to create new coroutine "
                      "to handle request");

        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    /*  move code closure to new coroutine */
    lua_xmove(L, co, 1);

    /*  set closure's env table to new coroutine's globals table */
    ngx_http_lua_get_globals_table(co);
    lua_setfenv(co, -2);

    /*  save nginx request in coroutine globals table */
    ngx_http_lua_set_req(co, r);

    /*  {{{ initialize request context */
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    dd("ctx = %p", ctx);

    if (ctx == NULL) {
        return NGX_ERROR;
    }

    ngx_http_lua_reset_ctx(r, L, ctx);

    ctx->entered_access_phase = 1;

    ctx->cur_co_ctx = &ctx->entry_co_ctx;
    ctx->cur_co_ctx->co = co;
    ctx->cur_co_ctx->co_ref = co_ref;
#ifdef NGX_LUA_USE_ASSERT
    ctx->cur_co_ctx->co_top = 1;
#endif

    /*  }}} */

    /*  {{{ register request cleanup hooks */
    if (ctx->cleanup == NULL) {
        cln = ngx_http_cleanup_add(r, 0);
        if (cln == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        cln->handler = ngx_http_lua_request_cleanup_handler;
        cln->data = ctx;
        ctx->cleanup = &cln->handler;
    }
    /*  }}} */

    ctx->context = NGX_HTTP_LUA_CONTEXT_ACCESS;

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    if (llcf->check_client_abort) {
        r->read_event_handler = ngx_http_lua_rd_check_broken_connection;

#if (NGX_HTTP_V2)
        if (!r->stream) {
#endif

        rev = r->connection->read;

        if (!rev->active) {
            if (ngx_add_event(rev, NGX_READ_EVENT, 0) != NGX_OK) {
                return NGX_ERROR;
            }
        }

#if (NGX_HTTP_V2)
        }
#endif

    } else {
        r->read_event_handler = ngx_http_block_reading;
    }

    // 进入线程调度 thread scheduler
    rc = ngx_http_lua_run_thread(L, r, ctx, 0);

    dd("returned %d", (int) rc);

    if (rc == NGX_ERROR || rc > NGX_OK) {
        return rc;
    }

    c = r->connection;

    if (rc == NGX_AGAIN) {
        rc = ngx_http_lua_run_posted_threads(c, L, r, ctx);

        if (rc == NGX_ERROR || rc == NGX_DONE || rc > NGX_OK) {
            return rc;
        }

        if (rc != NGX_OK) {
            return NGX_DECLINED;
        }

    } else if (rc == NGX_DONE) {
        ngx_http_lua_finalize_request(r, NGX_DONE);

        rc = ngx_http_lua_run_posted_threads(c, L, r, ctx);

        if (rc == NGX_ERROR || rc == NGX_DONE || rc > NGX_OK) {
            return rc;
        }

        if (rc != NGX_OK) {
            return NGX_DECLINED;
        }
    }

#if 1
    if (rc == NGX_OK) {
        if (r->header_sent) {
            dd("header already sent");

            /* response header was already generated in access_by_lua*,
             * so it is no longer safe to proceed to later phases
             * which may generate responses again */

            if (!ctx->eof) {
                dd("eof not yet sent");

                rc = ngx_http_lua_send_chain_link(r, ctx, NULL
                                                  /* indicate last_buf */);
                if (rc == NGX_ERROR || rc > NGX_OK) {
                    return rc;
                }
            }

            return NGX_HTTP_OK;
        }

        return NGX_OK;
    }
#endif

    return NGX_DECLINED;
}
```

### 2. ngx_http_lua_run_thread

```c
// 调度线程
/*
 * description:
 *  run a Lua coroutine specified by ctx->cur_co_ctx->co
 * return value:
 *  NGX_AGAIN:      I/O interruption: r->main->count intact
 *  NGX_DONE:       I/O interruption: r->main->count already incremented by 1
 *  NGX_ERROR:      error
 *  >= 200          HTTP status code
 */
ngx_int_t
ngx_http_lua_run_thread(lua_State *L, ngx_http_request_t *r,
    ngx_http_lua_ctx_t *ctx, volatile int nrets)
{
    ngx_http_lua_co_ctx_t   *next_coctx, *parent_coctx, *orig_coctx;
    int                      rv, success = 1;
    lua_State               *next_co;
    lua_State               *old_co;
    const char              *err, *msg, *trace;
    ngx_int_t                rc;
#if (NGX_PCRE)
    ngx_pool_t              *old_pool = NULL;
#endif

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua run thread, top:%d c:%ud", lua_gettop(L),
                   r->main->count);

    /* set Lua VM panic handler */
    lua_atpanic(L, ngx_http_lua_atpanic);

    dd("ctx = %p", ctx);

    NGX_LUA_EXCEPTION_TRY {
        // 协程刚刚创建？
        if (ctx->cur_co_ctx->thread_spawn_yielded) {
            ngx_http_lua_probe_info("thread spawn yielded");

            ctx->cur_co_ctx->thread_spawn_yielded = 0;
            nrets = 1;
        }

        for ( ;; ) {

            dd("calling lua_resume: vm %p, nret %d", ctx->cur_co_ctx->co,
               (int) nrets);

#if (NGX_PCRE)
            /* XXX: work-around to nginx regex subsystem */
            old_pool = ngx_http_lua_pcre_malloc_init(r->pool);
#endif

            /*  run code */
            dd("ctx: %p", ctx);
            dd("cur co: %p", ctx->cur_co_ctx->co);
            dd("cur co status: %d", ctx->cur_co_ctx->co_status);

            // 当前待执行线程
            orig_coctx = ctx->cur_co_ctx;

#ifdef NGX_LUA_USE_ASSERT
            dd("%p: saved co top: %d, nrets: %d, true top: %d",
               orig_coctx->co,
               (int) orig_coctx->co_top, (int) nrets,
               (int) lua_gettop(orig_coctx->co));
#endif

#if DDEBUG
            if (lua_gettop(orig_coctx->co) > 0) {
                dd("top elem: %s", luaL_typename(orig_coctx->co, -1));
            }
#endif

            ngx_http_lua_assert(orig_coctx->co_top + nrets
                                == lua_gettop(orig_coctx->co));

            // 执行线程
            rv = lua_resume(orig_coctx->co, nrets);

#if (NGX_PCRE)
            /* XXX: work-around to nginx regex subsystem */
            ngx_http_lua_pcre_malloc_done(old_pool);
#endif

#if 0
            /* test the longjmp thing */
            if (rand() % 2 == 0) {
                NGX_LUA_EXCEPTION_THROW(1);
            }
#endif

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "lua resume returned %d", rv);

            switch (rv) {
            // rv == YIELD 线程主动让出
            case LUA_YIELD:
                /*  yielded, let event handler do the rest job */
                /*  FIXME: add io cmd dispatcher here */

                ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                               "lua thread yielded");

#ifdef NGX_LUA_USE_ASSERT
                dd("%p: saving curr top after yield: %d (co-op: %d)",
                   orig_coctx->co,
                   (int) lua_gettop(orig_coctx->co), (int) ctx->co_op);
                orig_coctx->co_top = lua_gettop(orig_coctx->co);
#endif
                // uri rewrite，执行 server_rewrite
                if (r->uri_changed) {
                    return ngx_http_lua_handle_rewrite_jump(L, r, ctx);
                }
                // 调用了 ngx.exit，请求处理结束处理
                if (ctx->exited) {
                    return ngx_http_lua_handle_exit(L, r, ctx);
                }

                // 调用了 ngx.exec，执行 server_rewrite
                if (ctx->exec_uri.len) {
                    return ngx_http_lua_handle_exec(L, r, ctx);
                }

                /*
                 * check if coroutine.resume or coroutine.yield called
                 * lua_yield()
                 */
                switch(ctx->co_op) {
                    // 等待 io 事件触发
                case NGX_HTTP_LUA_USER_CORO_NOP:
                    dd("hit! it is the API yield");

                    ngx_http_lua_assert(lua_gettop(ctx->cur_co_ctx->co) == 0);

                    ctx->cur_co_ctx = NULL;

                    return NGX_AGAIN;

                    // 创建新线程，break 后再次执行循环，执行新创建的线程
                case NGX_HTTP_LUA_USER_THREAD_RESUME:

                    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                                   "lua user thread resume");

                    ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP;
                    nrets = lua_gettop(ctx->cur_co_ctx->co) - 1;
                    dd("nrets = %d", nrets);

#ifdef NGX_LUA_USE_ASSERT
                    /* ignore the return value (the thread) already pushed */
                    orig_coctx->co_top--;
#endif

                    break;

                // 用户线程调用 resume 运行其他线程
                // 此时 ctx->cur_co_ctx 是待执行的线程，
                // 在整个模块中只有 ngx_http_lua_coroutine_resume 函数设置 co_op 为 RESUME 状态
                case NGX_HTTP_LUA_USER_CORO_RESUME:
                    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                                   "lua coroutine: resume");

                    /*
                     * the target coroutine lies at the base of the
                     * parent's stack
                     */
                    ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP;

                    old_co = ctx->cur_co_ctx->parent_co_ctx->co;

                    nrets = lua_gettop(old_co);
                    if (nrets) {
                        dd("moving %d return values to parent", nrets);
                        // 将父线程调用 resume 的 nrets 个参数从父线程移动到子线程
                        lua_xmove(old_co, ctx->cur_co_ctx->co, nrets);

#ifdef NGX_LUA_USE_ASSERT
                        ctx->cur_co_ctx->parent_co_ctx->co_top -= nrets;
#endif
                    }
                    // break 后会执行 resume 调用的函数
                    break;

                default:
                    /* ctx->co_op == NGX_HTTP_LUA_USER_CORO_YIELD */

                    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                                   "lua coroutine: yield");

                    ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP;

                    // entry thread 无父线程，只能忽略其返还参数，并尝试将其放入 posted_threads 中
                    if (ngx_http_lua_is_thread(ctx)) {
                        ngx_http_lua_probe_thread_yield(r, ctx->cur_co_ctx->co);

                        /* discard any return values from user
                         * coroutine.yield()'s arguments */
                        lua_settop(ctx->cur_co_ctx->co, 0);

#ifdef NGX_LUA_USE_ASSERT
                        ctx->cur_co_ctx->co_top = 0;
#endif

                        ngx_http_lua_probe_info("set co running");
                        ctx->cur_co_ctx->co_status = NGX_HTTP_LUA_CO_RUNNING;

                        if (ctx->posted_threads) {
                            ngx_http_lua_post_thread(r, ctx, ctx->cur_co_ctx);
                            ctx->cur_co_ctx = NULL;
                            return NGX_AGAIN;
                        }

                        /* no pending threads, so resume the thread
                         * immediately */

                        nrets = 0;
                        continue;
                    }

                    /* being a user coroutine that has a parent */
                    // 用户线程，找到其父线程，并将结果拷贝到父线程栈中，继续进行循环，触发父线程处理
                    nrets = lua_gettop(ctx->cur_co_ctx->co);

                    next_coctx = ctx->cur_co_ctx->parent_co_ctx;
                    next_co = next_coctx->co;

                    /*
                     * prepare return values for coroutine.resume
                     * (true plus any retvals)
                     */
                    lua_pushboolean(next_co, 1);

                    if (nrets) {
                        dd("moving %d return values to next co", nrets);
                        lua_xmove(ctx->cur_co_ctx->co, next_co, nrets);
#ifdef NGX_LUA_USE_ASSERT
                        ctx->cur_co_ctx->co_top -= nrets;
#endif
                    }

                    nrets++;  /* add the true boolean value */

                    ctx->cur_co_ctx = next_coctx;

                    break;
                }

                /* try resuming on the new coroutine again */
                continue;

            // rv == 0 线程退出
            case 0:

                ngx_http_lua_cleanup_pending_operation(ctx->cur_co_ctx);

                ngx_http_lua_probe_coroutine_done(r, ctx->cur_co_ctx->co, 1);

                ctx->cur_co_ctx->co_status = NGX_HTTP_LUA_CO_DEAD;

                if (ctx->cur_co_ctx->zombie_child_threads) {
                    ngx_http_lua_cleanup_zombie_child_uthreads(r, L, ctx,
                                                               ctx->cur_co_ctx);
                }

                ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                               "lua light thread ended normally");

                if (ngx_http_lua_is_entry_thread(ctx)) {
                    // 清空栈
                    lua_settop(L, 0);

                    ngx_http_lua_del_thread(r, L, ctx, ctx->cur_co_ctx);

                    dd("uthreads: %d", (int) ctx->uthreads);

                    if (ctx->uthreads) {

                        ctx->cur_co_ctx = NULL;
                        return NGX_AGAIN;
                    }

                    /* all user threads terminated already */
                    goto done;
                }

                // 用户线程
                if (ctx->cur_co_ctx->is_uthread) {
                    /* being a user thread */

                    lua_settop(L, 0);

                    parent_coctx = ctx->cur_co_ctx->parent_co_ctx;

                    if (ngx_http_lua_coroutine_alive(parent_coctx)) {
                        if (ctx->cur_co_ctx->waited_by_parent) {
                            ngx_http_lua_probe_info("parent already waiting");
                            ctx->cur_co_ctx->waited_by_parent = 0;
                            success = 1;
                            goto user_co_done;
                        }

                        ngx_http_lua_probe_info("parent still alive");

                        if (ngx_http_lua_post_zombie_thread(r, parent_coctx,
                                                            ctx->cur_co_ctx)
                            != NGX_OK)
                        {
                            return NGX_ERROR;
                        }

                        lua_pushboolean(ctx->cur_co_ctx->co, 1);
                        lua_insert(ctx->cur_co_ctx->co, 1);

                        ctx->cur_co_ctx->co_status = NGX_HTTP_LUA_CO_ZOMBIE;
                        ctx->cur_co_ctx = NULL;
                        return NGX_AGAIN;
                    }

                    ngx_http_lua_del_thread(r, L, ctx, ctx->cur_co_ctx);
                    ctx->uthreads--;

                    if (ctx->uthreads == 0) {
                        if (ngx_http_lua_entry_thread_alive(ctx)) {
                            ctx->cur_co_ctx = NULL;
                            return NGX_AGAIN;
                        }

                        /* all threads terminated already */
                        goto done;
                    }

                    /* some other user threads still running */
                    ctx->cur_co_ctx = NULL;
                    return NGX_AGAIN;
                }

                /* being a user coroutine that has a parent */

                success = 1;

user_co_done:

                nrets = lua_gettop(ctx->cur_co_ctx->co);

                next_coctx = ctx->cur_co_ctx->parent_co_ctx;

                if (next_coctx == NULL) {
                    /* being a light thread */
                    goto no_parent;
                }

                next_co = next_coctx->co;

                /*
                 * ended successful, coroutine.resume returns true plus
                 * any return values
                 */
                lua_pushboolean(next_co, success);

                if (nrets) {
                    lua_xmove(ctx->cur_co_ctx->co, next_co, nrets);
                }

                if (ctx->cur_co_ctx->is_uthread) {
                    ngx_http_lua_del_thread(r, L, ctx, ctx->cur_co_ctx);
                    ctx->uthreads--;
                }

                nrets++;
                ctx->cur_co_ctx = next_coctx;

                ngx_http_lua_probe_info("set parent running");

                next_coctx->co_status = NGX_HTTP_LUA_CO_RUNNING;

                ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                               "lua coroutine: lua user thread ended normally");

                continue;
            // rv == LUA_ERRRUN
            case LUA_ERRRUN:
                err = "runtime error";
                break;

            // rv == LUA_ERRSYNTAX
            case LUA_ERRSYNTAX:
                err = "syntax error";
                break;

            case LUA_ERRMEM:
                err = "memory allocation error";
                ngx_quit = 1;
                break;

            case LUA_ERRERR:
                err = "error handler error";
                break;

            default:
                err = "unknown error";
                break;
            }

            // 以下为线程执行出错处理
            if (ctx->cur_co_ctx != orig_coctx) {
                ctx->cur_co_ctx = orig_coctx;
            }

            // 栈顶存储错误信息
            if (lua_isstring(ctx->cur_co_ctx->co, -1)) {
                dd("user custom error msg");
                msg = lua_tostring(ctx->cur_co_ctx->co, -1);

            } else {
                msg = "unknown reason";
            }

            ngx_http_lua_cleanup_pending_operation(ctx->cur_co_ctx);

            ngx_http_lua_probe_coroutine_done(r, ctx->cur_co_ctx->co, 0);

            ctx->cur_co_ctx->co_status = NGX_HTTP_LUA_CO_DEAD;

            ngx_http_lua_thread_traceback(L, ctx->cur_co_ctx->co,
                                          ctx->cur_co_ctx);
            trace = lua_tostring(L, -1);

            // 用户线程
            if (ctx->cur_co_ctx->is_uthread) {
                ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                              "lua user thread aborted: %s: %s\n%s",
                              err, msg, trace);

                lua_settop(L, 0);

                parent_coctx = ctx->cur_co_ctx->parent_co_ctx;
                // 父线程存活
                if (ngx_http_lua_coroutine_alive(parent_coctx)) {
                    if (ctx->cur_co_ctx->waited_by_parent) {
                        // 父线程在等待子线程，执行父线程
                        ctx->cur_co_ctx->waited_by_parent = 0;
                        success = 0;
                        goto user_co_done;
                    }

                    if (ngx_http_lua_post_zombie_thread(r, parent_coctx,
                                                        ctx->cur_co_ctx)
                        != NGX_OK)
                    {
                        return NGX_ERROR;
                    }

                    lua_pushboolean(ctx->cur_co_ctx->co, 0);
                    lua_insert(ctx->cur_co_ctx->co, 1);

                    ctx->cur_co_ctx->co_status = NGX_HTTP_LUA_CO_ZOMBIE;
                    ctx->cur_co_ctx = NULL;
                    return NGX_AGAIN;
                }

                // 父线程已经结束
                ngx_http_lua_del_thread(r, L, ctx, ctx->cur_co_ctx);
                ctx->uthreads--;

                if (ctx->uthreads == 0) {
                    if (ngx_http_lua_entry_thread_alive(ctx)) {
                        ctx->cur_co_ctx = NULL;
                        return NGX_AGAIN;
                    }

                    /* all threads terminated already */
                    goto done;
                }

                /* some other user threads still running */
                ctx->cur_co_ctx = NULL;
                return NGX_AGAIN;
            }

            // entry 线程
            if (ngx_http_lua_is_entry_thread(ctx)) {
                ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                              "lua entry thread aborted: %s: %s\n%s",
                              err, msg, trace);

                lua_settop(L, 0);

                /* being the entry thread aborted */

                if (r->filter_finalize) {
                    ngx_http_set_ctx(r, ctx, ngx_http_lua_module);
                }

                ngx_http_lua_request_cleanup(ctx, 0);

                dd("headers sent? %d", r->header_sent || ctx->header_sent);

                if (ctx->no_abort) {
                    ctx->no_abort = 0;
                    return NGX_ERROR;
                }

                return (r->header_sent || ctx->header_sent) ? NGX_ERROR :
                       NGX_HTTP_INTERNAL_SERVER_ERROR;
            }
            // 按说不会执行到此处
            /* being a user coroutine that has a parent */

            next_coctx = ctx->cur_co_ctx->parent_co_ctx;
            if (next_coctx == NULL) {
                goto no_parent;
            }

            next_co = next_coctx->co;

            ngx_http_lua_probe_info("set parent running");

            next_coctx->co_status = NGX_HTTP_LUA_CO_RUNNING;

            /*
             * ended with error, coroutine.resume returns false plus
             * err msg
             */
            lua_pushboolean(next_co, 0);
            lua_xmove(ctx->cur_co_ctx->co, next_co, 1);
            nrets = 2;

            ctx->cur_co_ctx = next_coctx;

            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "lua coroutine: %s: %s\n%s", err, msg, trace);

            /* try resuming on the new coroutine again */
            continue;
        }

    } NGX_LUA_EXCEPTION_CATCH {
        dd("nginx execution restored");
    }

    return NGX_ERROR;

no_parent:

    lua_settop(L, 0);

    ctx->cur_co_ctx->co_status = NGX_HTTP_LUA_CO_DEAD;

    if (r->filter_finalize) {
        ngx_http_set_ctx(r, ctx, ngx_http_lua_module);
    }

    ngx_http_lua_request_cleanup(ctx, 0);

    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "lua handler aborted: "
                  "user coroutine has no parent");

    return (r->header_sent || ctx->header_sent) ?
                NGX_ERROR : NGX_HTTP_INTERNAL_SERVER_ERROR;

done:

    if (ctx->entered_content_phase
        && r->connection->fd != (ngx_socket_t) -1)
    {
        // 发送应答 header，body
        rc = ngx_http_lua_send_chain_link(r, ctx,
                                          NULL /* last_buf */);

        if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
            return rc;
        }
    }

    return NGX_OK;
}
```