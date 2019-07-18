---
layout: post
title: NGX_HTTP_LOG_PHASE 阶段介绍
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## phase_engine初始化

NGINX 中每个阶段的处理函数是在 `ngx_http_core_run_phases` 函数中调用

```c
void
ngx_http_core_run_phases(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_phase_handler_t   *ph;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    ph = cmcf->phase_engine.handlers;

    while (ph[r->phase_handler].checker) {

        rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

        if (rc == NGX_OK) {
            return;
        }
    }
}

```

其中 `phase_engine` 初始化如下：

```c
static ngx_int_t
ngx_http_init_phase_handlers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf)
{
    ngx_int_t                   j;
    ngx_uint_t                  i, n;
    ngx_uint_t                  find_config_index, use_rewrite, use_access;
    ngx_http_handler_pt        *h;
    ngx_http_phase_handler_t   *ph;
    ngx_http_phase_handler_pt   checker;

    cmcf->phase_engine.server_rewrite_index = (ngx_uint_t) -1;
    cmcf->phase_engine.location_rewrite_index = (ngx_uint_t) -1;
    find_config_index = 0;
    use_rewrite = cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers.nelts ? 1 : 0;
    use_access = cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers.nelts ? 1 : 0;

    n = 1                  /* find config phase */
        + use_rewrite      /* post rewrite phase */
        + use_access       /* post access phase */
        + cmcf->try_files;

    for (i = 0; i < NGX_HTTP_LOG_PHASE; i++) {
        n += cmcf->phases[i].handlers.nelts;
    }

    ph = ngx_pcalloc(cf->pool, n * sizeof(ngx_http_phase_handler_t) + sizeof(void *));
    if (ph == NULL) {
        return NGX_ERROR;
    }

    cmcf->phase_engine.handlers = ph;
    n = 0;

    for (i = 0; i < NGX_HTTP_LOG_PHASE; i++) {
        h = cmcf->phases[i].handlers.elts;

        switch (i) {

        case NGX_HTTP_SERVER_REWRITE_PHASE:
            if (cmcf->phase_engine.server_rewrite_index == (ngx_uint_t) -1) {
                cmcf->phase_engine.server_rewrite_index = n;
            }
            checker = ngx_http_core_rewrite_phase;

            break;

        case NGX_HTTP_FIND_CONFIG_PHASE:
            find_config_index = n;

            ph->checker = ngx_http_core_find_config_phase;
            n++;
            ph++;

            continue;

        case NGX_HTTP_REWRITE_PHASE:
            if (cmcf->phase_engine.location_rewrite_index == (ngx_uint_t) -1) {
                cmcf->phase_engine.location_rewrite_index = n;
            }
            checker = ngx_http_core_rewrite_phase;

            break;

        case NGX_HTTP_POST_REWRITE_PHASE:
            if (use_rewrite) {
                ph->checker = ngx_http_core_post_rewrite_phase;
                ph->next = find_config_index;
                n++;
                ph++;
            }

            continue;

        case NGX_HTTP_ACCESS_PHASE:
            checker = ngx_http_core_access_phase;
            n++;
            break;

        case NGX_HTTP_POST_ACCESS_PHASE:
            if (use_access) {
                ph->checker = ngx_http_core_post_access_phase;
                ph->next = n;
                ph++;
            }

            continue;

        case NGX_HTTP_TRY_FILES_PHASE:
            if (cmcf->try_files) {
                ph->checker = ngx_http_core_try_files_phase;
                n++;
                ph++;
            }

            continue;

        case NGX_HTTP_CONTENT_PHASE:
            checker = ngx_http_core_content_phase;
            break;

        default:
            checker = ngx_http_core_generic_phase;
        }

        n += cmcf->phases[i].handlers.nelts;

        for (j = cmcf->phases[i].handlers.nelts - 1; j >=0; j--) {
            ph->checker = checker;
            ph->handler = h[j];
            ph->next = n;
            ph++;
        }
    }

    return NGX_OK;
}

```

从上面可以看到 `phase_engine` 中不存在 `NGX_HTTP_LOG_PHASE` 阶段的处理，所以在 `ngx_http_core_run_phases` 中不会调用日志处理阶段。

## `NGX_HTTP_LOG_PHASE` 阶段调用

在 `NGINX` 源码中搜索 `NGX_HTTP_LOG_PHASE`，可以找到日志记录阶段的调用函数 `ngx_http_log_request`：

