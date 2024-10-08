---
layout: post
title: 'Axon入门' 
date: 2021-09-10
author: 李新
tags:  Axon
---

### (1). CQRS概述
为什么要选择CQRS模式来实现微服务?                     
在传统的应用结构中,应用通常要对在数据库中持久化的数据进行操作.通常为数据模型实体使用唯一的数据库,同时用于读取和写入.数据的设计由写入和更新操作驱动,以保持数据一致性.开发人员尝试使用范式设计的技术来最小化数据冗余.     
虽然以规范化的形式存储数据,但这可能对读取操作是不利的.为了提取一些数据,开发人员需要编写复杂的SQL查询,来从多个表中连接(join)数据.      
此外,在许多应用中,数据创建一次并且仅偶尔进行修改,可能读取多次.             
因此,开发人员需要特别注意读取性能,访问数据应该尽可能少的查询,并且每个查询期间执行的业务逻辑应该被最小化.这就是CQRS模式的由来.    

CQRS基本思想是将对领域对象的操作划分为两个不同的类型:
+ 查询 - 返回结果并且不更改系统状态的方法. 
+ 命令 - 更改系统状态但不返回值的方法.  

!["CQRS"](/assets/axon/imgs/cqrs-axon.png)

["爱奇艺Axon实践过程"](http://dockone.io/article/10711)

### (2). Axon是什么?

Axon Framework用来帮助开发人员构建基于命令查询责任分类(Command Query Responsibility Segregation: CQRS)设计模式的可伸缩、可扩展和可维护应用程序的框架.  

### (3). 案例代码
```
# AxonBank git仓库地址.
https://github.com/help-lixin/AxonBank.git

# AxonBank 项目结构图
lixin-macbook:AxonBank lixin$ tree -L 1
.
├── core                                  # Command Handler
├── core-api                              # Command
├── docker                                # Docker
├── docker-compose.yml
├── query                                 # Repository(查询使用)
└── web                                   # web界面
```

### (4). Axon架构图

!["Axon架构图"](/assets/axon/imgs/axon-detailed-architecture-overview.png)

### (5). Axon概念
+ Command(代表着要执行的操作)
+ CommandBus(接受Command,并路由给CommandHandler)  
+ CommandHandler(负责处理Command)  
+ Repository(提供仓储检索功能)
+ Aggregate(业务领域对象)
+ Event(Aggregate在状态变更时需要发送Domain Event来通知系统中的其他组件)    
+ EventBus(将事件分派给所有感兴趣的 Event Listener)  
+ EventHandler(接收并处理事件)   
+ EventStore(事件存储)   

### (6). AxonBank案例中的Command
!["AxonBank案例中的Command"](/assets/axon/imgs/axon-command-list.jpg)

### (7). CommandBus接口详解
!["CommandBus"](/assets/axon/imgs/axon-command-bus.jpg)

+ CommandBus               : 接口定义
+ RecordingCommandBus      : 只做记录
+ SimpleCommandBus         : CommandBus的实现,分配命令给订阅到特定命令类型的handler.
+ AsynchronousCommandBus   : 异步处理命令.
+ DisruptorCommandBus      : Disruptor性能非常高的异步CommandBus实现.  
+ DistributedCommandBus    : 分布式的CommandBus实现,由多个CommandBus的实例组成,并一起工作来分担负载.   

### (8). CommandHandler
```
// 1. 把CommandHandler交给Spring
@Bean
public BankAccountCommandHandler bankAccountCommandHandler() {
	return new BankAccountCommandHandler(axonConfiguration.repository(BankAccount.class), eventBus);
}

// 2. 定义Command Handler
public class BankAccountCommandHandler {

    private Repository<BankAccount> repository;
    private EventBus eventBus;

    public BankAccountCommandHandler(Repository<BankAccount> repository, EventBus eventBus) {
        this.repository = repository;
        this.eventBus = eventBus;
    }

	// 2. 给方法添加注解,并且,在注解上指定:Command
    @CommandHandler
    public void handle(DebitSourceBankAccountCommand command) {
        try {
            Aggregate<BankAccount> bankAccountAggregate = repository.load(command.getBankAccountId());
            bankAccountAggregate.execute(bankAccount -> bankAccount
                    .debit(command.getAmount(), command.getBankTransferId()));
        } catch (AggregateNotFoundException exception) {
            eventBus.publish(asEventMessage(new SourceBankAccountNotFoundEvent(command.getBankTransferId())));
        }
    }

    @CommandHandler
    public void handle(CreditDestinationBankAccountCommand command) {
        try {
            Aggregate<BankAccount> bankAccountAggregate = repository.load(command.getBankAccountId());
            bankAccountAggregate.execute(bankAccount -> bankAccount
                    .credit(command.getAmount(), command.getBankTransferId()));

        }
        catch (AggregateNotFoundException exception) {
            eventBus.publish(asEventMessage(new DestinationBankAccountNotFoundEvent(command.getBankTransferId())));
        }
    }
} // end BankAccountCommandHandler
```
### (9). EventHandler
```
@Component
public class BankAccountEventListener {
	
	// ***************************************************************
	// 事件的处理器
	// ***************************************************************
	@EventHandler
	public void on(BankAccountCreatedEvent event) {
		repository.save(new BankAccountEntry(event.getId(), 0, event.getOverdraftLimit()));
		
		broadcastUpdates();
	}
}	
```
### (10). 总结
先对Axon有一个概念的了解和入门,后续会对Axon进行源码剖析! 