---
layout: post
title: Valgrind DEBUG NGINX
category: NGINX
tags: NGINX,C,Linux,Valgrind
keywords: NGINX,C,Linux,Valgrind
description: 
---

## 一 valgrind 安装

```shell
wget http://www.valgrind.org/downloads/valgrind-3.14.0.tar.bz2
tar jxvf valgrind-3.14.0.tar.bz2
cd valgrind-3.14.0

./configure
make 
sudo make install
```

## 二 分析 NGINX

```shell
# 启动测试
valgrind --trace-children=yes --log-file=memcheck.log --tool=memcheck --leak-check=full /srv/nginx/sbin/nginx

# 进行测试命令
# ...

# 关闭程序
/srv/nginx/sbin/nginx -s quit
```

## 三 输出说明

- definitely lost

  > 确认丢失。程序中存在内存泄露，应尽快修复。当程序结束时如果一块动态分配的内存没有被释放且通过程序内的指针变量均无法访问这块内存则会报这个错误。

- indirectly lost

  > 间接丢失。当使用了含有指针成员的类或结构时可能会报这个错误。这类错误无需直接修复，他们总是与“definitely lost”一起出现，只要修复“definitely lost”即可。

- still reachable

  > 可以访问，未丢失但也未释放。如果程序是正常结束的，那么它可能不会造成程序崩溃，但长时间运行有可能耗尽系统资源，因此笔者建议修复它。如果程序是崩溃（如访问非法的地址而崩溃）而非正常结束的，则应当暂时忽略它，先修复导致程序崩溃的错误，然后重新检测。

- suppressed

  > 已被解决。出现了内存泄露但系统自动处理了。可以无视这类错误。