---
layout: post
title: ngx_http_dyups_module 模块
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## 一 功能

`ngx_http_dyups_module` 模块能够在不进行 `reload` 的情况下动态更新 `upstream` 配置。

## 二 指令

### 1. dyups_interface

```
Syntax: dyups_interface
Default: `none`
Context: `loc`
```

使用当前 `location` 处理 `upstream{}` 的查看、添加、删除操作。

### 2. dyups_read_msg_timeout

```
Syntax: dyups_read_msg_timeout `time`
Default: `1s`
Context: `main`
```

`worker` 进程读取共享内存中操作指令的间隔时间。`dyups` 模块中每个 `worker` 收到外部指令后需要将指令同步到共享内存，其他 `worker` 将指令处理一遍，同步各个 `worker` 间的状态。

### 3. dyups_shm_zone_size

```
Syntax: dyups_shm_zone_size `size`
Default: `2MB`
Context: `main`
```

`dyups` 模块使用共享内存保存指令，`dyups_shm_zone_size` 指令设置共享内存大小。

### 4. dyups_trylock

```
Syntax: dyups_trylock `on | off`
Default: `off`
Context: `main`
```

### 5. dyups_read_msg_log

```
Syntax: dyups_read_msg_log `on | off`
Default: `off`
Context: `main`
```

控制 `dyups` 模块执行指令是否输出日志。

## 三 调用接口

当使用 `dyups_interface` 配置 `location` 处理指令后，当前 `location` 可以处理 `restful` 类型的指令。

### GET 请求

#### 1. 查看主机详细信息

路径：`/detail`
输出所有的 `upstream{}` 以及其中的 `server`。

#### 2. 查看所有 `upstrem{}`

路径： `/list`
输出所有的 `upstream{}`。

#### 3. 查看单个 `upstream` 信息

路径：`/upstream/name`
根据 `upstream{}` 的名字查找其配置，输出其中的所有 `server`。

### POST 请求

#### 1. 更新 `upstream` 中 `server` 列表

路径：`/upstream/name`
更新 `name` 指定的 `upstream{}` 中的 `server` 列表。会创建一个新的 `upstream{}` 配置，使用 `body` 中指定的 `server`。`body` 内数据格式为：

```
server ip:port;
```

### DELETE

#### 1. 删除某个 `upstream`

路径：`/upstream/name`
删除由 `name` 指定的 `upstream{}`。会将 `upstream{}` 的 `dyups` 配置标记为 `deleting`，待引用计数为零时标记为 `deleted`，之后就可以重用共享内存。

## 四 安装测试

### 1. 安装

```bash
cd nginx-1.17.2

./configure \
--add-module=/opt/src/tengine-2.3.1/modules/ngx_http_upstream_dyups_module

make
make install
```

### 2. 配置

```nginx.conf
# vi $NGINX_PATH/conf/dyups/dyups.conf

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    dyups_upstream_conf  dyups_upstream.conf;

    include dyups_upstream.conf;

    server {
        listen   8080;
        # location 会根据 host 变量做代理，默认 host 变量会从请求域名、从请求头 host 中取值
        location / {
            proxy_pass http://$host;
        }
    }

    server {
        listen 8088;
        location / {
            return 200 "8088";
        }
    }

    server {
        listen 8089;
        location / {
            return 200 "8089";
        }
    }

    server {
        listen 8081;
        # dyups 操作接口
        location / {
            dyups_interface;
        }
    }

}
```

```nginx.conf
# vi $NGINX_PATH/conf/dyup/dyup_upstream.conf
upstream host1 {
    server 127.0.0.1:8088;
}

upstream host2 {
    server 127.0.0.1:8089;
}
```


### 3. 测试命令

