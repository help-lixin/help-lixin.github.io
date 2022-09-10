---
layout: post
title: 'COLA简单入门' 
date: 2022-09-10
author: 李新
tags:  COLA
---

### (1). 概述
前面对Spring StateMachine进行了源码分析,
总体来说:Spring StateMachine是一个有状态的状态机,
而我期望的是一个更轻一些的状态机,同过度娘了解到:COLA,所以,特意把COLA的源码拉下来,先进行一个简单的入门,然后,再深入源码. 
### (2). 添加依赖包
```
<dependency>
	<groupId>com.alibaba.cola</groupId>
	<artifactId>cola-component-statemachine</artifactId>
	<version>4.4.0-SNAPSHOT</version>
</dependency>
```
### (3). COLA简单案例
```
package com.alibaba.cola.test;

import com.alibaba.cola.statemachine.Action;
import com.alibaba.cola.statemachine.Condition;
import com.alibaba.cola.statemachine.StateMachine;
import com.alibaba.cola.statemachine.StateMachineFactory;
import com.alibaba.cola.statemachine.builder.StateMachineBuilder;
import com.alibaba.cola.statemachine.builder.StateMachineBuilderFactory;
import org.junit.Assert;
import org.junit.Test;

public class StateMachineTest {
    
	// 1. 定义状态机的唯一名称
    static String MACHINE_ID = "TestStateMachine";

	// 2. 定义所有的状态
    static enum States {
        STATE1, STATE2, STATE3, STATE4
    }

    // 3. 定义所有的事件
    static enum Events {
        EVENT1, EVENT2, EVENT3, EVENT4, INTERNAL_EVENT
    }

	// 4. 可以传递一个上下文
    static class Context{
        String operator = "frank";
        String entityId = "123465";
    }
	
	// 5. 自定义检查条件
	private Condition<Context> checkCondition() {
		return new Condition<Context>() {
			@Override
			public boolean isSatisfied(Context context) {
				System.out.println("Check condition : "+context);
				return true;
			}
		};
	} // end checkCondition
	
	// 6. 定义符合触发事件时执行的动作.
	private Action<States, Events, Context> doAction() {
		return (from, to, event, ctx)->{
			System.out.println(ctx.operator+" is operating "+ctx.entityId+" from:"+from+" to:"+to+" on:"+event);
		};
	} // end doAction
	
	
	// **********************************************************************
	// 7. 开始编排Source与Target和Event
	// **********************************************************************
    @Test
    public void testExternalNormal(){
        StateMachineBuilder<States, Events, Context> builder = StateMachineBuilderFactory.create();
        builder.externalTransition()
                .from(States.STATE1)
                .to(States.STATE2)
                .on(Events.EVENT1)
                .when(checkCondition())
                .perform(doAction());
        StateMachine<States, Events, Context> stateMachine = builder.build(MACHINE_ID);
		
		// 8. 触发事件(需要注意:此处要带上事件的Source状态)
        States target = stateMachine.fireEvent(States.STATE1, Events.EVENT1, new Context());
        Assert.assertEquals(States.STATE2, target);
    } // end testExternalNormal
}
```
### (4). 运行结果
```
Check condition : com.alibaba.cola.test.StateMachineTest$Context@2cfb4a64
frank is operating 123465 from:STATE1 to:STATE2 on:EVENT1
```
### (5). 总结
这个案例是官网的,通过运行,大概能知道COLA的大体原理,与Spring StateMachine不同的点在于,触发事件时,是需要带上源始状态的,也不需要配置初始状态.  

### (6). 源码学习目录
["COLA源码之Transition(一)"](2022/09/10/COLA-Transition.html) 