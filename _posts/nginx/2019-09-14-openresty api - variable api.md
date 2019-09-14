---
layout: post
title: OpenResty Api - variable api
category: NGINX
tags: NGINX,OpenResty,C,Lua
keywords: NGINX,OpenResty,C,Lua
description:
---

## 一 概述

`variable` 接口可以读取、设置变量。在 `lua-nginx` 模块中使用 `ngx.var` 表的 `__index` 和 `__newindex` 元方法实现变量的读取、设置功能。

## 二 实现

### 1.`ngx.var.VARIABLE`

NGINX 中变量不能动态创建，需要预定义变量。预定义的变量定义在 `ngx_http_core_module` 的 `variables_hash` 表中。在 `__index` 元方法中，除了查找 `variables_hash` 表，还会根据变量的前缀在请求头（`http_`）、应答头(`sent_http_`)、发向上游的头(`upstream_http_`)、请求 `cookie`(`cookie_`)、发向上游的 `cookie`(`upstream_cookie_`)以及请求 `uri` 参数(`arg_`)中查找变量。

对于 `__newindex` 元方法，只能更新预定义变量，无法更新请求头、请求参数等。

