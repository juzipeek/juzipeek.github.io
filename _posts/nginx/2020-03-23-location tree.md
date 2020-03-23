---
layout: post
title: Nginx 匹配树构建
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description:
---

## 一 概述

`location` 是 nginx 的核心指令, 在 `ngx_http_block` 处理 `http` 指令过程中, 当所有 http 模块解析完配置后会遍历所有 `server`(保存在 `cmcf->servers` 中)建立匹配匹配树. 配置匹配树构建需要两个步骤: 初始化(`ngx_http_init_locations`)和构建匹配树(`ngx_http_init_static_location_trees`).

## 二 匹配树预处理

匹配树预处理是将 location 队列中的 location 进行**分类,排序**. 处理结束后会在移除队列中的无名 location, 命名 location 存储在 `cscf->named_locations` 中,正则 location 存储在 `clcf->regex_locations` 中. **队列中仅保存绝对匹配和前缀匹配 location(并且是排序好的)**.

```c
static ngx_int_t
ngx_http_init_locations(ngx_conf_t *cf, ngx_http_core_srv_conf_t *cscf,
    ngx_http_core_loc_conf_t *pclcf)
{
    ngx_uint_t                   n;
    ngx_queue_t                 *q, *locations, *named, tail;
    ngx_http_core_loc_conf_t    *clcf;
    ngx_http_location_queue_t   *lq;
    ngx_http_core_loc_conf_t   **clcfp;
#if (NGX_PCRE)
    ngx_uint_t                   r;
    ngx_queue_t                 *regex;
#endif

    locations = pclcf->locations;

    if (locations == NULL) {
        return NGX_OK;
    }

    // 将当前层级下所有 location 排序
    // 排序后顺序为: 精确匹配-前缀匹配-正则匹配-命名 location-无名 location. 精确匹配在前, 无名 location 最后.
    ngx_queue_sort(locations, ngx_http_cmp_locations);

    named = NULL;
    n = 0;
#if (NGX_PCRE)
    regex = NULL;
    r = 0;
#endif

    for (q = ngx_queue_head(locations);
         q != ngx_queue_sentinel(locations);
         q = ngx_queue_next(q))
    {
        lq = (ngx_http_location_queue_t *) q;

        clcf = lq->exact ? lq->exact : lq->inclusive;

        if (ngx_http_init_locations(cf, NULL, clcf) != NGX_OK) {
            return NGX_ERROR;
        }

#if (NGX_PCRE)

        if (clcf->regex) {
            r++;
            // 因为已经排序, 只需要用 regex 变量存储第一个节点, r 指示有多少个正则匹配节点即可
            if (regex == NULL) {
                regex = q;
            }

            continue;
        }

#endif

        if (clcf->named) {
            n++;
            // 因为已经排序, 只需要用 regex 变量存储第一个节点, r 指示有多少个正则匹配节点即可
            if (named == NULL) {
                named = q;
            }

            continue;
        }

        // 无名 location
        if (clcf->noname) {
            break;
        }
    }

    // 有两种情况会跳出上面的循环: 遍历结束; 遇到无名 location.
    // 遍历结束时不需要将 q 从队列中移除; 遇到无名 location 时将其从队列移除.
    if (q != ngx_queue_sentinel(locations)) {
        ngx_queue_split(locations, q, &tail);
    }

    // 将所有命名 location 单独存储
    if (named) {
        clcfp = ngx_palloc(cf->pool,
                           (n + 1) * sizeof(ngx_http_core_loc_conf_t *));
        if (clcfp == NULL) {
            return NGX_ERROR;
        }

        // 命名 location 保存在 server 层级的 named_locations 内
        cscf->named_locations = clcfp;

        for (q = named;
             q != ngx_queue_sentinel(locations);
             q = ngx_queue_next(q))
        {
            lq = (ngx_http_location_queue_t *) q;

            *(clcfp++) = lq->exact;
        }

        *clcfp = NULL;

        // 将命名 location 移除
        ngx_queue_split(locations, named, &tail);
    }

#if (NGX_PCRE)

    // 将所有正则 location 单独存储
    if (regex) {

        clcfp = ngx_palloc(cf->pool,
                           (r + 1) * sizeof(ngx_http_core_loc_conf_t *));
        if (clcfp == NULL) {
            return NGX_ERROR;
        }

        // 正则 location 保存在上层 location 的 regex_locations 内
        pclcf->regex_locations = clcfp;

        for (q = regex;
             q != ngx_queue_sentinel(locations);
             q = ngx_queue_next(q))
        {
            lq = (ngx_http_location_queue_t *) q;

            *(clcfp++) = lq->exact;
        }

        *clcfp = NULL;

        // 将正则 location 移除
        ngx_queue_split(locations, regex, &tail);
    }

#endif

    return NGX_OK;
}
```

