---
layout: post
title: ngx_http_limit_req_module 模块
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## 一 概述

`ngx_http_limit_req_module` 模块可以用来限制请求频率。我们要明确限制的请求对象，如何区分一个请求？可以使用请求对端地址、URL 参数或者其他头\包体信息。NGINX 使用多进程模式，如果仅在进程内实现频率限制，同一个请求被均衡到其他进程时频率限制会失效，此时需要使用共享内存同步频率信息。

## 二 模块说明

`ngx_http_limit_req_module` 介入阶段：`NGX_HTTP_PREACCESS_PHASE`。

### 1. `limit_req_zone` 指令

```nginx
# 指令
limit_req_zone key zone=name:size rate=rate;
```

可使用域：`http`
功能：设置一块共享内存用来保存 `key` 的状态参数。如果限制域的存储空间耗尽了，对于后续所有请求，服务器都会返回 503。

| 字段 |                         含义                         |                            可取值                            |
| ---- | :--------------------------------------------------: | :----------------------------------------------------------: |
| key  |        使用 key 在共享内存中保存/查询其对应值        | 变量（例如 $binary_remote_addr）、字符串（name）或者两者混合 |
| rate |              每个 key 平均访问频率限制               |         1r/s（每秒一个请求）,10r/m（每分钟十个请求）         |
| zone | 共享内存声明，name 为共享内存名，size 为共享内存大小 |                                                              |

### 2. `limit_req` 指令

```nginx
# 语法 
limit_req zone=name [burst=number] [nodelay]; 
```

可使用域：`http\server\location`


| 字段    | 含义                                                         | 可取值                         |
| ------- | ------------------------------------------------------------ | ------------------------------ |
| zone    | 引用 `limit_req_zone` 定义的共享内存                         | `limit_req_zone` 定义的 `name` |
| burst   | 突发请求数，默认值为零，设置允许超过频率限制的请求数。如果超过平均频率的请求数小于 burst，那么请求会被延时处理；如果超过平均频率的请求数大于 burst，那么会返回 503 | 数值                           |
| nodelay | 如果未设置 `nodelay`，在超过频率后会将当前请求放入定时器中，待时间到达后再进行请求处理。如果设置了 `nodelay` 则请求处理结束 | nodelay                        |

### 3. `limit_req_status`

```nginx
# 语法
limit_req_status code;# code 默认为 503
```

可使用域：`http\server\location`
功能：设置拒绝请求的响应状态码

### 4. `limit_req_log_level`

```nginx
# 语法 
limit_req_log_level info | notice | warn | error;
```

可使用域：`http\server\location`
功能：设置因为访问频率过高拒绝或延迟处理请求时记录的日志级别，`limit_req_log_level` 设置的是拒绝处理请求的日志级别，延迟处理请求的日志级别比拒绝处理请求的日志级别低一个级别。例如：`limit_req_log_level notice` 拒绝处理的日志级别为 `notice`，延迟的日志级别为 `info`。

## 三 实现

在 `ngx_http_limit_req_module` 中 `key` 使用红黑树结构存储。

### 1. 模块声明与指令定义

```c
// 指令定义
static ngx_command_t  ngx_http_limit_req_commands[] = {
    { ngx_string("limit_req_zone"),	// 定义共享内存
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE3,
      ngx_http_limit_req_zone,
     // ...
    },
    { ngx_string("limit_req"),// 定义访问限制
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE123,
      ngx_http_limit_req,
     // ...
    },
    { ngx_string("limit_req_log_level"), // 触发限制访问（拒绝/延时）时记录日志的级别
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_enum_slot,// 使用枚举设置变量配置值
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_limit_req_conf_t, limit_log_level),
      &ngx_http_limit_req_log_levels // 枚举值定义
    },
    { ngx_string("limit_req_status"), // 返回状态码
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_limit_req_conf_t, status_code),
      &ngx_http_limit_req_status_bounds // 枚举值定义
    },
      ngx_null_command
};
// 模块的上下文
static ngx_http_module_t  ngx_http_limit_req_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_limit_req_init,               /* postconfiguration 设置函数指针 */
    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */
    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */
    ngx_http_limit_req_create_conf,        /* create location configuration */
    ngx_http_limit_req_merge_conf          /* merge location configuration */
};
// 模块定义
ngx_module_t  ngx_http_limit_req_module = {
    NGX_MODULE_V1,
    &ngx_http_limit_req_module_ctx,        /* module context */
    ngx_http_limit_req_commands,           /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
	// ...
    NGX_MODULE_V1_PADDING
};
```

