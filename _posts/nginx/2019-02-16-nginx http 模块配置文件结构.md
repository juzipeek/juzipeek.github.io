---

---

## gdb 输出

```
# 
p *(ngx_http_conf_ctx_t *)ngx_cycle->conf_ctx[8]
# http_core_loc_conf
p *(ngx_http_core_loc_conf_t*)(*(ngx_http_conf_ctx_t *)ngx_cycle->conf_ctx[8])->loc_conf
# http_core_loc_conf 内部内容查看
p *(*(ngx_http_core_loc_conf_t*)(*(ngx_http_conf_ctx_t *)ngx_cycle->conf_ctx[8])->loc_conf)->static_locations
p *(ngx_http_location_queue_t*)(*(ngx_http_core_loc_conf_t*)(*(ngx_http_conf_ctx_t *)ngx_cycle->conf_ctx[8])->loc_conf)->locations

p *(ngx_http_location_queue_t*)(*(ngx_http_core_loc_conf_t*)(*(ngx_http_conf_ctx_t *)ngx_cycle->conf_ctx[8])->loc_conf[1])->locations

p *(ngx_http_conf_ctx_t *)((ngx_http_conf_ctx_t *)ngx_cycle->conf_ctx[8])->main_conf
p *(ngx_http_core_main_conf_t *)((ngx_http_conf_ctx_t *)(ngx_cycle->conf_ctx[8]))->main_conf[0]
p *(ngx_http_core_srv_conf_t*)(((ngx_http_core_main_conf_t *)((ngx_http_conf_ctx_t *)(ngx_cycle->conf_ctx[8]))->main_conf[0])->servers)
```

## main 层次

`main` 层次的存储使用的是 `ngx_cycle_t::conf_ctx`，它只存储 `core` 模块的配置数据。`core` 模块数量很少，所以 `ngx_cyclte_t::conf_ctx` 数组几乎为空，主要存储 `ngx_core_module` 模块的配置数据，用以解析 `master`、`daemon` 等指令。

`conf_ctx` 四层指针的理解：第一层指针数组是核心模块使用；对于 HTTP 模块，指针数组中的每个元素指向的是 `void *` 类型的指针数组（`ngx_http_conf_ctx_t *` 类型指针数组），此指针数组用于保存不同 HTTP 模块的配置。

## http 层次

当解析到 `http` 指令时会调用 `ngx_http_block` 函数进行解析，会创建 `ngx_http_conf_ctx_t` 结构。http main 层次的 `ngx_http_conf_ctx_t` 结构不仅保存了所有 http 模块的 `main` 配置数据，同时也保存了所有 http 模块的 `server` 和 `location` 配置数据。这样实现了 Nginx 配置文件的指令作用域功能：低层次的指令可以在高层次进行配置，高层次指令的值作为低层次指令的默认值，多个低层次的指令可以共享高层次指令的值。

## server 层次

`ngx_http_core_module` 模块是第一个 `http` 模块，它的配置数据结构 `ngx_http_core_main_conf_t` 里有个动态数组成员 `servers`，保存了所有 `server` 相关的信息（`ngx_http_core_srv_conf_t`）。
每个 `server` 块同样使用 `ngx_http_conf_ctx_t` 里的三个数组来保存模块配置数据，以为 `http` 块的配置不会出现在 `server` 块中，所有不会创建 `main` 配置数据，而是直接引用。

## location 层次

`ngx_http_core_module` 模块使用结构体 `ngx_http_core_loc_conf_t` 以队列的形式保存 server 里的所有 location 信息。

location 块与 http 和 server 块一样，使用 `ngx_http_conf_ctx_t` 的三个数组保存模块配置数据，但用指针引用上层 main 和 server 的配置数据，只调用模块的 `create_loc_conf`，只用一个数组持有所有 http 模块在本 location 里的配置数据。