## 三 匹配树构建

### 1. 匹配树预处理 - 合并前缀匹配与精确匹配

前缀匹配和精确匹配 location 是否应该共存? 在 nginx 中支持前缀匹配与精确匹配相同的情况, 即可以同时存在 `location = /a` 与 `location /a`. 但是如果同时存在两个相同的精确匹配或两个相同的前缀匹配是错误的. 函数 `ngx_http_join_exact_locations` 有两个功能: 用于判断是否有两个相同的前缀或精确匹配 location; 将相同的 location 合并到精确匹配 location 中.

```c
static ngx_int_t
ngx_http_join_exact_locations(ngx_conf_t *cf, ngx_queue_t *locations)
{
    ngx_queue_t                *q, *x;
    ngx_http_location_queue_t  *lq, *lx;

    q = ngx_queue_head(locations);

    while (q != ngx_queue_last(locations)) {

        x = ngx_queue_next(q);

        lq = (ngx_http_location_queue_t *) q;
        lx = (ngx_http_location_queue_t *) x;

        if (lq->name->len == lx->name->len
            && ngx_filename_cmp(lq->name->data, lx->name->data, lx->name->len)
               == 0)
        {
            // 两个相同的精确匹配或前缀匹配
            if ((lq->exact && lx->exact) || (lq->inclusive && lx->inclusive)) {
                ngx_log_error(NGX_LOG_EMERG, cf->log, 0,
                              "duplicate location \"%V\" in %s:%ui",
                              lx->name, lx->file_name, lx->line);

                return NGX_ERROR;
            }

            // 删除前缀匹配, 合并到精确匹配中
            lq->inclusive = lx->inclusive;

            ngx_queue_remove(x);

            continue;
        }

        q = ngx_queue_next(q);
    }

    return NGX_OK;
}
```

### 2. 匹配树预处理 - 创建构建匹配树用的 list

函数 `ngx_http_create_locations_list` 用来将 `locations` 队列中有相同前缀的前缀匹配 `location` 放在最短前缀的 `list` 成员中, 用于后续构建二叉树使用. 此过程是递归过程.

```c
static void
ngx_http_create_locations_list(ngx_queue_t *locations, ngx_queue_t *q)
{
    u_char                     *name;
    size_t                      len;
    ngx_queue_t                *x, tail;
    ngx_http_location_queue_t  *lq, *lx;

    // 递归结束依据
    if (q == ngx_queue_last(locations)) {
        return;
    }

    lq = (ngx_http_location_queue_t *) q;

    // 跳过非前缀匹配
    if (lq->inclusive == NULL) {
        ngx_http_create_locations_list(locations, ngx_queue_next(q));
        return;
    }

    len = lq->name->len;
    name = lq->name->data;

    // 找到最后一个具有 q 节点名前缀的节点的 next 节点 x
    for (x = ngx_queue_next(q);
         x != ngx_queue_sentinel(locations);
         x = ngx_queue_next(x))
    {
        lx = (ngx_http_location_queue_t *) x;

        // len > lx->name->len 是注意点, 如果 lx->name->len < len 在进行比较时会出现 coredump
        if (len > lx->name->len
            || ngx_filename_cmp(name, lx->name->data, len) != 0)
        {
            break;
        }
    }

    q = ngx_queue_next(q);
    // 未找到具有 q 节点名前缀的节点, 递归处理 q 的下一个节点
    if (q == x) {
        ngx_http_create_locations_list(locations, x);
        return;
    }

    // 将原 locatins 队列以 q 为节点分成两个队列, q 作为新队列的头
    // tail 是临时变量, 在调用 ngx_queue_add 后, tail 会从队列中移除
    ngx_queue_split(locations, q, &tail);
    ngx_queue_add(&lq->list, &tail);

    // x == ngx_queue_sentinel(locations) 说明已经遍历到队列尾, 现在对拆分出的队列递归处理
    if (x == ngx_queue_sentinel(locations)) {
        ngx_http_create_locations_list(&lq->list, ngx_queue_head(&lq->list));
        return;
    }

    // x 非队列尾, 需要将 x 及其后续合并到原 locations 队列
    ngx_queue_split(&lq->list, x, &tail);
    ngx_queue_add(locations, &tail);

    // 分别对两个拆分出来的队列进行处理, 先递归, 再遍历
    ngx_http_create_locations_list(&lq->list, ngx_queue_head(&lq->list));

    ngx_http_create_locations_list(locations, x);
}
```