```bash
sudo /usr/local/nginx/sbin/nginx  -c /usr/local/nginx/conf/dyups/dyups.conf

# 设置 host 头为 host1 或 host2，使用 host 变量做反向代理
$curl -H "host: host1" 127.0.0.1:8080
8088
$curl -H "host: host2" 127.0.0.1:8080
8089

# 查看所有 upstream{} 的详细信息
$curl 127.0.0.1:8081/detail
host1
server 127.0.0.1:8088 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

host2
server 127.0.0.1:8089 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

# 动态添加 upstream{}
# 添加 dyhost upstream 块，里面有两个 server
curl -d "server 127.0.0.1:8089;server 127.0.0.1:8088;" 127.0.0.1:8081/upstream/dyhost

# 再次查看所有 upstream{} 的详细信息
$curl 127.0.0.1:8081/detail
host1
server 127.0.0.1:8088 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

host2
server 127.0.0.1:8089 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

dyhost
server 127.0.0.1:8089 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0
server 127.0.0.1:8088 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

# 访问 dyhost 后端
$curl -H "host: dyhost" 127.0.0.1:8080
8089
$curl -H "host: dyhost" 127.0.0.1:8080
8088

# 更新 dyhost upstream{}
$curl -d "server 127.0.0.1:8089;" 127.0.0.1:8081/upstream/dyhost

# 再次查看所有 upstream{} 的详细信息
$curl 127.0.0.1:8081/detail
host1
server 127.0.0.1:8088 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

host2
server 127.0.0.1:8089 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0

dyhost
server 127.0.0.1:8089 weight=1 max_conns=0 max_fails=1 fail_timeout=10 backup=0 down=0
```

## 五 实现

`dyups` 模块提供 `dyups_interface` 指令将 `location` 做为 `dyups` 对外接口，提供对 `upstream` 的查看、新增、删除管理功能。在 `dyups` 内部，使用定时器、基于共享内存的消息队列实现 `upstream` 的同步。

全局变量 `ngx_http_dyups_api_enable` 用来标识当前进程是否启用 `dyups` 功能。`ngx_http_dyups_deleted_upstream` 是 `postconfig` 阶段创建的冗余配置，在执行删除指令时会将 `upstream` 的配置更新为 `ngx_http_dyups_deleted_upstream`，非共享内存。`ngx_http_dyups_shm_generation` 用来作为 `dyups` 的代，在共享初始化时使用。每次进行 `reload` 操作，`ngx_http_dyups_shm_generation` 会进行累加，`reload` 创建的进程会使用当前“代”的共享内存进行处理。`ngx_dyups_global_ctx` 作为全局配置用来存储 `slab`、共享内存配置、消息处理定时器，在共享内存初始化、进程初始化时进行初始化。

### 1. 共享内存

在 `dyups` 代中并没有对某个"代"共享内存的清理工作（`reload` 时会释放内存），其实观察共享内存的初始化回调可以发现 `dyups` 并没有在共享内存上创建附加的管理信息， `ngx_dyups_global_ctx` 是进程中的变量，通过 `shpool` 使用共享内存。在共享内存中除了 `sh` 之外，只有消息队列中的消息。在消息被销毁是会将内存归还共享内存。

```c
// 共享内存初始化函数
static ngx_int_t
ngx_http_dyups_init_shm_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_slab_pool_t    *shpool;
    ngx_dyups_shctx_t  *sh;
    // 直接引用共享内存作为 slab
    shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;
    // 进程状态数组
    sh = ngx_slab_alloc(shpool, sizeof(ngx_dyups_shctx_t));
    if (sh == NULL) {
        return NGX_ERROR;
    }

    ngx_dyups_global_ctx.sh = sh;
    ngx_dyups_global_ctx.shpool = shpool;

    ngx_queue_init(&sh->msg_queue);

    sh->version = 0;
    sh->status = NULL;

    return NGX_OK;
}

// 消息销毁
static void
ngx_dyups_destroy_msg(ngx_slab_pool_t *shpool, ngx_dyups_msg_t *msg)
{
    if (msg->pid) {
        ngx_slab_free_locked(shpool, msg->pid);
    }

    if (msg->name.data) {
        ngx_slab_free_locked(shpool, msg->name.data);
    }

    if (msg->content.data) {
        ngx_slab_free_locked(shpool, msg->content.data);
    }

    ngx_slab_free_locked(shpool, msg);
}
```

