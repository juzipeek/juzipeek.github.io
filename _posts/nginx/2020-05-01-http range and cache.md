---
layout: post
title: HTTP Range 缓存
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## HTTP Range 请求

如果 HTTP 请求的头部有 Range 头, 如 `Range: bytes=1024-2047` 表示客户端请求文件的第 1025 到第 2048 个字节内容(以 0 为索引开始). 这时服务器只会响应文件的这部分内容, 响应的状态码为 `206`, 表示返回的是响应的一部分. 如果服务器不支持 Range 请求, 仍然会返回整个文件, 这时状态码仍是 200.

**Range 请求需要服务端支持, 使用 NGINX 搭建静态页面服务方便演示**.

## 测试示例

```nginx.conf
proxy_cache_path /tmp/cache levels=1:2 keys_zone=cache:100m;  

server {
    listen 8000 default_server;

    location /doc {
        add_header Accept-Ranges bytes;
        alias /tmp/website/;
    }

    location / {
        slice             100k;
        proxy_cache       cache;
        proxy_cache_key   $uri$slice_range;
        proxy_set_header  Range $slice_range;
        proxy_cache_valid 200 206 1h;
        proxy_pass http://127.0.0.1:8000/doc/bigfile.txt;
    }
}
```

测试命令:

```bash
$curl -o /dev/null -i -v "http://127.0.0.1:8000/"
$curl -o /dev/null -i -v -r 0,10 "http://127.0.0.1:8000/"
$curl -o /dev/null -i -H "Range: bytes=0-10" -v "http://127.0.0.1:8000/"
$curl -o /dev/null -i -H "Range: bytes=0-10, 11-30" -v "http://127.0.0.1:8000/"
$curl -o /dev/null -i -H "Range: bytes=0-10, 102500-102510" -v "http://127.0.0.1:8000/"
```

## 源码阅读

nginx 官方提供 `slice` 模块用于将响应按固定大小进行切分, 单纯的切分并没有太大意义, 当切分与缓存功能配合就非常有价值: 大文件切分后可以并行使用 Range 请求多个片段提高效率.

通过 nginx 进行反向代理, 如果根据 Range 头创建缓存键是不合理的. 使用 `slice` 模块, 内容按固定区间进行切分并确定缓存键. 客户端会接收到 `Content-Range: bytes 234-639/8000` 应答头, **客户端应以 `Content-Range` 为准而非 `Range` 请求头**.

### 1. slice_range 变量获取

`slice_range` 变量用于获取"当前" `range` 的起止偏移, 类似 `bytes=0-1023` 格式. 偏移是根据 `slice` 指令配置对齐的.

```c
static ngx_int_t
ngx_http_slice_range_variable(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    u_char                     *p;
    ngx_http_slice_ctx_t       *ctx;
    ngx_http_slice_loc_conf_t  *slcf;

    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);

    // 访问 slice_range 变量并不会更新 range 的起止位置, `slice_filter` 模块在 `ngx_http_slice_header_filter` 处理中会更新 `start` 偏移.
    if (ctx == NULL) {
        if (r != r->main || r->headers_out.status) {
            v->not_found = 1;
            return NGX_OK;
        }

        slcf = ngx_http_get_module_loc_conf(r, ngx_http_slice_filter_module);

        if (slcf->size == 0) {
            v->not_found = 1;
            return NGX_OK;
        }

        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_slice_ctx_t));
        if (ctx == NULL) {
            return NGX_ERROR;
        }

        ngx_http_set_ctx(r, ctx, ngx_http_slice_filter_module);

        p = ngx_pnalloc(r->pool, sizeof("bytes=-") - 1 + 2 * NGX_OFF_T_LEN);
        if (p == NULL) {
            return NGX_ERROR;
        }

        // ngx_http_slice_get_start 函数用于从 range 请求头获取开始字节数
        // 计算出边界
        ctx->start = slcf->size * (ngx_http_slice_get_start(r) / slcf->size);

        ctx->range.data = p;
        ctx->range.len = ngx_sprintf(p, "bytes=%O-%O", ctx->start,
                                     ctx->start + (off_t) slcf->size - 1)
                         - p;
    }

    v->data = ctx->range.data;
    v->valid = 1;
    v->not_found = 0;
    v->no_cacheable = 1;
    v->len = ctx->range.len;

    return NGX_OK;
}
```

### 2. proxy_set_header

`proxy_set_header` 用于在发往上游的请求中添加请求头, **在配置解析阶段**, `proxy_set_header` 指令处理函数会在当前 `location` 配置项 `headers_source` 数组添加自定义请求头.

在创建发送到上游的请求时, 会根据自定义请求头构建请求.*除自定义请求头外 `proxy_pass` 模块会默认添加一部分请求头, 在 `ngx_http_proxy_headers` 表(开启缓存时使用 `ngx_http_proxy_cache_headers` 表)有定义*.

### 3. create request

`proxy_pass` 模块会在 `CONTENT` 阶段介入请求处理, 当开启缓存功能时会在共享内存中保存缓存键, 在磁盘文件保存缓存数据. 在 `ngx_http_upstream_init_request` 函数处理中会调用 `ngx_http_proxy_create_request` 函数创建发送到上游的请求.

