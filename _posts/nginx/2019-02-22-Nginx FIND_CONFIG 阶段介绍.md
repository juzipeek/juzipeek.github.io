---
layout: post
title: Nginx FIND_CONFIG 阶段介绍
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## location 匹配规则

location 的 5 种可选配置前缀：

`~`：正则匹配（区分大小写）

`~*`：正则匹配（不区分大小写）

`=`：精确匹配

`^~`：只匹配字符串，不查询正则表达式

`@`：制定一个命名的 location

上述前缀中，只有 `~` 和 `~*` 是正则匹配，对于 NGINX 来说，优先级最高的是 `=` 匹配，接下来是普通匹配，最后会去匹配正则匹配。
使用如下规则：

1. 如果 `=` 匹配符合，会立即使用该 location 配置，而不会对后续 location 配置进行匹配。
2. 对于 @ 匹配，只有请求是内部重定向请求时才会使用，普通请求不会进行匹配。
3. 对于没有匹配成功上述两种匹配的所有普通匹配的 location，NGINX 进行最长前缀匹配，匹配结果与配置文件中 location 的编辑顺序并没有关系。
4. 得到普通匹配的最长前缀匹配 location 以后，如果这个 location 不是 `^~` 匹配（标识不查询正则表达式），那么并不会结束匹配过程，而是会继续进行正则匹配，按照配置文件中 location 的编辑顺序得到首个匹配成功的正则匹配 location。
5. 如果同时匹配到了普通匹配和正则匹配的 location 结果，nginx 会选取正则匹配的 location 作为最终的匹配结果。

## 功能介绍

`NGX_HTTP_FIND_CONFIG_PHASE` 是 Nginx 中不可介入的处理阶段，主要功能是对当前请求进行 `location` 查找。查找的顺序：

1.  根据请求获取请求所在 server 块的 loc_conf 配置
2.  从 clcf 的静态 location 三叉树（左右子树以及当前节点的子树）查找，匹配最大前缀匹配
3.  进行正则表达式查找

## NGINX 框架代码

```c
// FIND_CONFIG 阶段的 checker 函数
ngx_int_t
ngx_http_core_find_config_phase(ngx_http_request_t *r, ngx_http_phase_handler_t *ph)
{
    u_char                    *p;
    size_t                     len;
    ngx_int_t                  rc;
    ngx_http_core_loc_conf_t  *clcf;

    r->content_handler = NULL;
    r->uri_changed = 0;

    // 查找 location
    rc = ngx_http_core_find_location(r);

    if (rc == NGX_ERROR) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_OK;
    }

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    // 本请求非内部请求，location 是内部 location => 拒绝访问
    if (!r->internal && clcf->internal) {
        ngx_http_finalize_request(r, NGX_HTTP_NOT_FOUND);
        return NGX_OK;
    }

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "using configuration \"%s%V\"",
                   (clcf->noname ? "*" : (clcf->exact_match ? "=" : "")),
                   &clcf->name);

    // 根据查找到的 location 更新请求的属性
    ngx_http_update_location_config(r);

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http cl:%O max:%O",
                   r->headers_in.content_length_n, clcf->client_max_body_size);

    // 检查 location 内对包体大小的限制以及当前请求包体是否大于限制
    if (r->headers_in.content_length_n != -1
        && !r->discard_body
        && clcf->client_max_body_size
        && clcf->client_max_body_size < r->headers_in.content_length_n)
    {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "client intended to send too large body: %O bytes",
                      r->headers_in.content_length_n);

        r->expect_tested = 1;
        (void) ngx_http_discard_request_body(r);
        ngx_http_finalize_request(r, NGX_HTTP_REQUEST_ENTITY_TOO_LARGE);
        return NGX_OK;
    }

    // 301 重定向处理
    // 返回 301 应答码，并设置 http 头指向新的 location
    if (rc == NGX_DONE) {
        ngx_http_clear_location(r);

        r->headers_out.location = ngx_list_push(&r->headers_out.headers);
        if (r->headers_out.location == NULL) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return NGX_OK;
        }

        r->headers_out.location->hash = 1;
        ngx_str_set(&r->headers_out.location->key, "Location");

        if (r->args.len == 0) {
            r->headers_out.location->value = clcf->name;

        } else {
            len = clcf->name.len + 1 + r->args.len;
            p = ngx_pnalloc(r->pool, len);

            if (p == NULL) {
                ngx_http_clear_location(r);
                ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                return NGX_OK;
            }

            r->headers_out.location->value.len = len;
            r->headers_out.location->value.data = p;

            p = ngx_cpymem(p, clcf->name.data, clcf->name.len);
            *p++ = '?';
            ngx_memcpy(p, r->args.data, r->args.len);
        }

        ngx_http_finalize_request(r, NGX_HTTP_MOVED_PERMANENTLY);
        return NGX_OK;
    }

    r->phase_handler++;
    return NGX_AGAIN;
}
```

