---
layout: post
title: 'Eureka服务集群续租与集群同步'
date: 2019-06-29
author: 李新
tags: Eureka
---

### (1).Eureka续租
```
// UP/DOWN
// PUT  http://domain/eureka/apps/eurekaInstanceId?status=UP&lastDirtyTimestamp=1593149266991


http://localhost:2222/eureka/apps/EUREKA-SERVER/172.17.1.135:eureka-server:1111?status=UP&lastDirtyTimestamp=1593149266991

http://localhost:1111/eureka/apps/TEST-PROVIDER/172.17.1.135:test-provider:8080?status=UP&lastDirtyTimestamp=1593246410112
```
### (2).Eureka集群间同步

```
// 剔除自己节点,其余节点的数据.
{
    "replicationList":[
        {
            "appName":"EUREKA-SERVER",
            "id":"172.17.1.135:eureka-server:3333",
            "lastDirtyTimestamp":1593149702221,
            "status":"UP",
            "action":"Heartbeat"
        }]
}
```

```
// POST /eureka/peerreplication/batch/
http://localhost:3333/eureka/peerreplication/batch
```

### (3).下线某个节点
```
DELETE /eureka/apps/EUREKA-SERVER/172.17.1.135:eureka-server:1111
// http://localhost:1111/eureka/apps/EUREKA-SERVER/TEST-PROVIDER/172.17.1.135:test-provider:8080?status=DOWN&lastDirtyTimestamp=1593246410112
```