---
layout: post
title: Nginx stream 模块
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## stream 模块

在 `NGINX` 内部四层协议处理是从 `ngx_stream_module` 模块的 `stream` 指令处理开始。`ngx_stream_module` 是核心模块，作用与 `ngx_http_module` 相同。

## 处理阶段定义

在 `NGX_STREAM_MODULE` 模块中定义了：`NGX_STREAM_POST_ACCEPT_PHASE`、`NGX_STREAM_PREACCESS_PHASE`、`NGX_STREAM_ACCESS_PHASE`、`NGX_STREAM_SSL_PHASE`、`NGX_STREAM_PREREAD_PHASE`、`NGX_STREAM_CONTENT_PHASE`、`NGX_STREAM_LOG_PHASE` 七个处理阶段。阅读 `ngx_stream_block` 中调用的 `ngx_stream_init_phase_handlers` 函数代码：

```c
static ngx_int_t
ngx_stream_init_phase_handlers(ngx_conf_t *cf, ngx_stream_core_main_conf_t *cmcf)
{
    ngx_int_t                     j;
    ngx_uint_t                    i, n;
    ngx_stream_handler_pt        *h;
    ngx_stream_phase_handler_t   *ph;
    ngx_stream_phase_handler_pt   checker;

    n = 1 /* content phase */;

    for (i = 0; i < NGX_STREAM_LOG_PHASE; i++) {
        n += cmcf->phases[i].handlers.nelts;
    }

    ph = ngx_pcalloc(cf->pool, n * sizeof(ngx_stream_phase_handler_t) + sizeof(void *));
    if (ph == NULL) {
        return NGX_ERROR;
    }

    cmcf->phase_engine.handlers = ph;
    n = 0;

    for (i = 0; i < NGX_STREAM_LOG_PHASE; i++) {
        h = cmcf->phases[i].handlers.elts;

        switch (i) {
        case NGX_STREAM_PREREAD_PHASE:
            checker = ngx_stream_core_preread_phase;
            break;

        // 最后的 continue; 说明这个阶段只能有一个处理函数
        case NGX_STREAM_CONTENT_PHASE:
            ph->checker = ngx_stream_core_content_phase;
            n++;
            ph++;
            continue;

        default:
            checker = ngx_stream_core_generic_phase;
        }

        n += cmcf->phases[i].handlers.nelts;

        for (j = cmcf->phases[i].handlers.nelts - 1; j >= 0; j--) {
            ph->checker = checker;
            ph->handler = h[j];
            ph->next = n;
            ph++;
        }
    }

    return NGX_OK;
}

```

七个阶段中除了 `NGX_STREAM_CONTENT_PHASE` 阶段不允许介入，其他阶段用户都可以介入处理。

## ngx_stream_optimize_servers

调用 `ngx_stream_add_ports` 和 `ngx_stream_optimize_servers` 添加端口监听的回调函数。在 `ngx_stream_optimize_servers` 函数中，把绑定的 `IP` 和端口以及对应的配置赋值到所监听结构体的 `servers` 的变量。添加的目的主要是为了能够在请求到达的时候找到对应的配置，也就是为了根据五元组中目的 `IP` 和端口来获取监听该 `IP` 和端口所对应的配置。

