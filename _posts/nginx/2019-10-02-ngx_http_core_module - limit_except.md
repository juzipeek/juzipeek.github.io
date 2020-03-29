---
layout: post
title: ngx_http_core_module - limit_except
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description:
---

## 一 概述

`limit_except` 限制一个 `location` 内允许的 HTTP 方法。

## 二 指令

```nginx
Syntax: 	limit_except method ... { ... }
Default: 	—
Context: 	location
```

可选的 `method` ：GET, HEAD, POST, PUT, DELETE, MKCOL, COPY, MOVE, OPTIONS, PROPFIND, PROPPATCH, LOCK, UNLOCK, PATCH。示例：

```nginx
limit_except GET {
    allow 192.168.1.0/32;
    deny  all;
}
```

配置含义是允许所有的 `GET`（`HEAD`） 请求，其他类型的请求如果来源非 `192.168.1.0/32` 则禁止访问。

如果使用 `POST` 方式进行请求会返回：

```
403 Forbidden
```

## 三 实现

其实 `limit_except` 指令主要是根据方法使用不同的 `loc_conf` 配置项。`ngx_http_update_location_config` 会根据请求方法判断是否需要更新请求的 `loc_conf`。当时有 `limit_except` 配置的 `loc_conf` 后，在 `ngx_http_access_module` 模块会对请求客户端地址进行判断。

指令处理：

```c
static char *
ngx_http_core_limit_except(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t *pclcf = conf;

    char                      *rv;
    void                      *mconf;
    ngx_str_t                 *value;
    ngx_uint_t                 i;
    ngx_conf_t                 save;
    ngx_http_module_t         *module;
    ngx_http_conf_ctx_t       *ctx, *pctx;
    ngx_http_method_name_t    *name;
    ngx_http_core_loc_conf_t  *clcf;

    if (pclcf->limit_except) {
        return "duplicate";
    }

    pclcf->limit_except = 0xffffffff;

    value = cf->args->elts;

    for (i = 1; i < cf->args->nelts; i++) {
        for (name = ngx_methods_names; name->name; name++) {

            if (ngx_strcasecmp(value[i].data, name->name) == 0) {
                pclcf->limit_except &= name->method;
                goto next;
            }
        }

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid method \"%V\"", &value[i]);
        return NGX_CONF_ERROR;

    next:
        continue;
    }

    if (!(pclcf->limit_except & NGX_HTTP_GET)) {
        pclcf->limit_except &= (uint32_t) ~NGX_HTTP_HEAD;
    }

    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }

    pctx = cf->ctx;
    ctx->main_conf = pctx->main_conf;
    ctx->srv_conf = pctx->srv_conf;

    // 创建新的 loc_conf 配置
    ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->loc_conf == NULL) {
        return NGX_CONF_ERROR;
    }

    for (i = 0; cf->cycle->modules[i]; i++) {
        if (cf->cycle->modules[i]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[i]->ctx;

        if (module->create_loc_conf) {

            mconf = module->create_loc_conf(cf);
            if (mconf == NULL) {
                return NGX_CONF_ERROR;
            }

            ctx->loc_conf[cf->cycle->modules[i]->ctx_index] = mconf;
        }
    }


    clcf = ctx->loc_conf[ngx_http_core_module.ctx_index];
    pclcf->limit_except_loc_conf = ctx->loc_conf;
    clcf->loc_conf = ctx->loc_conf;
    clcf->name = pclcf->name;
    clcf->noname = 1;
    clcf->lmt_excpt = 1;

    if (ngx_http_add_location(cf, &pclcf->locations, clcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    // 使用新的 loc_conf 进行解析,解析 deny、allow 指令。
    // 其实 limit_except 指令主要是根据方法使用不同的 loc_conf 配置项。
    // ngx_http_update_location_config 会根据请求方法判断是否需要更新请求的 loc_conf 
    save = *cf;
    cf->ctx = ctx;
    cf->cmd_type = NGX_HTTP_LMT_CONF;
		// 对 limit_except {} 进行指令处理
    // 只有 allow、deny 可以在 LMT_CONF 中配置
    rv = ngx_conf_parse(cf, NULL);

    *cf = save;

    return rv;
}
```

在请求处理过程中更新请求配置：

```c
void
ngx_http_update_location_config(ngx_http_request_t *r)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
		// 更新配置
    if (r->method & clcf->limit_except) {
        r->loc_conf = clcf->limit_except_loc_conf;
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    }

    // ... 省略其他代码
}

```

`allow`、`deny` 指令定义：

```c
static ngx_command_t  ngx_http_access_commands[] = {
    // 可以出来 NGX_HTTP_LMT_CONF 块内配置
    { ngx_string("allow"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LMT_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_access_rule,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("deny"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LMT_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_access_rule,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

      ngx_null_command
};
```