```c
static void
ngx_http_log_request(ngx_http_request_t *r)
{
    ngx_uint_t                  i, n;
    ngx_http_handler_pt        *log_handler;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    log_handler = cmcf->phases[NGX_HTTP_LOG_PHASE].handlers.elts;
    n = cmcf->phases[NGX_HTTP_LOG_PHASE].handlers.nelts;

    for (i = 0; i < n; i++) {
        log_handler[i](r);
    }
}
```

而 `ngx_http_log_request` 在整个 `NGINX` 源码中有两个调用点 `ngx_http_finalize_request` 和 `ngx_http_free_request`。可以看到日志阶段是在给出应答后才进行调用，`NGINX` 提供的 `$request_time` 耗时是从接收到请求信息到给出应答，并且已被客户端接收的耗时。

## `ngx_http_log_module`

```nginx
Syntax:	access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
access_log off;
Default:	access_log logs/access.log combined;
Context:	http, server, location, if in location, limit_except
```

`access_log` 指令用来配置访问日志的路径、格式以及缓存。在 `format` 参数中可以使用 `syslog:` 前缀来指定使用 `syslog` 进行日志收集。如果在当前层级（`http{}`、`server{}`、`location{}` 等）使用 `access_log off` 配置，会在当前层级禁止日志输出。

如果使用 `buffer` 或 `gzip` 参数，`access_log` 会对输出进行缓存。`buffer` 参数会指定缓存的大小，`flush` 参数可以指定 `access_log` 从缓存刷新到文件的时间。触发缓存内容写入磁盘的条件：

- 在下次写 `log` 时内容无法在 `buffer` 中缓存；
- `buffer` 中的内容以及比 `flush` 参数指定的时间更长；
- `worker` 在重新打开文件或者关闭状态。

如果配置了 `gzip` 参数，日志被写入磁盘钱会进行压缩，可以指定压缩等级（1-9，1 速度最快、压缩比低）。使用 `gzip` 时 `buffer` 参数默认为 64K 字节。

其实日志目录参数 `path` 可以使用变量，不过使用变量后在每次写日志时会打开日志文件、写日志、关闭，因此不能使用 `buffer` 参数。使用 `open_log_file_cache` 可以缓存文件描述符。

```nginx
Syntax:	open_log_file_cache max=N [inactive=time] [min_uses=N] [valid=time];
open_log_file_cache off;
Default:	open_log_file_cache off;
Context:	http, server, location
```

定义一个缓存，用来保存 `access_log` 的 `path` 配置中包含变量文件描述符。

| 参数       | 含义                                                         | 备注            |
| ---------- | ------------------------------------------------------------ | --------------- |
| `max`      | 缓存可以存储文件描述符最多 `max` 个                          | 使用 `LRU` 踢除 |
| `inactive` | 缓存中文件描述符允许不活跃时间，超过此时间会关闭             |                 |
| `min_uses` | 与 `inactive` 配合，在 `inactive` 时间内使用次数小于 `min_uses` 值同样会关闭文件描述符。默认为 1。 |                 |
| `valid`    | 文件名有效期设置；超过此时间需要检查文件名是否发生变化（因为使用变量） |                 |

### 模块思路

模块实现思路比较简单，模块在指令解析过程中会创建 `ngx_http_log_t` 配置结构，并保存在模块的 `main_conf` 中。在 `postconfiguration` 阶段会在 `NGX_HTTP_LOG_PHASE` 处理阶段添加处理函数。`postconfiguration` 阶段处理函数：

```c
static ngx_int_t
ngx_http_log_init(ngx_conf_t *cf)
{
    ngx_str_t                  *value;
    ngx_array_t                 a;
    ngx_http_handler_pt        *h;
    ngx_http_log_fmt_t         *fmt;
    ngx_http_log_main_conf_t   *lmcf;
    ngx_http_core_main_conf_t  *cmcf;

    lmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_log_module);

    if (lmcf->combined_used) {
        if (ngx_array_init(&a, cf->pool, 1, sizeof(ngx_str_t)) != NGX_OK) {
            return NGX_ERROR;
        }

        value = ngx_array_push(&a);
        if (value == NULL) {
            return NGX_ERROR;
        }

        *value = ngx_http_combined_fmt;
        fmt = lmcf->formats.elts;

        if (ngx_http_log_compile_format(cf, NULL, fmt->ops, &a, 0)
            != NGX_CONF_OK)
        {
            return NGX_ERROR;
        }
    }

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_LOG_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_log_handler;

    return NGX_OK;
}
```