### 2. 共享内存初始化

```c
static char *
ngx_http_limit_req_zone(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    u_char                            *p;
    size_t                             len;
    ssize_t                            size;
    ngx_str_t                         *value, name, s;
    ngx_int_t                          rate, scale;
    ngx_uint_t                         i;
    ngx_shm_zone_t                    *shm_zone;
    ngx_http_limit_req_ctx_t          *ctx;
    ngx_http_compile_complex_value_t   ccv;

    value = cf->args->elts;

    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_limit_req_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }

    ngx_memzero(&ccv, sizeof(ngx_http_compile_complex_value_t));

    ccv.cf = cf;
    ccv.value = &value[1];
    ccv.complex_value = &ctx->key;

    if (ngx_http_compile_complex_value(&ccv) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    size = 0;
    rate = 1;
    scale = 1;
    name.len = 0;

    // 指令解析
    // 配置示例：limit_req_zone $binary_remote_addr zone=limit_ip:50m rate=10r/s;
    for (i = 2; i < cf->args->nelts; i++) {
        if (ngx_strncmp(value[i].data, "zone=", 5) == 0) {
            // name
            name.data = value[i].data + 5;
            p = (u_char *) ngx_strchr(name.data, ':');
            if (p == NULL) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "invalid zone size \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }
            name.len = p - name.data;
            // size
            s.data = p + 1;
            s.len = value[i].data + value[i].len - s.data;
            size = ngx_parse_size(&s);
            if (size == NGX_ERROR) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,"invalid zone size \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }
            // 共享内存大小不能小于 8 个 page_size
            if (size < (ssize_t) (8 * ngx_pagesize)) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "zone \"%V\" is too small", &value[i]);
                return NGX_CONF_ERROR;
            }

            continue;
        }

        if (ngx_strncmp(value[i].data, "rate=", 5) == 0) {
            len = value[i].len;
            p = value[i].data + len - 3;

            if (ngx_strncmp(p, "r/s", 3) == 0) {
                scale = 1;
                len -= 3;
            } else if (ngx_strncmp(p, "r/m", 3) == 0) {
                scale = 60;
                len -= 3;
            }

            rate = ngx_atoi(value[i].data + 5, len - 5);
            if (rate <= 0) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "invalid rate \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }

            continue;
        }

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "invalid parameter \"%V\"", &value[i]);
        return NGX_CONF_ERROR;
    }

    if (name.len == 0) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "\"%V\" must have \"zone\" parameter", &cmd->name);
        return NGX_CONF_ERROR;
    }

    ctx->rate = rate * 1000 / scale;
    // 将共享内存添加到 cycle->shared_memory 动态数组中，统一管理
    shm_zone = ngx_shared_memory_add(cf, &name, size, &ngx_http_limit_req_module);
    if (shm_zone == NULL) {
        return NGX_CONF_ERROR;
    }

    if (shm_zone->data) {
        ctx = shm_zone->data;
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "%V \"%V\" is already bound to key \"%V\"", 
                           &cmd->name, &name, &ctx->key.value);
        return NGX_CONF_ERROR;
    }
    
    // 共享内存通过 ngx_http_limit_req_init_zone 创建并初始化
    shm_zone->init = ngx_http_limit_req_init_zone;
    shm_zone->data = ctx;

    return NGX_CONF_OK;
}
```

`ngx_http_limit_req_zone` 函数并没有立即创建共享内存，仅将共享内存配置信息解析并放在 `cycle` 中统一管理。**配置解析，功能实现分开，思路非常棒，凡事都理清**。

**在调用 `ngx_http_limit_req_init_zone` 之前 NGINX 已经根据配置创建好了共享内存，在 `ngx_http_limit_req_init_zone` 函数中主要做的是如何使用这块共享内存**。NGINX 已经将共享内存申请的重复工作封装好，我们只需要将共享内存配置信息创建好，它可以替我们创建好共享内存直接使用。

