---
layout: post
title: ngx_http_upstream_init_request
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## 概述

可以理解 upstream 是 NGINX 作为客户端主动创建的一个到后端的连接，并将连接事件添加到事件循环中。在用户调用 `ngx_http_upstream_init` 会创建到后端的连接，而在 `ngx_http_upstream_init` 中会调用 `ngx_http_upstream_init_request` 作为主要的逻辑处理。

## ngx_http_upstream_init_request

```c
void
ngx_http_upstream_init(ngx_http_request_t *r)
{
    ngx_connection_t     *c;

    c = r->connection;

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, c->log, 0, "http init upstream, client timer: %d", c->read->timer_set);

#if (NGX_HTTP_V2)
    if (r->stream) {
        ngx_http_upstream_init_request(r);
        return;
    }
#endif

    // 删除客户端连接上的读超时定时器
    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }

    if (ngx_event_flags & NGX_USE_CLEAR_EVENT) {
        if (!c->write->active) {
            if (ngx_add_event(c->write, NGX_WRITE_EVENT, NGX_CLEAR_EVENT)== NGX_ERROR) {
                ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }
        }
    }

    ngx_http_upstream_init_request(r);
}


static void
ngx_http_upstream_init_request(ngx_http_request_t *r)
{
    ngx_str_t                      *host;
    ngx_uint_t                      i;
    ngx_resolver_ctx_t             *ctx, temp;
    ngx_http_cleanup_t             *cln;
    ngx_http_upstream_t            *u;
    ngx_http_core_loc_conf_t       *clcf;
    ngx_http_upstream_srv_conf_t   *uscf, **uscfp;
    ngx_http_upstream_main_conf_t  *umcf;

    if (r->aio) {
        return;
    }

    u = r->upstream;

#if (NGX_HTTP_CACHE)
	... // 缓存相关
#endif

    u->store = u->conf->store;

    // 检查与下游(即客户端)之间的 TCP 连接是否断开
    if (!u->store && !r->post_action && !u->conf->ignore_client_abort) {
        r->read_event_handler = ngx_http_upstream_rd_check_broken_connection;
        r->write_event_handler = ngx_http_upstream_wr_check_broken_connection;
    }

    if (r->request_body) {
        u->request_bufs = r->request_body->bufs;
    }

    // 调用 upstream 设置的回调方法，构建到上游服务器的请求
    if (u->create_request(r) != NGX_OK) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    if (ngx_http_upstream_set_local(r, u, u->conf->local) != NGX_OK) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    u->output.alignment = clcf->directio_alignment;
    u->output.pool = r->pool;
    u->output.bufs.num = 1;
    u->output.bufs.size = clcf->client_body_buffer_size;

    if (u->output.output_filter == NULL) {
        u->output.output_filter = ngx_chain_writer;
        u->output.filter_ctx = &u->writer;
    }

    u->writer.pool = r->pool;

    // upstream_states 用于表示上游响应的状
    if (r->upstream_states == NULL) {
        r->upstream_states = ngx_array_create(r->pool, 1, sizeof(ngx_http_upstream_state_t));
        if (r->upstream_states == NULL) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }
    } else {
        u->state = ngx_array_push(r->upstream_states);
        if (u->state == NULL) {
            ngx_http_upstream_finalize_request(r, u, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }
        ngx_memzero(u->state, sizeof(ngx_http_upstream_state_t));
    }

    // upstream 清理函数
    cln = ngx_http_cleanup_add(r, 0);
    if (cln == NULL) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }
    cln->handler = ngx_http_upstream_cleanup;
    cln->data = r;
    u->cleanup = &cln->handler;

    // 上游服务器的地址
    if (u->resolved == NULL) {
        // 如果创建 upstream 的代码中没有创建 resolved 配置，那么使用 upstream 的配置参数
        uscf = u->conf->upstream;
    } else {
        // 使用创建 upstream 时设置的 resolved 作为上游服务器地址
        host = &u->resolved->host;
        umcf = ngx_http_get_module_main_conf(r, ngx_http_upstream_module);
        uscfp = umcf->upstreams.elts;

        for (i = 0; i < umcf->upstreams.nelts; i++) {
            uscf = uscfp[i];
            if (uscf->host.len == host->len
                && ((uscf->port == 0 && u->resolved->no_port) || uscf->port == u->resolved->port)
                && ngx_strncasecmp(uscf->host.data, host->data, host->len) == 0)
            {
                goto found;
            }
        }

        if (u->resolved->sockaddr) {
            if (u->resolved->port == 0 && u->resolved->sockaddr->sa_family != AF_UNIX) {
                ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "no port in upstream \"%V\"", host);
                ngx_http_upstream_finalize_request(r, u, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }

            if (ngx_http_upstream_create_round_robin_peer(r, u->resolved) != NGX_OK) {
                ngx_http_upstream_finalize_request(r, u, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return;
            }

            ngx_http_upstream_connect(r, u);

            return;
        }

        if (u->resolved->port == 0) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "no port in upstream \"%V\"", host);
            ngx_http_upstream_finalize_request(r, u, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        // 进行域名解析
        temp.name = *host;
        // clcf->resolver 是保存在 ngx_http_core_module 中，通过指令 resolver 设置
        ctx = ngx_resolve_start(clcf->resolver, &temp);
        if (ctx == NULL) {
            ngx_http_upstream_finalize_request(r, u, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }

        if (ctx == NGX_NO_RESOLVER) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, "no resolver defined to resolve %V", host);
            ngx_http_upstream_finalize_request(r, u, NGX_HTTP_BAD_GATEWAY);
            return;
        }

        ctx->name = *host;
        ctx->handler = ngx_http_upstream_resolve_handler;
        ctx->data = r;
        ctx->timeout = clcf->resolver_timeout;

        u->resolved->ctx = ctx;

        // 进行域名解析
        if (ngx_resolve_name(ctx) != NGX_OK) {
            u->resolved->ctx = NULL;
            ngx_http_upstream_finalize_request(r, u, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }
        return;
    }

found:

    if (uscf == NULL) {
        ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0, "no upstream configuration");
        ngx_http_upstream_finalize_request(r, u, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    u->upstream = uscf;

#if (NGX_HTTP_SSL)
    u->ssl_name = uscf->host;
#endif

    if (uscf->peer.init(r, uscf) != NGX_OK) {
        ngx_http_upstream_finalize_request(r, u, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    u->peer.start_time = ngx_current_msec;
    if (u->conf->next_upstream_tries && u->peer.tries > u->conf->next_upstream_tries) {
        u->peer.tries = u->conf->next_upstream_tries;
    }
    
	// 连接上游服务器
    ngx_http_upstream_connect(r, u);
}
```

## resolver 指令说明

```nginx.conf
Syntax:	resolver address ... [valid=time] [ipv6=on|off];
Default:	—
Context:	http, server, location
```

在 `resolver` 指令中可以指定多个 dns，设置重置域名 ttl 值。