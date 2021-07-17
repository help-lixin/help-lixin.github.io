---
layout: post
title: 'Apollo 生产部署建议(八)'
date: 2021-07-16
author: 李新
tags:  Apollo
---

### (1). 概述
在这一小节,主要讲解,如何把Apollo部署到生产上,我这里因为,机器资源有限,就在本机部署,只要能让服务达到高可用即可.   
主要解决两个服务的高可用(apollo-configservice和apollo-adminservice).             
我这里只对:apollo-configservice解决高可用问题,至于,apollo-adminservice做法一样.  

### (2). 机器准备

|  机器            | 服务名称                |
|  ----           | ----                   |
| 127.0.0.1:8090  | apollo-adminservice    |
| 127.0.0.1:7070  | apollo-configservice   |
| 127.0.0.1:8080  | apollo-configservice   |
| 127.0.0.1:8070  | apollo-portal          |

### (3). 修改Eureka
```
UPDATE ApolloConfigDB.ServerConfig
SET value = 'http://127.0.0.1:8080/eureka/,http://127.0.0.1:7070/eureka/'
WHERE `Key` = 'eureka.service.url' ;
```
### (4). 查看Eureka信息
!["Eureka界面"](/assets/apollo/imgs/apollo-eureka-home.png)

### (5). 如何知道apollo-configservice是高可用的呢
> 打断点在ConfigServiceLocator类的updateConfigServices方法上,查看发起HTTP请求后的结果是否包含有两个:apollo-configservice的IP和端口即可.      

```
System.getProperties().put("env", "DEV");
System.getProperties().put("app.id", "7BBB492B-62F8-453F-B50B-0D568308E87A");

// 在这里配置的是:meta-service(config-service),从这里也能看出来,Apollo确实不能部署到生产上去的.
System.getProperties().put("apollo.meta", "http://127.0.0.1:7070");
// 可以指定多个:meta-service(config-service)
// System.getProperties().put("apollo.meta", "http://127.0.0.1:8080,http://127.0.0.1:7070");

Config config = ConfigService.getConfig("TEST1.jdbc");
String key = "jdbc.url";
String defaultValue = "default value";
String value = config.getProperty(key, defaultValue);
System.out.println("value: " + value);
```


```
# ConfigServiceLocator.updateConfigServices方法断点,查看请求后的结果集.
[
  ServiceDTO{appName='APOLLO-CONFIGSERVICE', instanceId='lixin-macbook.local:apollo-configservice:7070', homepageUrl='http://127.0.0.1:7070/'}, 
  ServiceDTO{appName='APOLLO-CONFIGSERVICE', instanceId='lixin-macbook.local:apollo-configservice:8080', homepageUrl='http://127.0.0.1:8080/'}
]
```