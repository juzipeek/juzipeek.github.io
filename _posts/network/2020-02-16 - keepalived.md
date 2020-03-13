---
layout: post
title: keepalived 学习
category: Network
tags: keepalived,network,ip,multi-cast
keywords: keepalived,network,ip, multi-cast
description: keepalived 学习
---

## 一 概述

keepalived 提供了三个主要功能:管理 LVS, LVS 集群健康检查, 实现系统网络服务高可用性. Keepalived 是通过 VRRP (Virtual Router Redundancy Protocol, 虚拟路由器冗余协议）来实现网络高可用的, 当 Master 节点不可用时 Backup 节点会接管 **IP 资源**。当 Master 节点恢复时, Backup 节点会释放结果的 IP 资源.

### 1. 安装启动

```bash
$cd keepalived
$./build_setup
$./configure --prefix=/usr/local/keepalived
$make
$sudo make install
$sudo /usr/local/keepalived/sbin/keepalived -G -D -l -f /usr/local/keepalived/etc/keepalived/keepalived.conf
```

## 二 虚拟路由冗余协议

VRRP (Virtual Router Redundancy Protocol 虚拟路由器冗余协议), 是为消除在静态缺省路由环境下的缺省路由器单点故障引起的网络失效而设计的主备模式的协议，使得在发生故障而进行设备功能切换时可以不影响内外数据通信，不需要再修改内部网络的网络参数。VRRP 协议具有 IP地址备份、优先路由选择功能。

VRRP 协议将两台或多台路由器设备虚拟成一个设备, 对外提供虚拟路由器 IP(一个或多个). 而在路由器组内部, 如果实际拥有这个对外 IP 的路由器如果工作正常的话就是 MASTER, 或者是通过算法选举产生，MASTER 实现针对虚拟路由器 IP 的各种网络功能，如 ARP 请求, ICMP, 以及数据的转发等; 其他设备不拥有该 IP, 状态是 BACKUP, 除了接收 MASTER 的 VRRP 状态通告信息外, 不执行对外的网络功能. 当主机失效时, BACKUP 将接管原先 MASTER 的网络功能.

配置 VRRP 协议时需要配置每个路由器的虚拟路由器 ID(VRID)和优先权值. 使用 VRID 将路由器进行分组, 具有相同 VRID 值的路由器为同一个组, VRID是一个 0～255 的正整数. 同一组中的路由器通过使用优先权值来选举 MASTER, 优先权大者为 MASTER, 优先权也是一个 0～255 的正整数。如果优先权相同, IP 地址大的为 Master.

VRRP 协议使用多播数据来传输 VRRP 数据, VRRP 数据使用特殊的虚拟源 MAC 地址发送数据而不是自身网卡的 MAC 地址. VRRP 运行时只有 MASTER 路由器定时发送 VRRP 通告信息, 表示 MASTER 工作正常. BACKUP 只接收 VRRP 数据, 不发送数据, 如果一定时间内没有接收到 MASTER 的通告信息, 各 BACKUP 将宣告自己成为 MASTER, 发送通告信息, 重新进行 MASTER 选举状态.

VRRP 使用 `224.0.0.18` 作为广播地址, 使用 IP 协议传输数据. VRRP 协议格式:

```plain
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Type  | Virtual Rtr ID|   Priority    | Count IP Addrs|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Auth Type   |   Adver Int   |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         IP Address (1)                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            .                                  |
|                            .                                  |
|                            .                                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         IP Address (n)                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Authentication Data (1)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Authentication Data (2)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 1. 单播

除了组播外, keepalived 支持使用单播进行工作.需要修改配置文件关闭 `vrrp_strict` 选项, 并配置单播地址.

## 三 实现

keepalived 中包含三个主要模块: core, check, vrrp. core 是程序运行主模块, 用来启动管理 check, vrrp 进程, 当 check, vrrp 进程挂掉时拉起进程. check 用来对 realserver 进行检测. vrrp 进行用来实现 vrrp 逻辑.

keepalived 中使用红黑树进行定时器管理, epoll 来进行网络事件处理. 在 `vrrp_dispatcher_init` 函数中调用 `vrrp_register_workers` 注册所有 `socket` 的读事件处理函数为 `vrrp_read_dispatcher_thread`, 用来处理 VRRP 通知消息, **最后会增加定时器触发 `vrrp_read_dispatcher_thread`, 进行 VRRP 超时检查**.

VRRP 子进程启动时, 会调用 `vrrp_restore_interfaces_startup` 函数, 依次对所有的 VIP 接口调用 `vrrp_restore_interface` 函数进行 VRRP 恢复. 在 `start_vrrp` 中会添加 `vrrp_dispatcher_init` 定时器, 触发发送 VRRP 通告.

### 1. 调试

使用 `gdb` 调试 keepalived 来分析源码. 因为会 `fork` 子进程进行处理, 需要使用 `set follow-fork-mode child` 配置在 `fork` 时调试子进程.

```gdb
sudo gdb /usr/local/keepalived/sbin/keepalived
set args -n -G -D -l -f /usr/local/keepalived/etc/keepalived/keepalived.conf
# 在 start_vrrp_child 函数内设置跟随子进程
set follow-fork-mode child
```

### 2. 发送通告

`vrrp_send_pkt` 函数实现通告发送, **使用配置的组播地址(224.0.0.18)作为目的地址**.

```c
static ssize_t
vrrp_send_pkt(vrrp_t * vrrp, unicast_peer_t *peer)
{
    struct sockaddr_storage *src = &vrrp->saddr;
    struct msghdr msg;
    struct iovec iov;
    char cbuf[256];

    /* Build the message data */
    memset(&msg, 0, sizeof(msg));
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    // 发送的 IP 数据包(datagram)内容
    iov.iov_base = vrrp->send_buffer;
    iov.iov_len = vrrp->send_buffer_size;

    /* Unicast sending path */
    if (peer && peer->address.ss_family == AF_INET) {
        msg.msg_name = &peer->address;
        msg.msg_namelen = sizeof(struct sockaddr_in);
    } else if (peer && peer->address.ss_family == AF_INET6) {
        msg.msg_name = &peer->address;
        msg.msg_namelen = sizeof(struct sockaddr_in6);
        vrrp_build_ancillary_data(&msg, cbuf, src, vrrp);
    } else if (vrrp->family == AF_INET) { /* Multicast sending path */
        msg.msg_name = &global_data->vrrp_mcast_group4; // 配置的组播地址
        msg.msg_namelen = sizeof(struct sockaddr_in);
    } else if (vrrp->family == AF_INET6) {
        msg.msg_name = &global_data->vrrp_mcast_group6;
        msg.msg_namelen = sizeof(struct sockaddr_in6);
        vrrp_build_ancillary_data(&msg, cbuf, src, vrrp);
    }

    /* Send the packet */
    return sendmsg(vrrp->sockets->fd_out, &msg, (peer) ? 0 : MSG_DONTROUTE);
}
```

## 参考文档

- [keepalived manpage](https://www.keepalived.org/manpage.html)
- [keepalived docs](https://www.keepalived.org/doc/configuration_synopsis.html)
- [keepalived 实现服务高可用](https://www.cnblogs.com/clsn/p/8052649.html)
- [Keepalived原理与实战精讲--VRRP协议](https://blog.csdn.net/u010391029/article/details/48311699)
- [使用 GDB 调试多进程程序](https://www.ibm.com/developerworks/cn/linux/l-cn-gdbmp/index.html)