### 3. 二叉匹配树构建

nginx 中的匹配树应该是三叉树, 有左右子树与当前节点的子树.

```c
static ngx_http_location_tree_node_t *
ngx_http_create_locations_tree(ngx_conf_t *cf, ngx_queue_t *locations,
    size_t prefix)
{
    size_t                          len;
    ngx_queue_t                    *q, tail;
    ngx_http_location_queue_t      *lq;
    ngx_http_location_tree_node_t  *node;

    // 中间节点作为根
    q = ngx_queue_middle(locations);

    lq = (ngx_http_location_queue_t *) q;
    len = lq->name->len - prefix;

    node = ngx_palloc(cf->pool, offsetof(ngx_http_location_tree_node_t, name) + len);
    if (node == NULL) {
        return NULL;
    }

    // 根节点
    node->left = NULL;
    node->right = NULL;
    node->tree = NULL;
    node->exact = lq->exact;
    node->inclusive = lq->inclusive;

    node->auto_redirect = (u_char) ((lq->exact && lq->exact->auto_redirect)
                           || (lq->inclusive && lq->inclusive->auto_redirect));

    // 当前节点的名; 仅保存于前缀不重复的部分
    node->len = (u_char) len;
    ngx_memcpy(node->name, &lq->name->data[prefix], len);

    ngx_queue_split(locations, q, &tail);

    if (ngx_queue_empty(locations)) {
        /*
         * ngx_queue_split() insures that if left part is empty,
         * then right one is empty too
         */
        goto inclusive;
    }

    // 左子树; 递归调用; 注意, 此时实参 prefix 为 prefix
    node->left = ngx_http_create_locations_tree(cf, locations, prefix);
    if (node->left == NULL) {
        return NULL;
    }

    // 右子树不包含根节点
    ngx_queue_remove(q);

    if (ngx_queue_empty(&tail)) {
        goto inclusive;
    }

    // 右子树; 递归调用; 注意, 此时实参 prefix 为 prefix
    node->right = ngx_http_create_locations_tree(cf, &tail, prefix);
    if (node->right == NULL) {
        return NULL;
    }

inclusive:

    if (ngx_queue_empty(&lq->list)) {
        return node;
    }

    // 对根节点包含的节点递归处理; 注意, 此时实参 prefix 为 prefix + len
    node->tree = ngx_http_create_locations_tree(cf, &lq->list, prefix + len);
    if (node->tree == NULL) {
        return NULL;
    }

    return node;
}
```

### 4. 总结

在匹配树构建过程中, nginx 会遍历所有的 `server` 进行前三个步骤构建匹配树, 每个步骤又是递归调用.

使用 `ngx_http_init_static_location_trees` 函数处理 server 内的 locations.

```c
static ngx_int_t
ngx_http_init_static_location_trees(ngx_conf_t *cf,
    ngx_http_core_loc_conf_t *pclcf)
{
    ngx_queue_t                *q, *locations;
    ngx_http_core_loc_conf_t   *clcf;
    ngx_http_location_queue_t  *lq;

    locations = pclcf->locations;

    // 递归调用, 结束判断依据
    if (locations == NULL) {
        return NGX_OK;
    }

    // 递归调用, 结束判断依据
    if (ngx_queue_empty(locations)) {
        return NGX_OK;
    }

    // 遍历第一层级的 location, 递归调用 ngx_http_init_static_location_trees 函数
    for (q = ngx_queue_head(locations);
         q != ngx_queue_sentinel(locations);
         q = ngx_queue_next(q))
    {
        lq = (ngx_http_location_queue_t *) q;

        clcf = lq->exact ? lq->exact : lq->inclusive;

        if (ngx_http_init_static_location_trees(cf, clcf) != NGX_OK) {
            return NGX_ERROR;
        }
    }

    // 对第一层级 location 进行步骤 a 处理
    if (ngx_http_join_exact_locations(cf, locations) != NGX_OK) {
        return NGX_ERROR;
    }

    // 对第一层级 location 进行步骤 b 处理
    ngx_http_create_locations_list(locations, ngx_queue_head(locations));

    // 对第一层级 location 进行步骤 c 处理
    pclcf->static_locations = ngx_http_create_locations_tree(cf, locations, 0);
    if (pclcf->static_locations == NULL) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

## 四 参考链接

- [Nginx 源代码笔记 - URI 匹配](https://ialloc.org/blog/ngx-notes-http-location/)
