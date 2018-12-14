---
layout: post
title: Context 包简介
category: C
tags: C
keywords: C
description: 动态库中的全局变量
---

## 概述

如果两个动态库中有相同的全局变量会不会冲突？

## 测试

动态库 1：

```c
// dynamic_1.c
// gcc -fPIC -shared -Wall -g dynamic_1.c -o libshared_1.so
// 
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>
#include <string.h>

#include <time.h>
#include <sys/time.h>
#include <sys/types.h>
#include <sys/timerfd.h>

#include "log.h"
#include "local.h"

int g_val = 12;

int hello() {

    trace_debug( "in shared_hello static_hello addr:[%p][%d]", hello, g_val);
    
    return 0;
}
```

动态库 2：

```c
// dynamic_2.c
// gcc -fPIC -shared -Wall -g dynamic_2.c -o libshared_2.so
// 
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>
#include <string.h>

#include <time.h>
#include <sys/time.h>
#include <sys/types.h>
#include <sys/timerfd.h>

#include "log.h"
#include "local.h"

int g_val = 13;

int hello() {

    trace_debug( "in shared_hello static_hello addr:[%p][%d]", hello, g_val);
    
    return 0;
}
```

测试：

```c
// dlopen.c
// gcc -rdynamic -Wall -g dlopen.c -ldl -o run
// 
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>
#include <string.h>

#include <dlfcn.h>
#include <time.h>
#include <sys/time.h>
#include <sys/types.h>
#include <sys/timerfd.h>

#include "log.h"

typedef int (*function_hello)();

int main(int argc, char **argv) {
    char log_file[256] = {0};

    sprintf( log_file,"%s/log.out", getenv("HOME") );
    log_init( log_file );

    void *handle1 = dlopen("./libshared_1.so", RTLD_NOW);
    if(!handle1) {
        trace_debug("load  failed! error:%s", dlerror());
        exit(1);
    }

    void *handle2 = dlopen("./libshared_2.so", RTLD_NOW);
    if(!handle2) {
        trace_debug("load  failed! error:%s", dlerror());
        exit(1);
    }

    //clear error info
    dlerror();

    function_hello f1  = (function_hello) dlsym(handle1, "hello");
    char *err = dlerror();
    if(err) {
        trace_debug("can't find symbol fcn! %s", err);
        exit(1);
    }

    function_hello f2  = (function_hello) dlsym(handle2, "hello");
    err = dlerror();
    if(err) {
        trace_debug("can't find symbol fcn! %s", err);
        exit(1);
    }

    f1();
    f2();

    dlclose(handle1);
    dlclose(handle2);

    return 0;
}
```

运行输出：

```shell
PID-TID:12984-12984|19:22:46.383|shared.c| hello| 19|D| in shared_hello static_hello addr:[0x7f2096fee6d0][12]
PID-TID:12984-12984|19:22:46.383|shared2.| hello| 20|D| in shared_hello 2 static_hello addr:[0x7f2096dec6d0][13]
```

说明两个动态库中的全局变量并不会冲突。

## TODO

原理上解释原因。

