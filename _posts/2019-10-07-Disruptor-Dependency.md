---
layout: post
title: 'Disruptor Dependency依赖处理'
date: 2019-10-07
author: 李新
tags: Disruptor
---

### (1). Disruptor依赖处理

### (2). 组合测试
```
package help.lixin.disruptor;

import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.EventHandlerGroup;

import help.lixin.disruptor.consumer.UserEventHandler1;
import help.lixin.disruptor.consumer.UserEventHandler2;
import help.lixin.disruptor.consumer.UserEventHandler3;
import help.lixin.disruptor.consumer.UserEventHandler4;
import help.lixin.disruptor.event.UserEvent;
import help.lixin.disruptor.exception.DefaultUserEventExceptionHandler;
import help.lixin.disruptor.factory.UserEventFactory;
import help.lixin.disruptor.pojo.User;
import help.lixin.disruptor.producer.UserEventProducer;

public class UserEventTest2 {
	public static void main(String[] args) throws Exception {
		AtomicInteger inc = new AtomicInteger(1);
		// 1. 初始化线程池工厂
		ThreadFactory factory = (r) -> {
			int index = inc.getAndIncrement();
			Thread t = new Thread(r);
			t.setName("disruptor thread(" + index + ")");
			return t;
		};

		// 2. 初始化RingBuffer的大小,必须是2的指数
		int bufferSize = 1024;

		// 3.Event处理器(消费者)
		UserEventHandler1 handler1 = new UserEventHandler1();
		UserEventHandler2 handler2 = new UserEventHandler2();
		UserEventHandler3 handler3 = new UserEventHandler3();
		UserEventHandler4 handler4 = new UserEventHandler4();

		// 默认生产者为:多线程模式
		Disruptor<UserEvent> disruptor = new Disruptor<UserEvent>(new UserEventFactory(), bufferSize, factory);

		/**
		 * 指定两个消费者串行处理<br/>
		 * h1 -> h2
		 */
		// disruptor.handleEventsWith(handler1).handleEventsWith(handler2);

		/**
		 * 指定两个消费者并行消费(写法:1).在内存中处理的对象是同一个<br/>
		 * -> h1 <br/>
		 * -> h2 <br/>
		 */
		// disruptor.handleEventsWith(handler1);
		// disruptor.handleEventsWith(handler2);

		/**
		 * 指定两个消费者并行消费(写法:2).在内存中处理的对象是同一个.<br/>
		 * -> h1 <br/>
		 * -> h2 <br/>
		 */
		// disruptor.handleEventsWith(handler1,handler2);

		/**
		 * h1和h2并行,h3和h4串行(方式1)<br/>
		 * -> h1 <br/>
		 * &nbsp;&nbsp;&nbsp;&nbsp; -> h3 -> h4 <br/>
		 * -> h2<br/>
		 */
		 // disruptor.handleEventsWith(handler1, handler2).handleEventsWith(handler3).handleEventsWith(handler4);
		
		/**
		 * h1和h2并行,h3和h4串行(方式2)<br/>
		 * -> h1 <br/>
		 * &nbsp;&nbsp;&nbsp;&nbsp; -> h3 -> h4 <br/>
		 * -> h2 <br/>
		 */
		// EventHandlerGroup<UserEvent> group1 =  disruptor.handleEventsWith(handler1);
		// EventHandlerGroup<UserEvent> group2 =  disruptor.handleEventsWith(handler2);
		// group1.and(group2).handleEventsWith(handler3).handleEventsWith(handler4);
		
		/**
		 * h1和h2以及h3和h4先并行,然后再聚合到:h1<br/>
		 * -> h1 -> h2 <br/>
		 * &nbsp;&nbsp;&nbsp; -> h1 <br/>
		 * -> h3 -> h4 <br/>
		 */
		// EventHandlerGroup<UserEvent> group1 =  disruptor.handleEventsWith(handler1).handleEventsWith(handler2);
		// EventHandlerGroup<UserEvent> group2 =  disruptor.handleEventsWith(handler3).handleEventsWith(handler4);
		// group1和group2都执行完之后,才能交给handler1
		// group1.and(group2).handleEventsWith(handler1);
		
		/**
		 * 从h4开始,并行h2和h3,然后,聚合到h1
		 *  &nbsp;&nbsp;  -> h2 <br/>
		 * -> h4 <br/>  &nbsp;&nbsp;&nbsp;&nbsp;  -> h1 <br/> 
		 * &nbsp;&nbsp;   -> h3 <br/>
		 */
		disruptor.handleEventsWith(handler4).handleEventsWith(handler2,handler3).handleEventsWith(handler1);
		
		
		 // 设置全局的异常处理器类
		disruptor.setDefaultExceptionHandler(new DefaultUserEventExceptionHandler());

		// 为消费者(handler1)指定异常处理类为:UserEventExceptionHandler
		// 如果有为某个消费者指定异常处理类,则会覆盖全局的异常处理类.
		// disruptor.handleExceptionsFor(handler1).with(new
		// UserEventExceptionHandler());

		// 该方法只能调用一次,并且所有的EventHandler必须在start之前添加,包括:ExeceptionHandler
		disruptor.start();

		RingBuffer<UserEvent> ringBuffer = disruptor.getRingBuffer();
		UserEventProducer producer = new UserEventProducer(ringBuffer);
		int id = 1;
		User user = new User();
		user.setId(id);
		user.setName("lixin " + id);
		user.setAge(id + 10);
		user.setAddress("深圳" + id);
		producer.publish(user);
		TimeUnit.SECONDS.sleep(1000000);
		disruptor.shutdown();
	}
}
```

### (3). 总结
> Disruptor的依赖处理非常灵活,直接看代码一点点测试吧! 