### 2. `list/detail/upstream` 请求处理

`list/detail/upstram` 用来查看 `dyups` 中 `upstream`、`server` 的信息。在调用指令时先进行消息处理，待状态同步后才进行请求处理。

### 3. 删除 `upstream` 请求处理

删除操作通过 `ngx_dyups_delete_upstream` 函数实现，函数流程：
```
---------------
| shpool 加锁 |
---------------
      |
      V
-------------------
| 处理消息队列中消息 |
-------------------
      |
      V
-------------------
| 执行 delete 操作 |
-------------------
      |
      V
---------------------------
| 发送 delete 指令到消息队列 |
---------------------------
      |
      V
------------------
|    处理结束     |
------------------
```

删除操作只是将当前 `upstream` 的配置（`ngx_http_upstream_srv_conf_t`）设置为一个无存活 `server` 的配置。如果开启健康检查，会调用健康检查模块的函数，将原先配置中的 `server` 从健康检查中删除。
在发送消息时，消息发送方会将消息的进程数组中第一个槽标记为当前进程 `pid`，可以通过此值寻找消息发送方。

在执行删除 `upstream` 指令时并未立即删除 `upstream` 的内存配置，只是将其标记为 `NGX_DYUPS_DELETING` 状态。这是因为当前内存可能被请求使用，只有在当前内存的引用计数为零时才会删除。

```c
// upstream 查找
static ngx_http_dyups_srv_conf_t *
ngx_dyups_find_upstream(ngx_str_t *name, ngx_int_t *idx)
{
    ngx_uint_t                      i;
    ngx_http_dyups_srv_conf_t      *duscfs, *duscf, *duscf_del;
    ngx_http_dyups_main_conf_t     *dumcf;
    ngx_http_upstream_srv_conf_t   *uscf;

    dumcf = ngx_http_cycle_get_module_main_conf(ngx_cycle,
                                                ngx_http_dyups_module);
    *idx = -1;
    duscf_del = NULL;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ngx_cycle->log, 0,
                   "[dyups] find dynamic upstream");

    duscfs = dumcf->dy_upstreams.elts;
    for (i = 0; i < dumcf->dy_upstreams.nelts; i++) {

        duscf = &duscfs[i];
        if (!duscf->dynamic) {
            continue;
        }

        // 对 NGX_DYUPS_DELETING 状态的 upstream 进行处理
        if (duscf->deleted == NGX_DYUPS_DELETING) {

            ngx_log_error(NGX_LOG_DEBUG, ngx_cycle->log, 0,
                          "[dyups] find upstream idx: %ui ref: %ui "
                          "on %V deleting",
                          i, *(duscf->ref), &duscf->upstream->host);

            // 如果引用计数变为零，则标记为 NGX_DYUPS_DELETED 状态
            if (*(duscf->ref) == 0) {
                ngx_log_error(NGX_LOG_INFO, ngx_cycle->log, 0,
                              "[dyups] free dynamic upstream in find upstream"
                              " %ui", duscf->idx);

                duscf->deleted = NGX_DYUPS_DELETED;

                // 释放内存
                if (duscf->pool) {
                    ngx_destroy_pool(duscf->pool);
                    duscf->pool = NULL;
                }
            }
        }
				// NGX_DYUPS_DELETING 状态的不应该再使用
        if (duscf->deleted == NGX_DYUPS_DELETING) {
            continue;
        }
				// 如果找不到则使用 NGX_DYUPS_DELETED 状态的配置
        if (duscf->deleted == NGX_DYUPS_DELETED) {
            *idx = i;
            duscf_del = duscf;
            continue;
        }

        uscf = duscf->upstream;
				// 不相同，继续
        if (uscf->host.len != name->len
            || ngx_strncasecmp(uscf->host.data, name->data, uscf->host.len)
               != 0)
        {
            continue;
        }

        *idx = i;

        return duscf;
    }

    return duscf_del;
}
```

