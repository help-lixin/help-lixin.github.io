---
layout: post
title: 'ThingsBoard设备管理(二)' 
date: 2021-08-07
author: 李新
tags:  ThingsBoard
---

### (1). 需求
> 通过模拟设备(MQTT.fx),与ThingsBoard进行交互.  

+ ThingsBoard添加设备
+ MQTT.fx 模拟设备

### (2). ThingsBoard添加设备

!["设备管理(一)"](/assets/thingsboard/imgs/thingsboard-dev-add-1.png)
!["设备管理(二)"](/assets/thingsboard/imgs/thingsboard-dev-add-2.jpg)
!["设备管理(三)"](/assets/thingsboard/imgs/thingsboard-dev-add-3.jpg)
!["设备管理(四)"](/assets/thingsboard/imgs/thingsboard-dev-add-4.jpg)

```
# 记住Access Token,在设备进行连接时,需要该token
tGRLBduQXsHwLGrxfLsB
```
### (3). MQTT.fx 模拟设备
!["添加模拟设备"](/assets/thingsboard/imgs/thingsboard-mqtt-fx-client-1.jpg)  
!["添加模s拟设备"](/assets/thingsboard/imgs/thingsboard-mqtt-fx-client-2.jpg)  
!["模拟设备准备连接ThingsBoard"](/assets/thingsboard/imgs/thingsboard-mqtt-fx-client-3.jpg)  
!["模拟设备连接ThingsBoard成功"](/assets/thingsboard/imgs/thingsboard-mqtt-fx-client-4.jpg)  

### (4). 上传数据
!["模拟设备向ThingsBoard发送遥测数据"](/assets/thingsboard/imgs/thingsboard-mqtt-fx-client-5.jpg)  

### (5). ThingsBoard 查看最新遥测数据
!["ThingsBoard查看最新遥测数据"](/assets/thingsboard/imgs/thingsboard-dev-data.png)