```c
static ngx_int_t
ngx_http_limit_req_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_http_limit_req_ctx_t  *octx = data;

    size_t                     len;
    ngx_http_limit_req_ctx_t  *ctx;

    ctx = shm_zone->data;

    if (octx) {
        if (ctx->key.value.len != octx->key.value.len
            || ngx_strncmp(ctx->key.value.data, octx->key.value.data, ctx->key.value.len) != 0)
        {
            ngx_log_error(NGX_LOG_EMERG, shm_zone->shm.log, 0,
                          "limit_req \"%V\" uses the \"%V\" key while previously it used the \"%V\" key", 
                          &shm_zone->shm.name, &ctx->key.value, &octx->key.value);
            return NGX_ERROR;
        }
        ctx->sh = octx->sh;
        ctx->shpool = octx->shpool;

        return NGX_OK;
    }

    // 直接使用 NGINX 创建的共享内存
    ctx->shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        ctx->sh = ctx->shpool->data;
        return NGX_OK;
    }

    ctx->sh = ngx_slab_alloc(ctx->shpool, sizeof(ngx_http_limit_req_shctx_t));
    if (ctx->sh == NULL) {
        return NGX_ERROR;
    }

    ctx->shpool->data = ctx->sh;
    // 初始化红黑树
    ngx_rbtree_init(&ctx->sh->rbtree, &ctx->sh->sentinel, ngx_http_limit_req_rbtree_insert_value);
    // 初始化 LRU 队列
    ngx_queue_init(&ctx->sh->queue);

    len = sizeof(" in limit_req zone \"\"") + shm_zone->shm.name.len;

    ctx->shpool->log_ctx = ngx_slab_alloc(ctx->shpool, len);
    if (ctx->shpool->log_ctx == NULL) {
        return NGX_ERROR;
    }

    ngx_sprintf(ctx->shpool->log_ctx, " in limit_req zone \"%V\"%Z", &shm_zone->shm.name);

    ctx->shpool->log_nomem = 0;

    return NGX_OK;
}
```

### 3. 访问限制配置

`limit_req_zone` 指令用来声明共享内存，`limit_req` 指令用来在当前范围启用访问限制。`limit_req` 指令主要功能是将访问限制配置添加到`main\server\location` 级别的模块配置结构体中，`main\server\location` 级别可以有多个访问限制，通过动态数组来组织。

```c
static char *
ngx_http_limit_req(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_limit_req_conf_t  *lrcf = conf;

    ngx_int_t                    burst;
    ngx_str_t                   *value, s;
    ngx_uint_t                   i, nodelay;
    ngx_shm_zone_t              *shm_zone;
    ngx_http_limit_req_limit_t  *limit, *limits;

    value = cf->args->elts;

    shm_zone = NULL;
    burst = 0;
    nodelay = 0;
    // 配置信息解析
    for (i = 1; i < cf->args->nelts; i++) {
        if (ngx_strncmp(value[i].data, "zone=", 5) == 0) {
            s.len = value[i].len - 5;
            s.data = value[i].data + 5;
            shm_zone = ngx_shared_memory_add(cf, &s, 0, &ngx_http_limit_req_module);
            if (shm_zone == NULL) {
                return NGX_CONF_ERROR;
            }
            continue;
        }

        if (ngx_strncmp(value[i].data, "burst=", 6) == 0) {
            burst = ngx_atoi(value[i].data + 6, value[i].len - 6);
            if (burst <= 0) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "invalid burst rate \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }
            continue;
        }

        if (ngx_strcmp(value[i].data, "nodelay") == 0) {
            nodelay = 1;
            continue;
        }

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "invalid parameter \"%V\"", &value[i]);
        return NGX_CONF_ERROR;
    }
    
    // 判断是否是非法配置
    if (shm_zone == NULL) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "\"%V\" must have \"zone\" parameter", &cmd->name);
        return NGX_CONF_ERROR;
    }

    // 获得访问配置限制的动态数组
    limits = lrcf->limits.elts;

    if (limits == NULL) {
        if (ngx_array_init(&lrcf->limits, cf->pool, 1, sizeof(ngx_http_limit_req_limit_t)) != NGX_OK)
        {
            return NGX_CONF_ERROR;
        }
    }
    // 直接通过共享内存判断是否已添加相同访问限制
    for (i = 0; i < lrcf->limits.nelts; i++) {
        if (shm_zone == limits[i].shm_zone) {
            return "is duplicate";
        }
    }
    // 添加新的访问限制项
    limit = ngx_array_push(&lrcf->limits);
    if (limit == NULL) {
        return NGX_CONF_ERROR;
    }

    limit->shm_zone = shm_zone;
    limit->burst = burst * 1000;
    limit->nodelay = nodelay;

    return NGX_CONF_OK;
}
```

