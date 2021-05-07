---
layout: post
title: 'Servicecomb Pack之Saga+Docker入门(二)'
date: 2021-05-04
author: 李新
tags:  Servicecomb-Pack
---

### (1). 源码构建
> 稍微吐槽下,官方提供的Servicecomb Pack案例,是依赖:Docker的.  
> 这样做有优势,可以利用docker快速启动项目(不必理会Pass层的依赖,比如:postgres/mysql...),缺点,不知道真正能快速启动的人有多少,毕竟,不是谁都会Docker的.   
> ["Saga Spring Demo官网详细介绍"](https://github.com/help-lixin/servicecomb-pack/blob/master/demo/saga-spring-demo/README.md)   

```
# clone源码后所在目录
lixin-macbook:servicecomb-pack-0.6.0 lixin$ pwd
/Users/lixin/GitRepository/servicecomb-pack-0.6.0

# 编译打包,并构建docker镜像
lixin-macbook:servicecomb-pack-0.6.0 lixin$ mvn clean install -DskipTests -Pdocker -Pdemo
```
### (2). 为sh添加可执行
```
lixin-macbook:servicecomb-pack-0.6.0 lixin$ cd ./demo/saga-spring-demo/
lixin-macbook:saga-spring-demo lixin$ chmod 755 saga-demo.sh 
```
### (3). 启动saga案例
```
lixin-macbook:saga-spring-demo lixin$ ./saga-demo.sh up
Starting saga-demo:0.6.0
Creating network "saga-spring-demo_default" with the default driver
Creating saga-spring-demo_database_1 ... done
Creating saga-spring-demo_alpha_1    ... done
Creating saga-spring-demo_hotel_1    ... done
Creating saga-spring-demo_car_1      ... done
Creating saga-spring-demo_booking_1  ... done
... ...
```
### (4). 查看docker启动的容器
!["Servicecomb Pack Saga Docker"](/assets/servicecomb-pack/imgs/saga-spring-demo.jpg)

### (5). 测试
> 业务知识:用户(test),向网站(booking)发起请求,订2辆台(car)和2间房(hotel).   

!["Saga案例"](/assets/servicecomb-pack/imgs/Servicecomb-Pack-Saga-Demo.jpg)

```
# 模拟订车和订房服务.
lixin-macbook:~ lixin$ curl -X POST http://localhost:8083/booking/test/2/2
test booking 2 rooms and 2 cars OK

# 查看服务状态.
lixin-macbook:~ lixin$ curl http://localhost:8081/bookings
[{"id":1,"name":"test","amount":2,"confirmed":true,"cancelled":false}]
```
### (6). 总结
> Demo是跑起来了,后续会对源码进行一个深度的剖析.   