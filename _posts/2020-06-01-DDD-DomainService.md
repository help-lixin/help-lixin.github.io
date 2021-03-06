---
layout: post
title: 'DDD 应用服务和领域服务(六)'
date: 2020-06-01
author: 李新
tags: DDD
---

### (1). 引言
> DDD的战略设计知识,来源于:极客时间<DDD实战课>和(https://www.cnblogs.com/sheng-jie/p/6931646.html)            
> DDD的战术设计知识来源于:https://github.com/citerus/dddsample-core.git    
### (2). DDD分层架构
!["DDD分层架构"](/assets/ddd/imgs/ddd.webp)

> 应用层(Application):负责展现层与领域层之间的协调,协调业务对象来执行特定的应用程序任务.它不包含业务逻辑.  
> 领域层(Domain): 负责表达业务概念,业务状态信息以及业务规则,是业务软件的核心.  
> 所以综合来看应用服务是用来表述应用行为，而领域服务用来表述领域行为。
> 那怎么理解应用行为和领域行为呢,应用行为描述了一个具体操作从开始到结束的每一个环节.  
> 领域行为是对应用行为的细化,用来处理具体的某一个环节.   
> 比如:手机购物,从购物车结算这一场景来举例,这就是一个应用行为.  
> 而这个应用行为又主要包括金额计算、支付、生成订单,这些子环节就可以理解为一个领域行为.
### (3). 什么是应用服务
> 应用服务是用来表达用例和用户故事(User Story)的主要手段.
> 应用层通过应用服务接口来暴露系统的全部功能.在应用服务的实现中,它负责*编排和转发*,它将要实现的功能委托给一个或多个领域对象来实现,它本身只负责处理业务用例的执行顺序以及结果的拼装.通过这样一种方式,它隐藏了领域层的复杂性及其内部实现机制.  
> 应用层相对来说是较“薄”的一层,除了定义应用服务之外,在该层我们可以进行:安全认证,权限校验,持久化事务控制,或者向其他系统发生基于事件的消息通知,另外还可以用于创建邮件以发送给客户等.    
> 应用层作为展现层与领域层的桥梁.*展现层使用VO(视图模型)进行界面展示,与应用层通过DTO(数据传输对象)进行数据交互,从而达到展现层与DO(领域对象)解耦的目的.*    
### (4). 什么是领域服务
> 领域层就是较“胖”的一层,因为它实现了全部"业务逻辑"并且通过"各种手段保证业务正确性".  
> 而什么是业务逻辑呢?业务流程、业务策略、业务规则、完整性约束等.  
> 当领域中的某个操作过程或转换过程不是实体或值对象的职责时,我们便应该将该操作放在一个单独的接口中,即领域服务.请确保该服务和通用语言时一致的;并且保证它是无状态的.  
### (5). 案例分析
> 转账问题来分析:  
> 1. 检查账号余额是否足够.   
> 2. 检查目标账户账号是否合法.   
> 3. 转账.   
> 4. 短信通知转账双方.   
> 其中1,2步是转账的合法性校验属于转账业务的一部分,所以,1,2,3均应该放到"领域层通过领域服务来实现".短信通知,它并不是是转账的核心业务,因为这根据具体情况而定,比如只有客户订阅了账号变动通知我才发短信.所以将第4步归类到应用服务中去实现,就确保了领域服务的纯粹性.  
> 领域逻辑应该只关心业务逻辑,才能保证领域逻辑的可重用性.而持久化放到应用层,我们就会有更多的选择性.  
### (6). 总结
> 1. 服务是行为的抽象.   
> 2. 应用服务通过委托:领域对象和领域服务来表达用例和用户故事.   
> 3. 领域对象(实体和值对象)负责单一操作.   
> 4. 领域服务用于协调多个领域对象共同完成某个业务操作.       
> 5. 应用服务不处理业务逻辑,领域服务处理业务逻辑.   