### 4. 共享内存查找

`ngx_http_limit_req_lookup` 用于在共享内存中查找、新增节点。当未查找到节点时会分配新的节点用以存储当前请求统计信息，并将当前节点加入 LRU 队列首部；当查找到时会将 LRU 队列中当前节点重新插入到首部，并计算请求频率以及超出值更新到节点中，根据是否超出返回相应应答。`account` 参数用来判断当前访问现在是否是当前域的最后限制，**`limit_req` 模块仅在最后一个访问限制配置中保存访问信息**。

```c
static ngx_int_t
ngx_http_limit_req_lookup(ngx_http_limit_req_limit_t *limit, ngx_uint_t hash, ngx_str_t *key, ngx_uint_t *ep, ngx_uint_t account)
{
    size_t                      size;
    ngx_int_t                   rc, excess;
    ngx_msec_t                  now;
    ngx_msec_int_t              ms;
    ngx_rbtree_node_t          *node, *sentinel;
    ngx_http_limit_req_ctx_t   *ctx;
    ngx_http_limit_req_node_t  *lr;

    now = ngx_current_msec;

    ctx = limit->shm_zone->data;

    // 红黑树查找
    node = ctx->sh->rbtree.root;
    sentinel = ctx->sh->rbtree.sentinel;
    while (node != sentinel) {
        // 递归查找左右子树
        if (hash < node->key) {
            node = node->left;
            continue;
        }
        if (hash > node->key) {
            node = node->right;
            continue;
        }

        /* hash == node->key */
        lr = (ngx_http_limit_req_node_t *) &node->color;
        rc = ngx_memn2cmp(key->data, lr->data, key->len, (size_t) lr->len);
        
        if (rc == 0) {
            // 找到节点
            
            // queue 使用 list_head 结构，所有根据当前节点就可以直接完成删除操作
            ngx_queue_remove(&lr->queue);
            ngx_queue_insert_head(&ctx->sh->queue, &lr->queue);

            ms = (ngx_msec_int_t) (now - lr->last);
            // lr->excess ：历史超出数
            // ctx->rate * ms : 根据与上次请求间隔计算出这段时间允许通过的个数
            excess = lr->excess - ctx->rate * ngx_abs(ms) / 1000 + 1000;
            if (excess < 0) {
                excess = 0;
            }

            *ep = excess;
            if ((ngx_uint_t) excess > limit->burst) {
                return NGX_BUSY;
            }
            
            // 仅在最后的访问限制中添加节点
            if (account) {
                lr->excess = excess;
                lr->last = now;
                return NGX_OK;
            }

            lr->count++; // 访问计数增加 1
            ctx->node = lr;
            return NGX_AGAIN;
        }
        // key 不相同，继续查找左右子树
        node = (rc < 0) ? node->left : node->right;
    }

    // 未找到，添加新节点
    *ep = 0;

    size = offsetof(ngx_rbtree_node_t, color)
           + offsetof(ngx_http_limit_req_node_t, data)
           + key->len;
    
    // 从 LRU 队列、红黑树中删除一或两个请求频率为零的节点
    // 1. 如果待删除的节点访问时间在一分钟内则不删除
    // 2. 如果待删除的节点请求频率大于零则不删除
    ngx_http_limit_req_expire(ctx, 1);

    node = ngx_slab_alloc_locked(ctx->shpool, size);
    if (node == NULL) {
        // 删除最旧的访问节点
        // 如果第二第三旧的节点在最近一分钟内没有访问则删除
        // 如果第二第三旧的节点访问频率等于零则删除
        ngx_http_limit_req_expire(ctx, 0);

        node = ngx_slab_alloc_locked(ctx->shpool, size);
        if (node == NULL) {
            ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "could not allocate node%s", ctx->shpool->log_ctx);
            return NGX_ERROR;
        }
    }

    node->key = hash;

    lr = (ngx_http_limit_req_node_t *) &node->color;

    lr->len = (u_short) key->len;
    lr->excess = 0;

    ngx_memcpy(lr->data, key->data, key->len);

    ngx_rbtree_insert(&ctx->sh->rbtree, node);

    ngx_queue_insert_head(&ctx->sh->queue, &lr->queue);

    if (account) {
        lr->last = now;
        lr->count = 0;
        return NGX_OK;
    }

    lr->last = 0;
    lr->count = 1; // 新节点，访问计数设置为 1

    ctx->node = lr;

    return NGX_AGAIN;
}
```

