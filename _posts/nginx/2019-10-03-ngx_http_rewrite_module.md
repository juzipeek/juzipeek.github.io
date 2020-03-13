---
layout: post
title: ngx_http_rewrite_module
category: NGINX
tags: NGINX,C
keywords: NGINX,C
description:
---

## 一 概述

`ngx_http_rewrite_module` 的主要作用有三个：改写请求的 `uri`、给客户端返回重定向（301 或 302）应答、按条件更新请求使用的配置（`uri` 重新匹配后会进行 `update_config` 操作）。模块提供的 `break`、`if`、`return`、`rewrite`、`set` 指令会按在配置文件中出现的顺序，或者说是解析配置文件时指令出现的顺序保存在 `ngx_http_rewrite_module` 模块的指令动态数组中。当请求到来时，会依次处理指令。

## 二 指令

### 1. `rewrite`

```nginx
Syntax: 	rewrite regex replacement [flag];
Default: 	—
Context: 	server, location, if
```

如果正则表达式与请求 `URI` 匹配，则将 `URI` 按 `replacement` 指定进行替换。如果 `replacement` 以 `http://`、`https://`、`$scheme` 开头，会进行重定向应答（与 `redirect` 标记行为一致）。可选的 `flag` 标记可以控制指令的处理，可取值：`last`,`break`,`redirect`,`permanent`。

- `last`

  停止处理当前的 `ngx_http_rewrite_module` 指令，使用重写后的 `URI` 重新开始进行 `location` 查找。内部重定向，客户端无感知。

- `break`

  与 `break` 指令相同，停止处理当前 `ngx_http_rewrite_module` 模块的指令，**不会触发 `location` 查找**。客户端无感知。

- `redirect`

  给客户端 `302` 临时重定向应答。

- `permanent`

  给客户端 `301` 永久重定向应答。

如果 `replacement` 替换字符串中包含新的请求参数，原始请求参数会拼接在新请求参数后。如果不希望添加原始请求参数，在参数最后添加 “?” 即可，示例：

```nginx
rewrite ^/users/(.*)$ /show?user=$1? last;
```

### 2. `break`

```nginx
Syntax: 	break;
Default: 	—
Context: 	server, location, if
```

停止 `ngx_http_rewrite_module` 模块的指令处理。

### 3. `if`

```nginx
Syntax: 	if (condition) { ... }
Default: 	—
Context: 	server, location
```

判断 `condition` 是否为 `true`，如果为 `true` 会执行括号内的 `rewrite` 模块指令，同时使用括号内的配置作为当前请求配置。可选条件为：变量、变量与字符串比较、变量与正则表达式匹配、判断文件是否存在、判断目录是否存在、判断文件/目录/符号链接是否存在、判断文件是否可执行。

### 4. `return`

```nginx
Syntax: 	return code [text];
					return code URL;
					return URL;
Default: 	—
Context: 	server, location, if
```

停止处理并将指定的状态码返回给客户端。 使用非标准的 `444` 状态码会关闭连接而不发送响应头。使用 `URL` 或 `text` 可以指定重定向 `URL`（对于代码 301、302、303、307 和 308 ）或响应正文（对于其他代码）。 响应正文文本和重定向 `URL` 可以包含变量。

### 5. `rewrite_log`

```nginx
Syntax: 	rewrite_log on | off;
Default: 	rewrite_log off;
Context: 	http, server, location, if
```

是否使用 `notice` 级别在 `error_log` 日志记录 `ngx_http_rewrite_module` 模块的指令处理结果。

### 6. `set`

```nginx
Syntax: 	set $variable value;
Default: 	—
Context: 	server, location, if
```

给变量指定值。

### 7. `uninitialized_variable_warn`

```nginx
Syntax: 	uninitialized_variable_warn on | off;
Default: 	uninitialized_variable_warn on;
Context: 	http, server, location, if
```

控制是否记录未初始化变量的警告日志。