### 4. 更新 `upstream` 请求处理

更新操作通过 `ngx_http_dyups_body_handler` 函数实现，因为更新操作需要根据包体中的信息进行更新，因此首先要将请求包体信息读出。函数流程：
```
---------------
| 读取请求包体 |
---------------
      |
      V
---------------
| 执行更新操作  |
---------------
      |
      V
---------------
| shpool 加锁 |
---------------
      |
      V
-------------------
| 处理消息队列中消息 |
-------------------
      |
      V
---------------------------------------------------
| 使用假的 upstream 名进行更新操作，验证请求数据是否正确 |
---------------------------------------------------
      |
      V
--------------
| 执行更新操作 |
--------------
      |
      V
---------------------------
| 删除旧的 upstream 配置 |
---------------------------
      |
      V
---------------------------
| 创建新的 upstream 配置 |
---------------------------
      |
      V
---------------------------
| 包体中 server 配置创建 |
---------------------------
      |
      V
----------------------------------------------
| 对新创建的 upstream 进行初始化                |
| 遍历所有的 HTTP 模块调用 create_srv_conf 函数 |
----------------------------------------------
      |
      V
---------------------------
| 发送 ADD 指令到消息队列 |
---------------------------
      |
      V
------------------
|    处理结束     |
------------------
```

### 5. 消息队列

基于共享内存的消息队列在共享内存初始化回调中创建，此时消息队列为空，在处理删除、新增指令时会向消息队列新增消息，消息被 `worker` 处理过之后会增加处理次数，如果次数达到 `worker_process` 数，则消息会被删除。消息处理锁：`shpool->mutex`。

在 `worker` 中，当消息处理定时器触发时会遍历消息队列，对其中的消息进行处理。每个 `worker` 会处理当前"代"中的所有消息，如果消息已经被所有 `worker` 处理过，会删除消息。

在进程退出时销毁所有动态配置的 `pool`（基于内存，非共享内存），共享内存并未清理。**这是因为创建的 `timer` 是不可取消事件，在 `worker` 退出时会阻止进程退出。在消息处理定时器触发，消息处理函数在将所有消息处理完毕后会添加定时器时，此时会判断当前进程的状态，如果进程为退出状态则不添加定时器，此时进程才会退出，保证了正常状况下共享内存能够被释放。**

### 6. `upstream` 引用计数

为了保证 `upstream` 内存能够正确回收同时又不会影响现有请求，`dyups` 模块使用引用计数机制在使用 `upstream` 时增加 `upstream` 的引用计数，在连接上注册清理函数，当连接关闭时引用计数减少。

`dyups` 模块在向 `upstream` 添加 `server` 时会对 `peer.init` 函数进行封装，每次成功建立连接后增加 `upstream` 的引用计数：

