---
layout: post
title: 修改常量值
category: C
tags: C
keywords: C
description:
---

## 一 常量值

`const` 修饰符修饰变量是不可修改的, 如果尝试修改 `const` 类型变量会编译失败. 但是使用指针修改 `const` 变量能修改常量.

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int const n = 10;
    printf("n = [%d]\n", n);

    // 编译错误
    // n = 11;
    // printf("n = [%d]\n", n);

    // 编译成功, 但是有 discarded-qualifiers 警告
    int *p = &n;
    *p = 12;
    printf("n = [%d]\n", n);
}
```

**`const` 修饰符是在编译时起作用的**, 编译出的汇编代码是没有差异的. 可以使用 `gcc -S` 分别编译两个版本进行对比. 这个特性不应该滥用.
