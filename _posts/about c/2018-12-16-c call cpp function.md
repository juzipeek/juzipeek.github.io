---
layout: post
title: c call cpp function
category: C
tags: C
keywords: C
description: c 调用 cpp 函数
---

## 1. 概述

在做开发时项目是用 `C` 语言写的，有时候需要调用 `C++` 接口或库。本文对此进行总结记录，以备不时之需。

## 2. `C++` 函数声明

当需要调用 `C++` 接口时可以写一个单独的 `C++` 源文件实现包裹函数对接口的操作。此时只需要对包裹函数做 `extern "C"` 声明。如下所示：

```c++
// dynamodb.cpp

#include <aws/core/Aws.h>
#include <aws/core/utils/Outcome.h>
#include <aws/dynamodb/DynamoDBClient.h>
#include <aws/dynamodb/model/AttributeDefinition.h>
#include <aws/dynamodb/model/GetItemRequest.h>
#include <aws/dynamodb/DynamoDBErrors.h>

#include "dynamodb.h"

static Aws::SDKOptions g_sdk_options;
static Aws::DynamoDB::DynamoDBClient *g_db_client;

/******************************************************************************
 * shutdown
 ******************************************************************************/
void
du_dynamodb_shutdown()
{
    Aws::ShutdownAPI(g_sdk_options);
}

/******************************************************************************
 * init
 ******************************************************************************/
void
du_dynamodb_init()
{
    Aws::InitAPI(g_sdk_options);

    Aws::Client::ClientConfiguration cfg;

    cfg.region = Aws::String(AWS_REGION);
    cfg.maxConnections = DYNAMODB_MAX_CONNECTION;
    cfg.connectTimeoutMs = DYNAMODB_CONNECTION_TIMEOUT;
    cfg.requestTimeoutMs = DYNAMODB_CONNECTION_TIMEOUT;

    // cfg.tcpKeepAliveIntervalMs = 30;

    g_db_client = new Aws::DynamoDB::DynamoDBClient(cfg);
}

/******************************************************************************
 *
 ******************************************************************************/
DU_DB_ERR
du_dynamodb_get_idfa_type(const char *did, int *idfa_type)
{
    DU_DB_ERR ret = DU_DB_ERR_INIT;
    Aws::DynamoDB::DynamoDBErrors dynamodb_err;
    Aws::DynamoDB::Model::GetItemRequest queryReq;

    do {
        queryReq.SetTableName(DEVICE_TABLE);
        queryReq.SetProjectionExpression(IDFA_TYPE_FIELD);

        if (!idfa_type) {
            ret = DU_DB_ERR_BAD_PARAMETER;
            break;
        }

        if (!did) {
            ret = DU_DB_ERR_BAD_PARAMETER;
            break;
        }
        Aws::DynamoDB::Model::AttributeValue did_key;
        did_key.SetS(did);
        queryReq.AddKey(QUERY_DID_FIELD, did_key);

        const Aws::DynamoDB::Model::GetItemOutcome&
            result = g_db_client->GetItem(queryReq);

        if (!result.IsSuccess()) {
            dynamodb_err = result.GetError().GetErrorType();
            if (Aws::DynamoDB::DynamoDBErrors::NETWORK_CONNECTION == dynamodb_err ||
                    Aws::DynamoDB::DynamoDBErrors::REQUEST_TIMEOUT == dynamodb_err) {
            }
            du_log(NGX_LOG_ERR, NULL, 0, "GetItem fail! error:%d", dynamodb_err);
            ret = DU_DB_ERR_QUERY_FAIL;
            break;
        }

        const Aws::Map<Aws::String, Aws::DynamoDB::Model::AttributeValue>&
            item = result.GetResult().GetItem();
        if (!item.size()) {
            // find attribute NULL
            du_log(NGX_LOG_ERR, NULL, 0, "GetResult fail! item size is zero.");
            ret = DU_DB_ERR_FETCH_NULL;
            break ;
        }

        try {
            *idfa_type = atoi(item.at("idfa_type").GetN().c_str());
        } catch (const std::exception& ex) {
            // error process
            // catch(const std::out_of_range& oor)
            du_log(NGX_LOG_ERR, NULL, 0, "get idfa_type fail! exception:%s", ex.what());
            ret = DU_DB_ERR_FETCH_NULL;
            break;
        }

        ret = DU_DB_OK;
    } while(0);

    du_log(NGX_LOG_DEBUG, NULL, 0, "dynamodb_get_idfa_type ret:%d", ret);

    return ret;
}

/*******************************************************************************
 *
 ******************************************************************************/
DU_DB_ERR
du_dynamodb_get_app_type(const char *table_name, const char *did, const char *ver_q, size_t ver_size,
                         char *ver, int *normal_times, int *duplicate_times, int *recall_times, int *update_times)
{
    DU_DB_ERR ret = DU_DB_ERR_INIT;
    Aws::DynamoDB::DynamoDBErrors dynamodb_err;
    Aws::DynamoDB::Model::GetItemRequest queryReq;

    do {
        if (!table_name || !did || !ver || !normal_times || !duplicate_times || !update_times || !recall_times) {
            // param error
            ret = DU_DB_ERR_BAD_PARAMETER;
            break;
        }

        queryReq.SetTableName(table_name);
        queryReq.SetProjectionExpression(APP_TYPE_FIELD);

        Aws::DynamoDB::Model::AttributeValue did_key;
        did_key.SetS(did);

        queryReq.AddKey(QUERY_DID_FIELD, did_key);
        if (ver_q && strlen(ver_q)) {
            Aws::DynamoDB::Model::AttributeValue ver_key;
            ver_key.SetS(ver_q);
            queryReq.AddKey(QUERY_VER_FIELD, ver_key);
        }


        const Aws::DynamoDB::Model::GetItemOutcome&
            result = g_db_client->GetItem(queryReq);
        if (!result.IsSuccess()) {
            // Not Found Or error
            dynamodb_err = result.GetError().GetErrorType();
            if (Aws::DynamoDB::DynamoDBErrors::NETWORK_CONNECTION == dynamodb_err ||
                    Aws::DynamoDB::DynamoDBErrors::REQUEST_TIMEOUT == dynamodb_err) {
            }
            du_log(NGX_LOG_ERR, NULL, 0, "GetItem fail! error:%d", dynamodb_err);
            ret = DU_DB_ERR_QUERY_FAIL;
            break;
        }

        const Aws::Map<Aws::String, Aws::DynamoDB::Model::AttributeValue>&
            item = result.GetResult().GetItem();
        if (!item.size()) {
            // find attribute NULL
            du_log(NGX_LOG_ERR, NULL, 0, "GetResult fail! item size is zero.");
            ret = DU_DB_ERR_FETCH_NULL;
            break;
        }

        try {
            snprintf(ver, ver_size, "%s", item.at("ver").GetS().c_str());
            *normal_times = atoi(item.at("normal").GetN().c_str());
            *duplicate_times = atoi(item.at("dup").GetN().c_str());
            *recall_times = atoi(item.at("recall").GetN().c_str());
            *update_times = atoi(item.at("update").GetN().c_str());
        } catch(const std::exception& ex) {
            // error process
            // catch(const std::out_of_range& oor)
            du_log(NGX_LOG_ERR, NULL, 0, "get app_type fail! exception:%s", ex.what());
            ret = DU_DB_ERR_FETCH_NULL;
            break;
        }

        // success
        ret = DU_DB_OK;
    } while(0);

    du_log(NGX_LOG_DEBUG, NULL, 0, "du_dynamodb_get_app_type ret:%d", ret);

    return ret;
}
```