```c
// 向 upstream 增加 server
static ngx_int_t
ngx_dyups_add_server(ngx_http_dyups_srv_conf_t *duscf, ngx_buf_t *buf)
{
    ngx_conf_t                           cf;
    ngx_http_upstream_init_pt            init;
    ngx_http_upstream_srv_conf_t        *uscf;
    ngx_http_dyups_upstream_srv_conf_t  *dscf;

    uscf = duscf->upstream;

    if (uscf->servers == NULL) {
        uscf->servers = ngx_array_create(duscf->pool, 4,
                                         sizeof(ngx_http_upstream_server_t));
        if (uscf->servers == NULL) {
            return NGX_ERROR;
        }
    }

    ngx_memzero(&cf, sizeof(ngx_conf_t));
    cf.name = "dyups_init_module_conf";
    cf.pool = duscf->pool;
    cf.cycle = (ngx_cycle_t *) ngx_cycle;
    cf.module_type = NGX_HTTP_MODULE;
    cf.cmd_type = NGX_HTTP_UPS_CONF;
    cf.log = ngx_cycle->log;
    cf.ctx = duscf->ctx;
    cf.args = ngx_array_create(duscf->pool, 10, sizeof(ngx_str_t));
    if (cf.args == NULL) {
        return NGX_ERROR;
    }

    if (ngx_dyups_parse_upstream(&cf, buf) != NGX_CONF_OK) {
        return NGX_ERROR;
    }

    ngx_memzero(&cf, sizeof(ngx_conf_t));
    cf.name = "dyups_init_upstream";
    cf.cycle = (ngx_cycle_t *) ngx_cycle;
    cf.pool = duscf->pool;
    cf.module_type = NGX_HTTP_MODULE;
    cf.cmd_type = NGX_HTTP_MAIN_CONF;
    cf.log = ngx_cycle->log;
    cf.ctx = duscf->ctx;

    // 对新创建的 upstream{} 进行 init 操作
    init = uscf->peer.init_upstream ? uscf->peer.init_upstream:
        ngx_http_upstream_init_round_robin;

    if (init(&cf, uscf) != NGX_OK) {
        return NGX_ERROR;
    }

#if (T_NGX_HTTP_UPSTREAM_RANDOM)
    {

    ngx_http_upstream_rr_peers_t        *peers, *backup;

    /* add init_number initialization */

    peers = uscf->peer.data;
    peers->init_number = ngx_random() % peers->number;
    backup = peers->next;

    if (backup) {
        backup->init_number = ngx_random() % backup->number;
    }

    }
#endif

    dscf = uscf->srv_conf[ngx_http_dyups_module.ctx_index];
  	// 保存原始的 peer.init 回调
    dscf->init = uscf->peer.init;

    // 对 peer.init 阶段进行封装，连接建立时增加 upstream 引用计数
    uscf->peer.init = ngx_http_dyups_init_peer;

    return NGX_OK;
}
```

引用计数增加操作：

```c
// 对原始 upstream 操作封装
static ngx_int_t
ngx_http_dyups_init_peer(ngx_http_request_t *r,
    ngx_http_upstream_srv_conf_t *us)
{
    ngx_int_t                            rc;
    ngx_pool_cleanup_t                  *cln;
    ngx_http_dyups_ctx_t                *ctx;
    ngx_http_dyups_upstream_srv_conf_t  *dscf;

    dscf = us->srv_conf[ngx_http_dyups_module.ctx_index];

    rc = dscf->init(r, us);

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ngx_cycle->log, 0,
                   "[dyups] dynamic upstream init peer: %i",
                   rc);

    if (rc != NGX_OK) {
        return rc;
    }

    ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_dyups_ctx_t));
    if (ctx == NULL) {
        return NGX_ERROR;
    }

    ctx->scf = dscf;
    ctx->data = r->upstream->peer.data;
    ctx->get = r->upstream->peer.get;
    ctx->free = r->upstream->peer.free;

    r->upstream->peer.data = ctx;
    // 修改连接的 peer.get peer.free 回调
    // get 操作并无特殊处理，但是 free 操作需要考虑连接池功能
    r->upstream->peer.get = ngx_http_dyups_get_peer;
    r->upstream->peer.free = ngx_http_dyups_free_peer;

#if (NGX_HTTP_SSL)
    r->upstream->peer.set_session = ngx_http_dyups_set_peer_session;
    r->upstream->peer.save_session = ngx_http_dyups_save_peer_session;
#endif

    cln = ngx_pool_cleanup_add(r->pool, 0);
    if (cln == NULL) {
        return NGX_ERROR;
    }

    // 增加 upstream 引用计数
    dscf->ref++;

    // 注册连接关闭的清理函数：减少 upstream 引用计数
    cln->handler = ngx_http_dyups_clean_request;
    cln->data = &dscf->ref;

    return NGX_OK;
}
```