### 4. ngx_http_slice_header_filter

在请求处理过程中, `header_filter` 和 `body_filter` 是在应答阶段触发, `header_filter` 会先被触发, 并且只会被触发一次. `slice` 模块会在 `header_filter` 做更新请求上下文中的 `start` 偏移, 用于下次子请求时作为 `range` 头起始偏移.

代码中判断 `ctx == NULL` 是因为 `slice` 模块有固定的使用方式(在 `proxy_pass` 中增加 `Range` 请求头, 并通过 `slice` 模块计算 `range` 偏移), 如果不按使用方式使用, `slice` 模块不处理.

```c
static ngx_int_t
ngx_http_slice_header_filter(ngx_http_request_t *r)
{
    off_t                            end;
    ngx_int_t                        rc;
    ngx_table_elt_t                 *h;
    ngx_http_slice_ctx_t            *ctx;
    ngx_http_slice_loc_conf_t       *slcf;
    ngx_http_slice_content_range_t   cr;

    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
    // ctx 为空, 之前肯定没有使用 slice_range 变量, 不需要 slice 模块介入
    if (ctx == NULL) {
        return ngx_http_next_header_filter(r);
    }

    if (r->headers_out.status != NGX_HTTP_PARTIAL_CONTENT) {
        if (r == r->main) {
            ngx_http_set_ctx(r, NULL, ngx_http_slice_filter_module);
            return ngx_http_next_header_filter(r);
        }

        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "unexpected status code %ui in slice response",
                      r->headers_out.status);
        return NGX_ERROR;
    }

    h = r->headers_out.etag;

    if (ctx->etag.len) {
        if (h == NULL
            || h->value.len != ctx->etag.len
            || ngx_strncmp(h->value.data, ctx->etag.data, ctx->etag.len)
               != 0)
        {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "etag mismatch in slice response");
            return NGX_ERROR;
        }
    }

    if (h) {
        ctx->etag = h->value;
    }

    // 解析 content-range 应答头中的应答范围
    if (ngx_http_slice_parse_content_range(r, &cr) != NGX_OK) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "invalid range in slice response");
        return NGX_ERROR;
    }

    if (cr.complete_length == -1) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "no complete length in slice response");
        return NGX_ERROR;
    }

    ngx_log_debug3(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http slice response range: %O-%O/%O",
                   cr.start, cr.end, cr.complete_length);

    slcf = ngx_http_get_module_loc_conf(r, ngx_http_slice_filter_module);

    end = ngx_min(cr.start + (off_t) slcf->size, cr.complete_length);

    if (cr.start != ctx->start || cr.end != end) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "unexpected range in slice response: %O-%O",
                      cr.start, cr.end);
        return NGX_ERROR;
    }

    // 更新 range 起始偏移
    ctx->start = end;
    ctx->active = 1;

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.status_line.len = 0;
    r->headers_out.content_length_n = cr.complete_length;
    r->headers_out.content_offset = cr.start;
    r->headers_out.content_range->hash = 0;
    r->headers_out.content_range = NULL;

    r->allow_ranges = 1;
    r->subrequest_ranges = 1;
    r->single_range = 1;

    rc = ngx_http_next_header_filter(r);

    if (r != r->main) {
        return rc;
    }

    r->preserve_body = 1;

    if (r->headers_out.status == NGX_HTTP_PARTIAL_CONTENT) {
        if (ctx->start + (off_t) slcf->size <= r->headers_out.content_offset) {
            ctx->start = slcf->size
                         * (r->headers_out.content_offset / slcf->size);
        }

        ctx->end = r->headers_out.content_offset
                   + r->headers_out.content_length_n;

    } else {
        ctx->end = cr.complete_length;
    }

    return rc;
}
```

### 5. ngx_http_slice_body_filter

在接收完上游应答后, 通过子请求方式再次发起 `range` 请求, 每个 `slice` 区间会创建一个独立子请求(顺序创建, 非并发创建). 每个子请求同样会经过 `header_filter` 和 `body_filter` 处理阶段, 在子请求的 `header_filter` 阶段会更新请求上下文中的 `start` 偏移.

子请求处理完毕后会调用父请求的 `r->write_event_handler`(是 `ngx_http_writer` 函数), 在 `ngx_http_writer` 会调用 `ngx_http_output_filter` 函数, 触发父请求的 `body_filter` 调用, 进而再次进入 `slice` 模块的 `body_filter` 处理, 继续创建子请求.

### 6. 其他说明

在 `slice` 模块, 创建子请求时使用 `NGX_HTTP_SUBREQUEST_CLONE` 调用 `ngx_http_subrequest`, 创建的子请求会从 `CONTENT` 阶段开始执行.

缓存文件创建是在对上游包体应答处理过程中实现的, 在 `ngx_http_upstream_process_request` 函数中, 会将 `proxy_pass` 临时文件文件(在 `$workdir/proxy_temp` 目录下)异动到缓存目录, 并使用缓存键作为文件名.

## 参考链接

-[Nginx的文件分片-slice模块](https://my.oschina.net/u/2539854/blog/1634878)
-[尝鲜：Nginx-1.9.8 推出的切片模块](https://pureage.info/2015/12/10/nginx-slice-module.html)
