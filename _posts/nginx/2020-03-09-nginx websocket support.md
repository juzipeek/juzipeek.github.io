---
layout: post
title: NGINX WebSocket 支持
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## NGINX 处理

WebSocket 处理的的关键点是 `Upgrade` 握手处理, 在 NGINX 中 `Upgrade` 是预定义的请求处理头. 在请求处理过程中会解析请求头, 并在存储在 `ngx_http_headers_in_t::upgrade` 中. 在 `proxy_pass` 模块内, 当服务端应答行为 `HTTP/1.1 101 Switching Protocols` 时, NGINX 对请求进行升级, 只进行 `package` 转发, 并且后端负载不会变化.

`proxy_pass` 模块处理上游 server 应答头, 检测到应答状态码为 `101` 时会标记需要进行升级 `upgrade`.

```c
static ngx_int_t
ngx_http_proxy_process_header(ngx_http_request_t *r)
{

    for ( ;; ) {

        rc = ngx_http_parse_header_line(r, &r->upstream->buffer, 1);
        // ... 省略无关代码
        // 上游应答头解析结束
        if (rc == NGX_HTTP_PARSE_HEADER_DONE) {
            /* a whole header has been parsed successfully */

            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "http proxy header done");

            u = r->upstream;

            if (u->headers_in.chunked) {
                u->headers_in.content_length_n = -1;
            }

            // NGX_HTTP_SWITCHING_PROTOCOLS == 101
            // 应答状态码为 101, 标记需要升级为套接字通讯
            if (u->headers_in.status_n == NGX_HTTP_SWITCHING_PROTOCOLS) {
                u->keepalive = 0;

                if (r->headers_in.upgrade) {
                    u->upgrade = 1;
                }
            }

            return NGX_OK;
        }

        // ...... 省略无关代码
        return NGX_HTTP_UPSTREAM_INVALID_HEADER;
    }
}
```

在给客户端发送应答时判断 `upgrade` 标记, 将上游读写事件修改为 `ngx_http_upstream_upgraded_read_upstream\ngx_http_upstream_upgraded_write_upstream`, 下游读写事件修改为 `ngx_http_upstream_upgraded_read_downstream\ngx_http_upstream_upgraded_write_downstream`. 当上下游读写事件到来时直接调用响应的处理函数, 而不经过 `proxy_pass` 模块的 `ngx_http_upstream_init` 等一系列回调函数.

```c
static void
ngx_http_upstream_send_response(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
    ssize_t                    n;
    ngx_int_t                  rc;
    ngx_event_pipe_t          *p;
    ngx_connection_t          *c;
    ngx_http_core_loc_conf_t  *clcf;

    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->post_action) {
        ngx_http_upstream_finalize_request(r, u, rc);
        return;
    }

    u->header_sent = 1;

    if (u->upgrade) {
        // 升级操作
        ngx_http_upstream_upgrade(r, u);
        return;
    }
    // ... 省略无关代码
}

static void
ngx_http_upstream_upgrade(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
    ngx_connection_t          *c;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    /* TODO: prevent upgrade if not requested or not possible */

    if (r != r->main) {
        ngx_log_error(NGX_LOG_ERR, c->log, 0,
                      "connection upgrade in subrequest");
        ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
        return;
    }

    r->keepalive = 0;
    c->log->action = "proxying upgraded connection";

    u->read_event_handler = ngx_http_upstream_upgraded_read_upstream;
    u->write_event_handler = ngx_http_upstream_upgraded_write_upstream;
    r->read_event_handler = ngx_http_upstream_upgraded_read_downstream;
    r->write_event_handler = ngx_http_upstream_upgraded_write_downstream;

    if (clcf->tcp_nodelay) {

        if (ngx_tcp_nodelay(c) != NGX_OK) {
            ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            return;
        }

        if (ngx_tcp_nodelay(u->peer.connection) != NGX_OK) {
            ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
            return;
        }
    }

    if (ngx_http_send_special(r, NGX_HTTP_FLUSH) == NGX_ERROR) {
        ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
        return;
    }

    if (u->peer.connection->read->ready
        || u->buffer.pos != u->buffer.last)
    {
        ngx_post_event(c->read, &ngx_posted_events);
        ngx_http_upstream_process_upgraded(r, 1, 1);
        return;
    }

    ngx_http_upstream_process_upgraded(r, 0, 1);
}
```
