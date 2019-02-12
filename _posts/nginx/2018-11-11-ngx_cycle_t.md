---
layout: post
title: ngx_cycle_t 中 connections 说明
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description: 
---

## ngx_cycle_t

```c
typedef struct ngx_cycle_s           ngx_cycle_t;
struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;

    ngx_log_t                *log;
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;	// 指向 connection 中的一个可用连接
    ngx_uint_t                free_connection_n;

    ngx_module_t            **modules;
    ngx_uint_t                modules_n;
    ngx_uint_t                modules_used;    /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;
    ngx_uint_t                reusable_connections_n;

    ngx_array_t               listening;
    ngx_array_t               paths;

    ngx_array_t               config_dump;
    ngx_rbtree_t              config_dump_rbtree;
    ngx_rbtree_node_t         config_dump_sentinel;

    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                files_n;

    ngx_uint_t                connection_n;
    ngx_connection_t         *connections;	// 与 free_connections 配合形成链表结构
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_cycle_t              *old_cycle;

    ngx_str_t                 conf_file;
    ngx_str_t                 conf_param;
    ngx_str_t                 conf_prefix;
    ngx_str_t                 prefix;
    ngx_str_t                 lock_file;
    ngx_str_t                 hostname;
};
```

## ngx_connection_s

```c
typedef struct ngx_connection_s      ngx_connection_t;

struct ngx_connection_s {
    void               *data;	// 本结构的巧妙之处
    ngx_event_t        *read;
    ngx_event_t        *write;

    ngx_socket_t        fd;

    ngx_recv_pt         recv;
    ngx_send_pt         send;
    ngx_recv_chain_pt   recv_chain;
    ngx_send_chain_pt   send_chain;

    ngx_listening_t    *listening;

    off_t               sent;

    ngx_log_t          *log;

    ngx_pool_t         *pool;

    int                 type;

    struct sockaddr    *sockaddr;
    socklen_t           socklen;
    ngx_str_t           addr_text;

    ngx_str_t           proxy_protocol_addr;
    in_port_t           proxy_protocol_port;

#if (NGX_SSL || NGX_COMPAT)
    ngx_ssl_connection_t  *ssl;
#endif

    struct sockaddr    *local_sockaddr;
    socklen_t           local_socklen;

    ngx_buf_t          *buffer;

    ngx_queue_t         queue;

    ngx_atomic_uint_t   number;

    ngx_uint_t          requests;

    unsigned            buffered:8;

    unsigned            log_error:3;     /* ngx_connection_log_error_e */

    unsigned            timedout:1;
    unsigned            error:1;
    unsigned            destroyed:1;

    unsigned            idle:1;
    unsigned            reusable:1;
    unsigned            close:1;
    unsigned            shared:1;

    unsigned            sendfile:1;
    unsigned            sndlowat:1;
    unsigned            tcp_nodelay:2;   /* ngx_connection_tcp_nodelay_e */
    unsigned            tcp_nopush:2;    /* ngx_connection_tcp_nopush_e */

    unsigned            need_last_buf:1;

#if (NGX_HAVE_AIO_SENDFILE || NGX_COMPAT)
    unsigned            busy_count:2;
#endif

#if (NGX_THREADS || NGX_COMPAT)
    ngx_thread_task_t  *sendfile_task;
#endif
};
```

​	NGINX 需要处理大量的网络连接，如何当有新的请求过来需要创建给它分配空间进行存储，当连接断开时需要将资源进行回收（释放或重置继续使用）。如果对每个连接都进行创建、释放存储资源会导致很多的内存申请释放调用导致性能降低。如果使用固定数组来保存连接信息，虽然避免了重复内存的申请释放调用，但是查找数组中使用、未使用连接会成为瓶颈（数组遍历）。NGINX 中使用了巧妙的方法解决这个问题，使用固定内存来存储连接信息，同时将每个节点通过一个 `data` 指针串联起来，这样连接表就成为一个链表。cycle 变量持有链表中第一可用的节点，当需要申请新的连接结构体时链表将当前节点给用户使用并前进；当需要释放连接时，将链表头作为释放节点的后继节点，链表头指向释放的节点。

## connections/free_connections 使用

​	connections 在 `ngx_event_process_init` 中初始化，示例代码：

```c
ngx_uint_t           m, i;
ngx_connection_t    *c, *next, *old;
// 内存分配
cycle->connections = ngx_alloc(sizeof(ngx_connection_t) * cycle->connection_n, cycle->log);

c = cycle->connections;
i = cycle->connection_n;
next = NULL;
// 链表建立
do {
    i--;

    c[i].data = next;
    c[i].read = &cycle->read_events[i];
    c[i].write = &cycle->write_events[i];
    c[i].fd = (ngx_socket_t) -1;

    next = &c[i];
} while (i);

// free_connections 为链表头
cycle->free_connections = next;
cycle->free_connection_n = cycle->connection_n;
```

​	获取连接 `ngx_get_connection`，从链表中取可用节点：

```c
ngx_connection_t  *c;

c = ngx_cycle->free_connections;			// 取节点值
ngx_cycle->free_connections = c->data;		// 链表头前进
ngx_cycle->free_connection_n--;
```

​	释放连接 `ngx_free_connection`，将节点归还给链表：

```c
void
ngx_free_connection(ngx_connection_t *c)
{
    c->data = ngx_cycle->free_connections;
    ngx_cycle->free_connections = c;
    ngx_cycle->free_connection_n++;

    // 可用忽略
    if (ngx_cycle->files && ngx_cycle->files[c->fd] == c) {
        ngx_cycle->files[c->fd] = NULL;
    }
}
```