### 5. 业务处理接口

`ngx_http_limit_req_handler` 函数介入 `NGX_HTTP_PREACCESS_PHASE` 处理阶段。

```c
static ngx_int_t
ngx_http_limit_req_handler(ngx_http_request_t *r)
{
    uint32_t                     hash;
    ngx_str_t                    key;
    ngx_int_t                    rc;
    ngx_uint_t                   n, excess;
    ngx_msec_t                   delay;
    ngx_http_limit_req_ctx_t    *ctx;
    ngx_http_limit_req_conf_t   *lrcf;
    ngx_http_limit_req_limit_t  *limit, *limits;

    if (r->main->limit_req_set) {
        return NGX_DECLINED;
    }

    // 取得 location 级别的访问限制动态数组
    lrcf = ngx_http_get_module_loc_conf(r, ngx_http_limit_req_module);
    limits = lrcf->limits.elts;

    excess = 0;

    rc = NGX_DECLINED;

#if (NGX_SUPPRESS_WARN)
    limit = NULL;
#endif

    for (n = 0; n < lrcf->limits.nelts; n++) {
        limit = &limits[n];
        ctx = limit->shm_zone->data;

		// 从请求中计算 key
        if (ngx_http_complex_value(r, &ctx->key, &key) != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
        // key 长度限制
        if (key.len == 0) {
            continue;
        }
        // key 长度限制
        if (key.len > 65535) {
            ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                          "the value of the \"%V\" key is more than 65535 bytes: \"%V\"",
                          &ctx->key.value, &key);
            continue;
        }
        
        hash = ngx_crc32_short(key.data, key.len);
        ngx_shmtx_lock(&ctx->shpool->mutex);
        // 共享内存中查找 key;访问是否超限在此处判断
        // rc 有可能取值：NGX_ERROR,NGX_OK,NGX_AGAIN,NGX_BUSY
        // NGX_ERROR => 出错
        // NGX_OK	 => 未超限
        // NGX_AGAIN => 查找其他访问限制
        // NGX_BUSY	 => 访问已超限
        rc = ngx_http_limit_req_lookup(limit, hash, &key, &excess, (n == lrcf->limits.nelts - 1));
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        ngx_log_debug4(NGX_LOG_DEBUG_HTTP, r->connection->log, 0, "limit_req[%ui]: %i %ui.%03ui",
                       n, rc, excess / 1000, excess % 1000);

        if (rc != NGX_AGAIN) {
            break;
        }
    }
    // NGX_DECLINED 表示不存在访问限制，即 lrcf->limits.nelts == 0；本模块不进行处理
    if (rc == NGX_DECLINED) {
        return NGX_DECLINED;
    }

    // 设置标记，已经进行过访问限制检查
    r->main->limit_req_set = 1;

    // 请求被限制
    if (rc == NGX_BUSY || rc == NGX_ERROR) {
        if (rc == NGX_BUSY) {
            ngx_log_error(lrcf->limit_log_level, r->connection->log, 0,
                          "limiting requests, excess: %ui.%03ui by zone \"%V\"",
                          excess / 1000, excess % 1000, &limit->shm_zone->shm.name);
        }

        // 遍历所有访问限制，因为本次访问被限制，应该将更新过的节点访问计数减 1
        // 在 ngx_http_limit_req_lookup 中已经将 ctx->node 设置为当前请求节点
        while (n--) {
            ctx = limits[n].shm_zone->data;
            if (ctx->node == NULL) {
                continue;
            }
            ngx_shmtx_lock(&ctx->shpool->mutex);
            ctx->node->count--;
            ngx_shmtx_unlock(&ctx->shpool->mutex);
            ctx->node = NULL;
        }

        // 范围设置的应答码
        return lrcf->status_code;
    }

    /* rc == NGX_AGAIN || rc == NGX_OK */
    if (rc == NGX_AGAIN) {
        excess = 0;
    }

    // 更新各个访问限制的超出值以及请求计数，计算需要的最大的延时时间
    delay = ngx_http_limit_req_account(limits, n, &excess, &limit);
    if (!delay) {
        return NGX_DECLINED;
    }

    ngx_log_error(lrcf->delay_log_level, r->connection->log, 0,
                  "delaying request, excess: %ui.%03ui, by zone \"%V\"",
                  excess / 1000, excess % 1000, &limit->shm_zone->shm.name);

    // 读事件处理
    if (ngx_handle_read_event(r->connection->read, 0) != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    r->read_event_handler = ngx_http_test_reading;
    r->write_event_handler = ngx_http_limit_req_delay;
    r->connection->write->delayed = 1;
    
    // 添加延时处理定时器，delay 时间到达后调用 ngx_http_limit_req_delay 函数处理
    ngx_add_timer(r->connection->write, delay);

    return NGX_AGAIN;
}
```

