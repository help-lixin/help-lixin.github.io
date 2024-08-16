---
layout: post
title: 'Transaction Outbox源码剖析' 
date: 2024-08-16
author: 李新
tags:  transaction-outbox
---

### (1). 概述

最近休息，对事务消息发件箱模式比较感兴趣，闲逛github时，发现有向我推一个新的框架，特意拿下来看下源码。

### (2). Transaction Outbox优势

跨平台数据库的支持,比较轻量级，对于事务消息同步到其它存储设备(比如:MQ)的话，是需要自己去实现的。

### (3). Transaction Outbox类图

![Transaction Outbox类图](/assets/transaction-outbox/images/TransactionOutbox.jpg)

### (4). Transaction Outbox官方案例
```
@RestController
class EventuallyConsistentController {

  private static final Logger LOGGER =
      LoggerFactory.getLogger(EventuallyConsistentController.class);

  @Autowired private CustomerRepository customerRepository;
  @Autowired private TransactionOutbox outbox;
  @Autowired private EventRepository eventRepository;
  @Autowired private EventPublisher eventPublisher;

  @SuppressWarnings("SameReturnValue")
  @RequestMapping("/createCustomer")
  @Transactional
  public String createCustomer() {
    LOGGER.info("Creating customers");
	// ***************************************************************************
	// schedule实际是对:EventPublisher类，进行AOP来着的
	// ***************************************************************************
    outbox
        .schedule(EventPublisher.class) // Just a trick to get autowiring to work.
        .publish(1L, "Created customers", LocalDateTime.now());
    customerRepository.save(new Customer(1L, "Martin", "Carthy"));
    customerRepository.save(new Customer(2L, "Dave", "Pegg"));
    LOGGER.info("Customers created");
    return "Done";
  }
} // end  EventuallyConsistentController



@Service
class EventPublisher {

  @Autowired 
  private EventRepository eventRepository;

  public void publish(long id, String description, LocalDateTime time) {
    eventRepository.save(new Event(id, description, time));
  }
} // end EventPublisher
```

### (5). Transaction Outbox大体原理
```
1. 对业务对象:EventPublisher进行AOP代理。
2. 调用业务对象的方法:publish(1L, "Created customers", LocalDateTime.now())，实际被AOP进行了处理来着的
3. 把方面业务对象的方法，参数类型，参数，进行JSON序列化
4. 创建TransactionOutboxEntry对象，包裹着上面的JSON以及一些其它信息。
5. 调用:Persistor.save方法，把TransactionOutboxEntry进行持久化。
6. 为：Transaction对象，添加Hook方法，即:事务提交后触发事件。
7. Hook方法进行两件事情,其中之一为：回调：TransactionOutboxListener
8. 另一个则是：回调：Submitter
```

### (6). Transaction Outbox部份源码摘要
```
private <T> T schedule(Class<T> clazz, String uniqueRequestId) {
    if (!initialized.get()) {
      throw new IllegalStateException("Not initialized");
    }
    return proxyFactory.createProxy(
        clazz,
        (method, args) ->
            uncheckedly(
                () -> {
				// **************************************************************
				// 1. 对方法和参数通过：TransactionalInvocation对象进行包裹
				// **************************************************************  
                  var extracted = transactionManager.extractTransaction(method, args);
				  
				  // **************************************************************
				  // 2. 创建：TransactionOutboxEntry对象
				  // **************************************************************  
                  TransactionOutboxEntry entry =
                      newEntry(
                          extracted.getClazz(),
                          extracted.getMethodName(),
                          extracted.getParameters(),
                          extracted.getArgs(),
                          uniqueRequestId);
                  validator.validate(entry);
				  
				  // **************************************************************  
				  // 3. 把TransactionOutboxEntry对象进行持久化(注意：与业务是在同一个事务之内来着的)
				  // **************************************************************  
                  persistor.save(extracted.getTransaction(), entry);
				  
                  extracted
                      .getTransaction()
                      .addPostCommitHook(
                          () -> {
							// **************************************************************  
							// 4. 回调:TransactionOutboxListener
							// **************************************************************  
                            listener.scheduled(entry);
							
							// **************************************************************
							// 5. 回调:Submitter,注意:当方法调用成功后，
							// 5.1 会剔除：TXNO_OUTBOX表中的数据来着的。
							// **************************************************************  
                            submitNow(entry);
                          });
                  log.debug(
                      "Scheduled {} for running after transaction commit", entry.description());
                  return null;
                }));
  }
```


TXNO_OUTBOX表结构

```
CREATE TABLE `TXNO_OUTBOX` (
  `id` varchar(36) NOT NULL,
  `invocation` mediumtext,
  `lastAttemptTime` timestamp(6) NULL DEFAULT NULL,
  `nextAttemptTime` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `attempts` int(11) DEFAULT NULL,
  `blocked` varchar(250) DEFAULT NULL,
  `version` int(11) DEFAULT NULL,
  `uniqueRequestId` varchar(250) DEFAULT NULL,
  `processed` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniqueRequestId` (`uniqueRequestId`),
  KEY `IX_TXNO_OUTBOX_1` (`processed`,`blocked`,`nextAttemptTime`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```