---
layout: post
title: coredumpctl
category: Linux
tags: Linux
keywords: Linux
description: 
---

## 概述

`coredumpctl` 用来从 `systemd-journald` 中检索 `coredump`，非 `systemd` 进程的 `coredump` 不能使用。

## 使用

```shell
# 列出日志系统收集的 coredump
coredumpctl list

# info
coredumpctl info [PID]

# 获得 coredump 文件
coredumpctl dump -o pid.coredump [PID]
```

## systemd

使用 `systemd` 管理服务时需要进行特殊配置才能产生 `coredump` 文件（现在我还找到可行配置）。对于被管理的 `NGINX` 服务需要进行 `stop` 然后使用 `/srv/nginx/sbin/nginx` 进行启动，才能保证生成 `coredump` 文件。