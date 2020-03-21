---
layout: post
title: OpenResty Api - worker id
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description:
---

## 一 概述

在使用 OpenResty 实现业务逻辑时经常需要只在一个 worker 内执行某项动作, 例如读取配置文件到共享内存. 可以使用 `ngx.worker.id` 函数获取 worker 序号, 相对于 `ngx.worker.pid` 得到进程 pid 更有用.

获取 worker 序号, 进程 pid 功能是使用 FFI 技术, 由 `resty.core.worker` 包对外提供服务(实现 FFI 库时应该使用此种方式, 避免在业务代码中直接通过 FFI 调用 C 函数).

`ngx.worker.pid` 调用 `lua-nginx` 模块的 `ngx_http_lua_ffi_worker_pid` 函数(从全局变量 ngx_pid 中获得进程 pid); `ngx.worker.id` 调用 `lua-nginx`模块的 `ngx_http_lua_ffi_worker_id` 函数. 

对于 `ngx.worker.id` 进程序号需要考虑怎么给进程分配序号这件事情, 序号什么时候会发生变化.
首先, 给进程分配序号是在 worker 启动时确定的: master 根据配置的 worker 进程数循环计数, 并将此计数作为参数传递给 worker 进程, **worker 进程获得的参数就是进程序号**. 第二, 其实第一点确定了序号是怎么产生的, 那只需要什么时候 worker 会启动, 分析此时启动 worker 的参数即可确定.

在 nginx 进程生命周期中有三种情况会启动 worker 进程: 初始运行(new binary 等同初始运行), 热加载, master 拉起挂掉 worker. 初始运行与热加载相同逻辑, 都需要启动一组新的 worker 进程; worker 挂掉后 master 重新拉起 worker 时会使用进程数组中保存的信息启动 worker, **会与原槽位的 worker 持有相同的序号**.

## 二 分配序号

```c
// master 启动 worker 进程
static void
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    ngx_int_t      i;
    ngx_channel_t  ch;

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start worker processes");

    ngx_memzero(&ch, sizeof(ngx_channel_t));

    ch.command = NGX_CMD_OPEN_CHANNEL;

    for (i = 0; i < n; i++) {

        // 将序号 i 传递给 worker 进程
        ngx_spawn_process(cycle, ngx_worker_process_cycle,
                          (void *) (intptr_t) i, "worker process", type);

        ch.pid = ngx_processes[ngx_process_slot].pid;
        ch.slot = ngx_process_slot;
        ch.fd = ngx_processes[ngx_process_slot].channel[0];

        ngx_pass_open_channel(cycle, &ch);
    }
}
static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    // 序号参数保存到全局变量 ngx_worker 中
    ngx_int_t worker = (intptr_t) data;
    ngx_worker = worker;

    ngx_worker_process_init(cycle, worker);

    ngx_setproctitle("worker process");
    ... // 省略无关代码
}
```

## 三 重新拉起 worker

在启动时 nginx 会注册 `SIGCHLD` 信号处理函数, 当 worker 挂掉时会调用信号处理函数, 设置 master 进程全局变量 `ngx_reap=1`. 在 master 进程的主循环逻辑中根据 `ngx_reap` 变量进入 `ngx_reap_children` 处理.

```c
static ngx_uint_t
ngx_reap_children(ngx_cycle_t *cycle)
{
    ngx_int_t         i, n;
    ngx_uint_t        live;
    ngx_channel_t     ch;
    ngx_core_conf_t  *ccf;

    ngx_memzero(&ch, sizeof(ngx_channel_t));

    ch.command = NGX_CMD_CLOSE_CHANNEL;
    ch.fd = -1;

    live = 0;
    for (i = 0; i < ngx_last_process; i++) {

        ngx_log_debug7(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "child: %i %P e:%d t:%d d:%d r:%d j:%d",
                       i,
                       ngx_processes[i].pid,
                       ngx_processes[i].exiting,
                       ngx_processes[i].exited,
                       ngx_processes[i].detached,
                       ngx_processes[i].respawn,
                       ngx_processes[i].just_spawn);

        if (ngx_processes[i].pid == -1) {
            continue;
        }

        if (ngx_processes[i].exited) {
            ...
            // 重新拉起 worker
            if (ngx_processes[i].respawn
                && !ngx_processes[i].exiting
                && !ngx_terminate
                && !ngx_quit)
            {
                // 使用存储在 ngx_processes 中的序号(data) 启动 worker 进程
                if (ngx_spawn_process(cycle, ngx_processes[i].proc,
                                      ngx_processes[i].data,
                                      ngx_processes[i].name, i)
                    == NGX_INVALID_PID)
                {
                    ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                                  "could not respawn %s",
                                  ngx_processes[i].name);
                    continue;
                }


                ch.command = NGX_CMD_OPEN_CHANNEL;
                ch.pid = ngx_processes[ngx_process_slot].pid;
                ch.slot = ngx_process_slot;
                ch.fd = ngx_processes[ngx_process_slot].channel[0];

                ngx_pass_open_channel(cycle, &ch);

                live = 1;

                continue;
            }

            ...

            if (i == ngx_last_process - 1) {
                ngx_last_process--;

            } else {
                ngx_processes[i].pid = -1;
            }

        } else if (ngx_processes[i].exiting || !ngx_processes[i].detached) {
            live = 1;
        }
    }

    return live;
}

// 在 master 第一次启动时同样会进入此函数; data 为 worker 序号
ngx_pid_t
ngx_spawn_process(ngx_cycle_t *cycle, ngx_spawn_proc_pt proc, void *data,
    char *name, ngx_int_t respawn)
{
    u_long     on;
    ngx_pid_t  pid;
    ngx_int_t  s;

    ... // 省略无关代码
    ngx_process_slot = s;

    pid = fork();

    switch (pid) {

    case -1:
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "fork() failed while spawning \"%s\"", name);
        ngx_close_channel(ngx_processes[s].channel, cycle->log);
        return NGX_INVALID_PID;

    case 0:
        ngx_parent = ngx_pid;
        ngx_pid = ngx_getpid();
        // 子进程作为 worker, 执行 work 处理逻辑, 将 data 传个 worker
        proc(cycle, data);
        break;

    default:
        break;
    }

    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start %s %P", name, pid);

    ngx_processes[s].pid = pid;
    ngx_processes[s].exited = 0;

    // master 拉起 worker 时 respawn >=0
    if (respawn >= 0) {
        return pid;
    }

    // master 初始启动 worker 时会走到此次; respawn 值为 NGX_PROCESS_RESPAWN
    // 将 worker 分配序号存储在 ngx_processes 数组中
    ngx_processes[s].proc = proc;
    ngx_processes[s].data = data;
    ngx_processes[s].name = name;
    ngx_processes[s].exiting = 0;

    switch (respawn) {

    case NGX_PROCESS_NORESPAWN:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_JUST_SPAWN:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 1;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_RESPAWN:
        ngx_processes[s].respawn = 1;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_JUST_RESPAWN:
        ngx_processes[s].respawn = 1;
        ngx_processes[s].just_spawn = 1;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_DETACHED:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 1;
        break;
    }

    if (s == ngx_last_process) {
        ngx_last_process++;
    }

    return pid;
}
```

## 四 获取进程序号

```c
int
ngx_http_lua_ffi_worker_id(void)
{
#if (nginx_version >= 1009001)
    if (ngx_process != NGX_PROCESS_WORKER
        && ngx_process != NGX_PROCESS_SINGLE)
    {
        return -1;
    }

    // 直接返回全局变量
    return (int) ngx_worker;
#else
    return -1;
#endif
}
```