```c
static char *
ngx_stream_optimize_servers(ngx_conf_t *cf, ngx_array_t *ports)
{
    ngx_uint_t                   i, p, last, bind_wildcard;
    ngx_listening_t             *ls;
    ngx_stream_port_t           *stport;
    ngx_stream_conf_port_t      *port;
    ngx_stream_conf_addr_t      *addr;
    ngx_stream_core_srv_conf_t  *cscf;

    port = ports->elts;
    for (p = 0; p < ports->nelts; p++) {
		// 对 PORT 下的 IP 地址排序。宽绑定（通配符绑定）的排序到最后。
        // 设置了特殊标识需要 BIND 的排在最前。不需要 BIND 的排到中间位置。
        ngx_sort(port[p].addrs.elts, (size_t) port[p].addrs.nelts, sizeof(ngx_stream_conf_addr_t), ngx_stream_cmp_conf_addrs);

        addr = port[p].addrs.elts;
        last = port[p].addrs.nelts;

        /*
         * if there is the binding to the "*:port" then we need to bind()
         * to the "*:port" only and ignore the other bindings
         */

        // 最后一个地址是否是宽绑定的，
        // 如果是那么没有特殊标识设置的就不需要绑定监听了，
        // 只需要绑定监听这个宽绑定 IP 就可以。
        if (addr[last - 1].opt.wildcard) {
            addr[last - 1].opt.bind = 1;
            bind_wildcard = 1;

        } else {
            bind_wildcard = 0;
        }

        i = 0;

        while (i < last) {
			// 有宽绑定且不需要单独绑定
            if (bind_wildcard && !addr[i].opt.bind) {
                i++;
                continue;
            }

            // 如果没有宽绑定，每个地址都需要监听
            // 如果有宽绑定，只需要绑定设置了特殊标识的地址和宽绑定地址，
            // 不需要绑定监听的都会被添加到宽绑定地址的监听数组中(ls->servers->addrs)
            ls = ngx_create_listening(cf, &addr[i].opt.sockaddr.sockaddr, addr[i].opt.socklen);
            if (ls == NULL) {
                return NGX_CONF_ERROR;
            }

            ls->addr_ntop = 1;
            ls->handler = ngx_stream_init_connection;
            ls->pool_size = 256;
            ls->type = addr[i].opt.type;

            cscf = addr->opt.ctx->srv_conf[ngx_stream_core_module.ctx_index];

            ls->logp = cscf->error_log;
            ls->log.data = &ls->addr_text;
            ls->log.handler = ngx_accept_log_error;
            ls->backlog = addr[i].opt.backlog;
            ls->rcvbuf = addr[i].opt.rcvbuf;
            ls->sndbuf = addr[i].opt.sndbuf;
            ls->wildcard = addr[i].opt.wildcard;

            ls->keepalive = addr[i].opt.so_keepalive;
#if (NGX_HAVE_KEEPALIVE_TUNABLE)
            ls->keepidle = addr[i].opt.tcp_keepidle;
            ls->keepintvl = addr[i].opt.tcp_keepintvl;
            ls->keepcnt = addr[i].opt.tcp_keepcnt;
#endif

#if (NGX_HAVE_INET6)
            ls->ipv6only = addr[i].opt.ipv6only;
#endif

#if (NGX_HAVE_REUSEPORT)
            ls->reuseport = addr[i].opt.reuseport;
#endif

            stport = ngx_palloc(cf->pool, sizeof(ngx_stream_port_t));
            if (stport == NULL) {
                return NGX_CONF_ERROR;
            }
            ls->servers = stport;
            stport->naddrs = i + 1;

            switch (ls->sockaddr->sa_family) {
#if (NGX_HAVE_INET6)
            case AF_INET6:
                if (ngx_stream_add_addrs6(cf, stport, addr) != NGX_OK) {
                    return NGX_CONF_ERROR;
                }
                break;
#endif
            default: /* AF_INET */
                if (ngx_stream_add_addrs(cf, stport, addr) != NGX_OK) {
                    return NGX_CONF_ERROR;
                }
                break;
            }

            if (ngx_clone_listening(cf, ls) != NGX_OK) {
                return NGX_CONF_ERROR;
            }

            addr++;
            last--;
        }
    }

    return NGX_CONF_OK;
}

```

[本节摘自网络](https://github.com/vislee/leevis.com/issues/139)

## NGX_STREAM_CONTENT_PHASE 阶段设置处理函数方法

参见 `ngx_stream_proxy_module` 模块的 `proxy_pass` 指令处理函数：

```c
static char *
ngx_stream_proxy_pass(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_stream_proxy_srv_conf_t *pscf = conf;

    ngx_url_t                            u;
    ngx_str_t                           *value, *url;
    ngx_stream_complex_value_t           cv;
    ngx_stream_core_srv_conf_t          *cscf;
    ngx_stream_compile_complex_value_t   ccv;

    if (pscf->upstream || pscf->upstream_value) {
        return "is duplicate";
    }

    cscf = ngx_stream_conf_get_module_srv_conf(cf, ngx_stream_core_module);

    // 处理函数设置
    cscf->handler = ngx_stream_proxy_handler;

    value = cf->args->elts;

    url = &value[1];

    ngx_memzero(&ccv, sizeof(ngx_stream_compile_complex_value_t));

    ccv.cf = cf;
    ccv.value = url;
    ccv.complex_value = &cv;

    if (ngx_stream_compile_complex_value(&ccv) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    if (cv.lengths) {
        pscf->upstream_value = ngx_palloc(cf->pool, sizeof(ngx_stream_complex_value_t));
        if (pscf->upstream_value == NULL) {
            return NGX_CONF_ERROR;
        }

        *pscf->upstream_value = cv;

        return NGX_CONF_OK;
    }

    ngx_memzero(&u, sizeof(ngx_url_t));

    u.url = *url;
    u.no_resolve = 1;

    pscf->upstream = ngx_stream_upstream_add(cf, &u, 0);
    if (pscf->upstream == NULL) {
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}
```

kef