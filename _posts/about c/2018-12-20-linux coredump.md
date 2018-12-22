---
layout: post
title: linux coredump 产生配置
category: C
tags: C
keywords: C
description: coredump
---

## 概述

在开发过程中有可能出现异常，虽然通过 `dmesg` 可以看到错误的 `ip` 但是只能定位到错误行号，执行上下文其实是看不到的。系统生成的 `coredump` 文件可以方便分析执行上下文。

## 开启 `coredump` 产生

### 1. `limits.conf` 配置

```shell
vi /etc/security/limits.conf

# 行未添加
* soft core 20480
* hard core 20480

# 需要重新登录后才能生效
ulimit -a
```

### 2. `core` 文件路径

```
vi /etc/sysctl.conf

# 添加记录
kernel.core_pattern = /tmp/core.%e.%t

# 生效
sysctl -p
```

## nginx 配置

可以在 `nginx.conf` 中配置 `coredump` 生成路径以及文件名：

```nginx.
worker_rlimit_core 100m;
working_directory  /tmp;
```