头文件写法：

```c++
// dynamodb.h
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>
#include <string.h>
#include <time.h>
#include <sys/time.h>
#include <sys/types.h>

#include "log.h"

#ifndef _DU_DYNAMODB_H__
#define _DU_DYNAMODB_H__

#ifdef __cplusplus
extern "C"
{
#endif

#define AWS_REGION "us-east-2"
#define DEVICE_TABLE "device"
#define IDFA_TYPE_FIELD "idfa_type"
#define QUERY_DID_FIELD "did"

#define APP_TYPE_FIELD "idfa_type"
#define QUERY_VER_FIELD "ver"

#define DYNAMODB_MAX_CONNECTION 40
#define DYNAMODB_CONNECTION_TIMEOUT 1000

#define du_log( level, log, errno, fmt, ... )   trace_debug( fmt "\t", ##__VA_ARGS__ );

/* 数据库操作返回码 */
typedef enum du_db_err {
    DU_DB_OK = 0,
    DU_DB_DB_OVER_MAX,
    DU_DB_ERR_BAD_PARAMETER,
    DU_DB_ERR_INIT,
    DU_DB_ERR_NEW_URI,
    DU_DB_ERR_CONNECT,
    DU_DB_ERR_CONNECT_TOO_OFTEN,
    DU_DB_ERR_NO_MEMORY,
    DU_DB_ERR_QUERY_FAIL,
    DU_DB_ERR_FETCH_NULL,
} DU_DB_ERR;

void
du_dynamodb_shutdown();

void
du_dynamodb_init();

DU_DB_ERR
du_dynamodb_get_idfa_type(const char *did, int *idfa_type);

DU_DB_ERR
du_dynamodb_get_app_type(const char *table_name, const char *did, const char *ver_q, size_t ver_size,
                         char *ver, int *normal_times, int *duplicate_times, int *recall_times, int *update_times);

#ifdef __cplusplus
}
#endif
#endif
```

以上示例并没有涉及到成员函数的调用，如果需要可以参见[链接](http://sheldonrush.github.io/sheldon.is.a.geek/2015/06/27/How-to-call-C-calss-member-function-from-C/)。

## 3. 编译注意

因为项目是用 `C` 编写，项目的执行文件需要使用 `gcc` 进行编译（Linux 平台）。此时需要添加链接选项 `-lstdc++`，不然项目编译会失败。

## 4. 参考链接

-   [mixing-c-and-cpp](https://isocpp.org/wiki/faq/mixing-c-and-cpp)
-   [How to call C++ class member function from C](http://sheldonrush.github.io/sheldon.is.a.geek/2015/06/27/How-to-call-C-calss-member-function-from-C/)