超出值的计算逻辑，算法：TODO。

### 6. 延时后处理

如果请求需要延时处理会将请求放入定时器中，当定时时间到后会执行定时器中超时处理函数 `ngx_http_limit_req_delay`。

```c
static void
ngx_http_limit_req_delay(ngx_http_request_t *r)
{
    ngx_event_t  *wev;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0, "limit_req delay");

    // 有写事件，处理，直接结束请求
    wev = r->connection->write;
    if (wev->delayed) {
        if (ngx_handle_write_event(wev, 0) != NGX_OK) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        }
        return;
    }

    // 有读事件，处理，直接结束请求
    if (ngx_handle_read_event(r->connection->read, 0) != NGX_OK) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    // 回归原有处理逻辑；继续进行 HTTP 各阶段处理
    r->read_event_handler = ngx_http_block_reading;
    r->write_event_handler = ngx_http_core_run_phases;
    ngx_http_core_run_phases(r);
}
```

## 四 HTTP 应答码解释

- 204

  服务器成功处理了请求，没有返回任何内容。

- 444

  Nginx 上 HTTP 服务器扩展。服务器不向客户端返回任何信息，并关闭连接（有助于阻止恶意软件）。

- 503

  服务不可用。

## 五 总结

在使用 `ngx_http_limit_req_module` 时可以自己添加变量，变量值为请求头或包体中内容用于区分请求，直接使用官方模块实现访问频率限制。

## 六 `ngx_http_limit_conn_module` 说明

`ngx_http_limit_conn_module` 与本模块功能相似，但是 `limit_conn` 的限制对象是连接数，只需要将键的连接数保存在共享内存中就可以。不过 `limit_conn` 同样有另外一个问题，在连接建立请求时可以累计连接数，但是连接关闭时如何减少连接数呢？`limit_conn` 采用的方法是在当前请求的连接池清理函数中添加清理函数 `ngx_http_limit_conn_cleanup` 用于减少计数清理内存。

**`ngx_http_limit_conn|req_module` 模块在共享内存空间不够时均返回配置的应答码给客户端**。

## 六 参考连接

-   [NGINX 官方文档](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html)

-   [HTTP 状态码](https://zh.wikipedia.org/wiki/HTTP%E7%8A%B6%E6%80%81%E7%A0%81)

-   [网友链接同时也是公式出处](https://blog.csdn.net/huzilinitachi/article/details/79694641)
