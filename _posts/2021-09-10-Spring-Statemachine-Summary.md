---
layout: post
title: 'Spring Statemachine入门' 
date: 2021-09-10
author: 李新
tags:  SpringStatemachine
---

### (1). 什么是状态机(Statemachine)
状态机是一种用来进行对象行为建模的工具,其作用主要是描述对象在它的生命周期内所经历的状态序列,以及如何响应来自外界的各种事件.在电商场景(订单、物流、售后)、社交(IM消息投递)、分布式集群管理(分布式计算平台任务编排)等场景都有大规模的使用.                     
Spring Statemachine是Spring官方提供的一个框架,供应用程序开发人员在Spring应用程序中使用状态机.支持状态的嵌套(substate),状态的并行(parallel,fork,join)、子状态机等等.   
### (2). Spring Statemachine 项目模块
```
lixin-macbook:spring-statemachine lixin$ tree -L 1
.
├── spring-statemachine-autoconfigure                         # 与Spring整合
├── spring-statemachine-bom                                   # 依赖构建物的清单
├── spring-statemachine-build-tests                           # 测试用例
├── spring-statemachine-cluster                               # 集群(已经被Spring Integration取代了)
├── spring-statemachine-core                                  # 核心模块
├── spring-statemachine-data                                  # 持久化操作
├── spring-statemachine-kryo                                  # 序列化
├── spring-statemachine-recipes                               # 
├── spring-statemachine-samples                               # 案例代码
├── spring-statemachine-starter                               # Spring Boot Starter
├── spring-statemachine-test                                 
├── spring-statemachine-uml                                   # 使用Eclipse Papyusr进行UML建模的支持
└── spring-statemachine-zookeeper
```
### (3). 演示案例
```
package com.example.orderservice;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.java.Log;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.statemachine.StateMachine;
import org.springframework.statemachine.config.EnableStateMachineFactory;
import org.springframework.statemachine.config.StateMachineConfigurerAdapter;
import org.springframework.statemachine.config.StateMachineFactory;
import org.springframework.statemachine.config.builders.StateMachineConfigurationConfigurer;
import org.springframework.statemachine.config.builders.StateMachineStateConfigurer;
import org.springframework.statemachine.config.builders.StateMachineTransitionConfigurer;
import org.springframework.statemachine.listener.StateMachineListenerAdapter;
import org.springframework.statemachine.state.State;
import org.springframework.statemachine.support.DefaultStateMachineContext;
import org.springframework.statemachine.support.StateMachineInterceptorAdapter;
import org.springframework.statemachine.transition.Transition;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import java.util.Date;
import java.util.Optional;
import java.util.UUID;

@SpringBootApplication
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

enum OrderEvents {
    FULFILL,   // 货物已到达
    PAY,       // 付款
    CANCEL    // 取消
}

enum OrderStates {
    SUBMITTED,    // 提交
    PAID,         // 支付
    FULFILLED,   // 订单完成
    CANCELLED    // 取消
}

// 定义Entity
@Entity(name = "ORDERS")
@Data
@NoArgsConstructor
@AllArgsConstructor
class Order {

    @Id
    @GeneratedValue
    private Long id;
    private Date datetime;

    private String state;

    public Order(Date d, OrderStates os) {
        this.datetime = d;
        this.setOrderState(os);
    }

    public OrderStates getOrderState() {
        return OrderStates.valueOf(this.state);
    }

    public void setOrderState(OrderStates s) {
        this.state = s.name();
    }

}

// JPA
interface OrderRepository extends JpaRepository<Order, Long> {
}


@Service
class OrderService {

    private final OrderRepository orderRepository;
    private final StateMachineFactory<OrderStates, OrderEvents> factory;

    // 定义Event的协议头Header
    private static final String ORDER_ID_HEADER = "orderId";

    OrderService(OrderRepository orderRepository, StateMachineFactory<OrderStates, OrderEvents> factory) {
        this.orderRepository = orderRepository;
        this.factory = factory;
    }

    Order byId(Long id) {
        return this.orderRepository.findOne(id);
    }

    Order create(Date when) {
        return this.orderRepository.save(new Order(when, OrderStates.SUBMITTED));
    }

    StateMachine<OrderStates, OrderEvents> pay(Long orderId, String paymentConfirmationNumber) {
        // 1. 根据orderId创建:StateMachine
        StateMachine<OrderStates, OrderEvents> sm = this.build(orderId);
        // 2. 构建事件
        Message<OrderEvents> paymentMessage = MessageBuilder.withPayload(OrderEvents.PAY)
                .setHeader(ORDER_ID_HEADER, orderId)
                .setHeader("paymentConfirmationNumber", paymentConfirmationNumber)
                .build();
       // 3. 发送消息
        sm.sendEvent(paymentMessage);
        // todo
        return sm;
    }

    StateMachine<OrderStates, OrderEvents> fulfill(Long orderId) {
        StateMachine<OrderStates, OrderEvents> sm = this.build(orderId);
        Message<OrderEvents> fulfillmentMessage = MessageBuilder.withPayload(OrderEvents.FULFILL)
                .setHeader(ORDER_ID_HEADER, orderId)
                .build();
        sm.sendEvent(fulfillmentMessage);
        return sm;
    }

    private StateMachine<OrderStates, OrderEvents> build(Long orderId) {
        Order order = this.orderRepository.findOne(orderId);
        String orderIdKey = Long.toString(order.getId());

        StateMachine<OrderStates, OrderEvents> sm = this.factory.getStateMachine(orderIdKey);
        sm.stop();
        sm.getStateMachineAccessor()
                .doWithAllRegions(sma -> {
                    sma.addStateMachineInterceptor(new StateMachineInterceptorAdapter<OrderStates, OrderEvents>() {
                        @Override
                        public void preStateChange(State<OrderStates, OrderEvents> state, Message<OrderEvents> message, Transition<OrderStates, OrderEvents> transition, StateMachine<OrderStates, OrderEvents> stateMachine) {
                            Optional.ofNullable(message).ifPresent(msg -> {
                                Optional.ofNullable(Long.class.cast(msg.getHeaders().getOrDefault(ORDER_ID_HEADER, -1L)))
                                        .ifPresent(orderId1 -> {
                                            Order order1 = orderRepository.findOne(orderId1);
                                            order1.setOrderState(state.getId());
                                            orderRepository.save(order1);
                                        });
                            });
                        }
                    });
                    sma.resetStateMachine(new DefaultStateMachineContext<>(order.getOrderState(), null, null, null));
                });
        sm.start();
        return sm;
    }
}

@Log
@Component
class Runner implements ApplicationRunner {

    private final OrderService orderService;

    Runner(OrderService orderService) {
        this.orderService = orderService;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // 创建订单
        Order order = this.orderService.create(new Date());
        // 支付订单
        StateMachine<OrderStates, OrderEvents> paymentStateMachine = orderService.pay(order.getId(), UUID.randomUUID().toString());
        log.info("after calling pay(): " + paymentStateMachine.getState().getId().name());
        log.info("order: " + orderService.byId(order.getId()));

        StateMachine<OrderStates, OrderEvents> fulfilledStateMachine = orderService.fulfill(order.getId());
        log.info("after calling fulfill(): " + fulfilledStateMachine.getState().getId().name());
        log.info("order: " + orderService.byId(order.getId()));
    }

}


@Log
@Configuration
@EnableStateMachineFactory
class SimpleEnumStatemachineConfiguration extends StateMachineConfigurerAdapter<OrderStates, OrderEvents> {
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderStates, OrderEvents> transitions) throws Exception {
        transitions
                .withExternal().source(OrderStates.SUBMITTED).target(OrderStates.PAID).event(OrderEvents.PAY)
                .and()
                .withExternal().source(OrderStates.PAID).target(OrderStates.FULFILLED).event(OrderEvents.FULFILL)
                .and()
                .withExternal().source(OrderStates.SUBMITTED).target(OrderStates.CANCELLED).event(OrderEvents.CANCEL)
                .and()
                .withExternal().source(OrderStates.PAID).target(OrderStates.CANCELLED).event(OrderEvents.CANCEL);
    }

    @Override
    public void configure(StateMachineStateConfigurer<OrderStates, OrderEvents> states) throws Exception {
        states
                .withStates()
                .initial(OrderStates.SUBMITTED)
                /*.stateEntry(OrderStates.SUBMITTED, context -> {
                    Long orderId = Long.class.cast(context.getExtendedState().getVariables().getOrDefault("orderId", -1L));
                    log.info("orderId is " + orderId + ".");
                    log.info("entering submitted state!");
                })*/
                .state(OrderStates.PAID)
                .end(OrderStates.FULFILLED)
                .end(OrderStates.CANCELLED);
    }

    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderStates, OrderEvents> config) throws Exception {

        StateMachineListenerAdapter<OrderStates, OrderEvents> adapter = new StateMachineListenerAdapter<OrderStates, OrderEvents>() {
            @Override
            public void stateChanged(State<OrderStates, OrderEvents> from, State<OrderStates, OrderEvents> to) {
                log.info(String.format("stateChanged(from: %s, to: %s)", from + "", to + ""));
            }
        };
        config.withConfiguration()
                .autoStartup(false)
                .listener(adapter);
    }
}
```
### (4). 代码下载
```
https://github.com/help-lixin/statemachine.git
```
### (5). 总结
["参考文献"](https://blog.csdn.net/qq_22076345/article/details/109266901)  