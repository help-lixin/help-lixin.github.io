---
layout: post
title: 'Pulsar源码目录结构介绍(一)' 
date: 2023-01-22
author: 李新
tags:  Pulsar
---

### (1). Pulsar Broker源码结构

|  模块名称                        | 介绍  |
|  ----                          | ----  |
| pulsar-broker-common           | Broker共用模块 |
| pulsar-broker                  | Broker服务端 |
| pulsar-broker-auth-athenz      | Athenz身份验证 |
| pulsar-broker-auth-sasl        | SASL身份验证 |
| pulsar-broker-shaded           | Broker打包模块 |

### (2). Pulsar Client源码结构

|  模块名称   | 介绍  |
|  ----  | ----  |
| pulsar-client                      | client模块 |
| pulsar-client-1x-base              | ? |
| pulsar-client-admin-api            | admin api模块 |
| pulsar-client-admin                | admin应该是管理topic |
| pulsar-client-admin-shaded         | admin打包模块 |
| pulsar-client-all                  | client打包模块 |
| pulsar-client-api                  | client api模块 |
| pulsar-client-auth-athenz          | client Athenz模块 |
| pulsar-client-auth-sasl            | client SASL模块 |
| pulsar-client-messagecrypto-bc     | client bc加密模块 |
| pulsar-client-tools                | client tools模块 |
| pulsar-client-tools-test           | client 测试模块 |
| pulsar-client-shaded               | client 打包模块 |
| pulsar-client-cpp                  | client C++模块 |


### (3). Pulsar 其它模块

|  模块名称   | 介绍  |
|  ----  | ----  |
|  pulsar-common                        | 公共模块 |
|  pulsar-proxy                         | proxy模块 |
|  pulsar-metadata                      | 元数据模块 |
|  pulsar-sql                           | ???????? |
|  pulsar-websocket                     | websocket模块 |
|  pulsar-zookeeper-utils               | 与zk交互模块 |

### (4). Pulsar模块总结
总体来说,Pulsar还是做了一些设计后再开发的,还是挺优秀的,不过,模块拆分得有一些乱,比如:broker,为什么不能像pulsar-io那样,做成一个父工程来承载,当然,也有可能是我的知识有限. 