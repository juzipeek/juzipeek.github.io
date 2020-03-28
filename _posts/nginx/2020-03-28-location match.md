---

layout: post
title: Nginx 路由匹配
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description:

---


## 一 概述

在看路由匹配树构建与匹配代码时发现有 `auto_redirect` 类型 `location`, 印象中并没有此种类型配置, 梳理出文档加深记忆.

## 二 路由匹配

路由匹配分成精确/前缀匹配和正则匹配两个阶段, 精确/前缀匹配是通过遍历 `server` 下的二叉树实现的, 正则匹配是通过遍历 `server` 下的正则匹配数组实现的.

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
    noregex = 0;
    pclcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    // 遍历匹配树
    rc = ngx_http_core_find_static_location(r, pclcf->static_locations);

    // 前缀匹配
    if (rc == NGX_AGAIN) {
        // 在递归查找过程中会修改 r->loc_conf, 先保存当前层级匹配的前缀匹配是否允许正则匹配
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

        noregex = clcf->noregex;

        /* look up nested locations */
        rc = ngx_http_core_find_location(r);
    }

    if (rc == NGX_OK || rc == NGX_DONE) {
        return rc;
    }

    /* rc == NGX_DECLINED or rc == NGX_AGAIN in nested location */

    //  当前层级启用进行正则匹配
    if (noregex == 0 && pclcf->regex_locations) {
        // 遍历正则匹配数组
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

    return rc;
}
```

函数 `ngx_http_core_find_static_location` 实现二叉树的查找, 查找过程比较简单, 但是函数会有 `NGX_DONE` ('auto redirect' 自动重定向)类型返回值. 什么是自动重定向, 又如何实现自动重定向?

## 三 自动重定向

在 location 指令处理函数中并没有标记 `auto_redirect` 类型的路由, 在 nginx 源码中只在 `ngx_http_proxy_module`, `ngx_http_fastcgi_module` 等转发模块中有设置. 例如在 `ngx_http_proxy_module` 模块的 `proxy_pass` 指令处理函数中, 如果 `location` 路径以 `/` 结尾, 那么当前 `location` 会被标记为 `auto_redirect`:

```c
static char *
ngx_http_proxy_pass(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    // 省略无关代码
    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_proxy_handler;

    if (clcf->name.len && clcf->name.data[clcf->name.len - 1] == '/') {
        clcf->auto_redirect = 1;
    }

    // 省略无关代码
}
```

`auto_redirect` 会有如下作用: **在请求匹配中, 如果请求路径匹配了 `auto_redirect` 类型, nginx 会直接返回 301 应答**. 如果有如下配置

```nginx.conf
location /case/ {
    proxy_pass http://backend;
}
```

使用 `/case` 路径请求会触发 301 重定向:

```bash
$curl -i "http://localhost:8000/case"
HTTP/1.1 301 Moved Permanently
Server: openresty/1.15.8.2
Date: Sat, 28 Mar 2020 15:17:44 GMT
Content-Type: text/html
Content-Length: 175
Location: http://localhost:8000/case/
Connection: keep-alive

# 带参数请求
$curl -i "http://localhost:8000/case?n=1"
HTTP/1.1 301 Moved Permanently
Server: openresty/1.15.8.2
Date: Sat, 28 Mar 2020 15:18:25 GMT
Content-Type: text/html
Content-Length: 175
Location: http://localhost:8000/case/?n=1
Connection: keep-alive
```

如果使用 `/case/` 则能正常转发到后端.

`auto_redirect` 触发需要的条件:

- location 必须启用 `proxy_pass`, `fastcgi_pass` 等
- location 配置路径必须以 `/` 结尾
- 请求路径与配置路径差一个 `/` 字符(如果有另外一个 `/case` 路径配置, 会匹配 `/case` 路径, 而不会触发 301)

个人认为 `auto_redirect` 存在是一种折中的选择, 因为用户配置的路径是 `/case/` 而请求路径是 `/case`, 这是用户自己配置的错误, nginx 尝试修正错误.
