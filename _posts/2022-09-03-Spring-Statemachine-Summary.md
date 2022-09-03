---
layout: post
title: 'Spring StateMachine订单案例入门' 
date: 2022-09-03
author: 李新
tags:  SpringStateMachine
---

### (1). 什么是状态机(Statemachine)
状态机是一种用来进行对象行为建模的工具,其作用主要是描述对象在它的生命周期内所经历的状态序列,以及如何响应来自外界的各种事件.在电商场景(订单、物流、售后)、社交(IM消息投递)、分布式集群管理(分布式计算平台任务编排)等场景都有大规模的使用.                     
Spring Statemachine是Spring官方提供的一个框架,供应用程序开发人员在Spring应用程序中使用状态机.支持状态的嵌套(substate),状态的并行(parallel,fork,join)、子状态机等等.   
### (2). 定义状态
```
public enum OrderStates {
	UNPAID, // 待支付
	WAITING_FOR_RECEIVE, // 待收货
	DONE // 结束
}
```
### (3). 定义事件
```
public enum OrderEvents {
	PAY, // 支付
	RECEIVE // 收货
}
```
### (4). 配置状态
```
@Configuration
@EnableStateMachine(name="orderSingleMachine")
public class StateMachineConfig extends EnumStateMachineConfigurerAdapter<OrderStates, OrderEvents> {

	private Logger logger = LoggerFactory.getLogger(getClass());

	@Override
	public void configure(StateMachineStateConfigurer<OrderStates, OrderEvents> states) throws Exception {
		states.withStates() // 初始状态
			  .initial(OrderStates.UNPAID)
				// 配置所有状态
			  .states(EnumSet.allOf(OrderStates.class));
	}

	@Override
	public void configure(StateMachineTransitionConfigurer<OrderStates, OrderEvents> transitions) throws Exception {
		transitions.withExternal()
				// 待支付 --> 待收货 (触发支付事件)
				.source(OrderStates.UNPAID).target(OrderStates.WAITING_FOR_RECEIVE).event(OrderEvents.PAY)
				.and()
				.withExternal()
				// 待收货 --> 结束(触发收货事件)
				.source(OrderStates.WAITING_FOR_RECEIVE).target(OrderStates.DONE).event(OrderEvents.RECEIVE);
	}
}
```
### (5). 配置状态转换的事件
```
@WithStateMachine(name="orderSingleMachine")
public class OrderSingleEventConfig {
private Logger logger = LoggerFactory.getLogger(getClass());
	
    /**
     * 当前状态UNPAID
     */
    @OnTransition(target = "UNPAID")
    public void create() {
        logger.info("---订单创建，待支付---");
    }
    
    /**
     * UNPAID->WAITING_FOR_RECEIVE 执行的动作
     */
    @OnTransition(source = "UNPAID", target = "WAITING_FOR_RECEIVE")
    public void pay() {
        logger.info("---用户完成支付，待收货---");
    }
    
    /**
     * WAITING_FOR_RECEIVE->DONE 执行的动作
     */
    @OnTransition(source = "WAITING_FOR_RECEIVE", target = "DONE")
    public void receive() {
        logger.info("---用户已收货，订单完成---");
    }

}
```
### (6). Controller
```
@RestController
@RequestMapping("/statemachine")
public class StateMachineController {
	
	@Autowired
	private StateMachine orderSingleMachine;

	@RequestMapping("/testSingleOrderState")
	public void testSingleOrderState() throws Exception {

		// 创建流程
		orderSingleMachine.start();

		// 触发PAY事件
		orderSingleMachine.sendEvent(OrderEvents.PAY);

		// 触发RECEIVE事件
		orderSingleMachine.sendEvent(OrderEvents.RECEIVE);

		// 获取最终状态
		System.out.println("最终状态：" + orderSingleMachine.getState().getId());
	} // end testSingleOrderState
}
```
### (7). 依赖配置
```
<properties>
	<java.version>1.8</java.version>
	<spring-statemachine.version>2.0.1.RELEASE</spring-statemachine.version>
</properties>


<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.statemachine</groupId>
			<artifactId>spring-statemachine-bom</artifactId>
			<version>${spring-statemachine.version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>


<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.statemachine</groupId>
	<artifactId>spring-statemachine-starter</artifactId>
</dependency>
```
### (8). 测试
```
lixin-macbook:~ lixin$ curl http://localhost:9991/statemachine/testSingleOrderState

2022-09-03 13:45:34.733  INFO 1462 --- [nio-9991-exec-2] tConfig$$EnhancerBySpringCGLIB$$9994113f : ---订单创建，待支付---
2022-09-03 13:45:34.743  INFO 1462 --- [nio-9991-exec-2] o.s.s.support.LifecycleObjectSupport     : started org.springframework.statemachine.support.DefaultStateMachineExecutor@40751c49
2022-09-03 13:45:34.743  INFO 1462 --- [nio-9991-exec-2] o.s.s.support.LifecycleObjectSupport     : started DONE UNPAID WAITING_FOR_RECEIVE  / UNPAID / uuid=bf21a999-a468-4c4b-8502-3a653247a6be / id=null
2022-09-03 13:45:34.758  INFO 1462 --- [nio-9991-exec-2] tConfig$$EnhancerBySpringCGLIB$$9994113f : ---用户完成支付，待收货---
2022-09-03 13:45:34.760  INFO 1462 --- [nio-9991-exec-2] tConfig$$EnhancerBySpringCGLIB$$9994113f : ---用户已收货，订单完成---
最终状态：DONE
```
### (9). 总结
先有一个简单的入门,后面,会对源码进行剖析.