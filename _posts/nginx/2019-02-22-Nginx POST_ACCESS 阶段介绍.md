---
layout: post
title: Nginx POST_ACCESS 阶段介绍
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## 功能介绍

`NGX_HTTP_POST_ACCESS_PHASE` 作为 Nginx 11 个处理阶段中不可修改的阶段其主要功能是判断 `NGX_HTTP_ACCESS_PHASE` 阶段处理的访问码是否有设置，如果有设置则给客户返回应答码。

总结来说，`NGX_HTTP_POST_ACCESS_PHASE` 用来给 `NGX_HTTP_ACCESS_PHASE` 阶段收尾，在 `NGX_HTTP_ACCESS_PHASE` 阶段仅关注是否授权访问，`NGX_HTTP_POST_ACCESS_PHASE` 阶段根据结果对连接做出处理。

## NGINX 框架代码说明

```c
// 各阶段处理函数初始化
static ngx_int_t
ngx_http_init_phase_handlers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf)
{
...
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

		// 本阶段不允许其他处理函数介入
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

// post_access_phase 阶段 checker 函数
// 本阶段其实无处理函数，只有 checker 函数。ngx_http_core_post_access_phase 作为 checker 函数被调用
ngx_int_t
ngx_http_core_post_access_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    ngx_int_t  access_code;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "post access phase: %ui", r->phase_handler);

    access_code = r->access_code;
	// 只要设置了 access_code 就会返回 access_code 应答码
    if (access_code) {
        if (access_code == NGX_HTTP_FORBIDDEN) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "access forbidden by rule");
        }

        r->access_code = 0;
        ngx_http_finalize_request(r, access_code);
        return NGX_OK;
    }

    r->phase_handler++;
    return NGX_AGAIN;
}
```

## 总结

在写 `ACCESS` 阶段处理函数时，并不需要直接将连接关闭，更好的方式是设置当前请求的 `r->access_code`，最终由 `NGX_HTTP_POST_ACCESS_PHASE` 进行应答处理。

只有将目的想清楚，功能点才能切分好设计才会清晰。