静态 location 查找：

```c
/*
 * NGX_OK       - exact or regex match
 * NGX_DONE     - auto redirect
 * NGX_AGAIN    - inclusive match
 * NGX_ERROR    - regex error
 * NGX_DECLINED - no match
 */

static ngx_int_t
ngx_http_core_find_location(ngx_http_request_t *r)
{
    ngx_int_t                  rc;
    ngx_http_core_loc_conf_t  *pclcf;
#if (NGX_PCRE)
    ngx_int_t                  n;
    ngx_uint_t                 noregex;
    ngx_http_core_loc_conf_t  *clcf, **clcfp;

    noregex = 0;
#endif

    // 首先从当前 server 块的 loc_conf 获取配置
    pclcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    rc = ngx_http_core_find_static_location(r, pclcf->static_locations);

    // 需要到子 location 中进行查找
    if (rc == NGX_AGAIN) {
#if (NGX_PCRE)
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
        noregex = clcf->noregex;
#endif
        /* look up nested locations */
        rc = ngx_http_core_find_location(r);
    }

    // NGX_OK 精确查找到
    // NGX_DONE 需要重定向
    if (rc == NGX_OK || rc == NGX_DONE) {
        return rc;
    }

    /* rc == NGX_DECLINED or rc == NGX_AGAIN in nested location */

#if (NGX_PCRE)

    if (noregex == 0 && pclcf->regex_locations) {

        for (clcfp = pclcf->regex_locations; *clcfp; clcfp++) {

            ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                           "test location: ~ \"%V\"", &(*clcfp)->name);

            n = ngx_http_regex_exec(r, (*clcfp)->regex, &r->uri);

            if (n == NGX_OK) {
                r->loc_conf = (*clcfp)->loc_conf;

                /* look up nested locations */
                rc = ngx_http_core_find_location(r);

                return (rc == NGX_ERROR) ? rc : NGX_OK;
            }

            if (n == NGX_DECLINED) {
                continue;
            }

            return NGX_ERROR;
        }
    }
#endif

    return rc;
}

/*
 * NGX_OK       - exact match
 * NGX_DONE     - auto redirect
 * NGX_AGAIN    - inclusive match
 * NGX_DECLINED - no match
 */

static ngx_int_t
ngx_http_core_find_static_location(ngx_http_request_t *r, ngx_http_location_tree_node_t *node)
{
    u_char     *uri;
    size_t      len, n;
    ngx_int_t   rc, rv;

    len = r->uri.len;
    uri = r->uri.data;
    rv = NGX_DECLINED;

    for ( ;; ) {

        if (node == NULL) {
            return rv;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "test location: \"%*s\"",
                       (size_t) node->len, node->name);
        n = (len <= (size_t) node->len) ? len : node->len;

        rc = ngx_filename_cmp(uri, node->name, n);

		// 字符串不匹配
		// 未查找到 location，继续进行查找
        if (rc != 0) {
            node = (rc < 0) ? node->left : node->right;
            continue;
        }

		// 字符串匹配，不过请求的 location 比当前比对上的 location 名长
		// 如果当前 location 包含子 location，继续查找子 location;否则查找其他 location
        if (len > (size_t) node->len) {

            if (node->inclusive) {

                r->loc_conf = node->inclusive->loc_conf;
                rv = NGX_AGAIN;

                node = node->tree;
                uri += n;
                len -= n;

                continue;
            }

            /* exact only */

            node = node->right;

            continue;
        }

		// 请求 location 与当前节点 location 名相同
        if (len == (size_t) node->len) {

            if (node->exact) {
                r->loc_conf = node->exact->loc_conf;
                return NGX_OK;

            } else {
                r->loc_conf = node->inclusive->loc_conf;
                return NGX_AGAIN;
            }
        }

        /* len < node->len */
        if (len + 1 == (size_t) node->len && node->auto_redirect) {
            r->loc_conf = (node->exact) ? node->exact->loc_conf: node->inclusive->loc_conf;
            rv = NGX_DONE;
        }

        node = node->left;
    }
}
```

