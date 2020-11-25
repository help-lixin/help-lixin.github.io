---
layout: post
title: 'Disruptor ExeceptionHandler'
date: 2019-10-07
author: 李新
tags: Disruptor
---

### (1). Disruptor为消费者配置异常处理

### (2). 定义业务模型(User)
```
package help.lixin.disruptor.pojo;

public class User {
	private Integer id;
	private String name;
	private Integer age;
	private String address;

	public Integer getId() {
		return id;
	}

	public void setId(Integer id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Integer getAge() {
		return age;
	}

	public void setAge(Integer age) {
		this.age = age;
	}

	public String getAddress() {
		return address;
	}

	public void setAddress(String address) {
		this.address = address;
	}

	@Override
	public String toString() {
		return "User [id=" + id + ", name=" + name + ", age=" + age + ", address=" + address + "]";
	}
}
```
### (3). 定义Event(UserEvent)
```
package help.lixin.disruptor.event;

public class UserEvent {
	private Integer id;
	private String name;
	private Integer age;
	private String address;

	public Integer getId() {
		return id;
	}

	public void setId(Integer id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Integer getAge() {
		return age;
	}

	public void setAge(Integer age) {
		this.age = age;
	}

	public String getAddress() {
		return address;
	}

	public void setAddress(String address) {
		this.address = address;
	}

	@Override
	public String toString() {
		return "UserEvent [id=" + id + ", name=" + name + ", age=" + age + ", address=" + address + "]";
	}
}

```
### (4). 定久Event工厂(UserEventFactory)
```
package help.lixin.disruptor.factory;

import com.lmax.disruptor.EventFactory;

import help.lixin.disruptor.event.UserEvent;

public class UserEventFactory implements EventFactory<UserEvent> {

	public UserEvent newInstance() {
		return new UserEvent();
	}
}

```
### (5). 定义EventHandler
> UserEventHandler1

```
package help.lixin.disruptor.consumer;

import java.util.concurrent.TimeUnit;

import com.lmax.disruptor.EventHandler;

import help.lixin.disruptor.event.UserEvent;

/**
 * 消费者一
 * @author lixin
 *
 */
public class UserEventHandler1 implements EventHandler<UserEvent> {

	@Override
	public void onEvent(UserEvent event, long sequence, boolean endOfBatch) throws Exception {
		System.out.println("1开始" + System.currentTimeMillis()+" ["+ Thread.currentThread().getName() +" ] UserEvent->" + event );
		if(event.getId() == 3) {
			throw new Exception("UserEventHandler1抛出异常.");
		}
		event.setAddress(event.getAddress() + "  Handler1");
		TimeUnit.SECONDS.sleep(1);
		System.out.println("1结束" + System.currentTimeMillis()+" ["+ Thread.currentThread().getName() +" ] UserEvent->" + event );
	}
}

```

> UserEventHandler2

```
package help.lixin.disruptor.consumer;

import java.util.concurrent.TimeUnit;

import com.lmax.disruptor.EventHandler;

import help.lixin.disruptor.event.UserEvent;

/**
 * 消费者一
 * @author lixin
 *
 */
public class UserEventHandler2 implements EventHandler<UserEvent> {

	@Override
	public void onEvent(UserEvent event, long sequence, boolean endOfBatch) throws Exception {
		System.out.println("2开始" + System.currentTimeMillis()+" ["+ Thread.currentThread().getName() +" ] UserEvent->" + event );
		if(event.getId() == 4) {
			throw new Exception("UserEventHandler2抛出异常.");
		}
		event.setAddress(event.getAddress() + "  Handler2");
		TimeUnit.SECONDS.sleep(2);
		System.out.println("2结束" + System.currentTimeMillis()+" ["+ Thread.currentThread().getName() +" ] UserEvent->" + event );
	}
}

```

> UserEventHandler3

```
package help.lixin.disruptor.consumer;

import java.util.concurrent.TimeUnit;

import com.lmax.disruptor.EventHandler;

import help.lixin.disruptor.event.UserEvent;

/**
 * 消费者一
 * @author lixin
 *
 */
public class UserEventHandler3 implements EventHandler<UserEvent> {

	@Override
	public void onEvent(UserEvent event, long sequence, boolean endOfBatch) throws Exception {
		System.out.println("3开始" + System.currentTimeMillis()+" ["+ Thread.currentThread().getName() +" ] UserEvent->" + event );
		if(event.getId() == 4) {
			throw new Exception("UserEventHandler3抛出异常.");
		}
		event.setAddress(event.getAddress() + "  Handler3");
		TimeUnit.SECONDS.sleep(3);
		System.out.println("3结束" + System.currentTimeMillis()+" ["+ Thread.currentThread().getName() +" ] UserEvent->" + event );
	}
}

```

> UserEventHandler4

```
package help.lixin.disruptor.consumer;

import java.util.concurrent.TimeUnit;

import com.lmax.disruptor.EventHandler;

import help.lixin.disruptor.event.UserEvent;

/**
 * 消费者一
 * @author lixin
 *
 */
public class UserEventHandler4 implements EventHandler<UserEvent> {

	@Override
	public void onEvent(UserEvent event, long sequence, boolean endOfBatch) throws Exception {
		System.out.println("4开始" + System.currentTimeMillis()+" ["+ Thread.currentThread().getName() +" ] UserEvent->" + event );
		if(event.getId() == 4) {
			throw new Exception("UserEventHandler4抛出异常.");
		}
		event.setAddress(event.getAddress() + "  Handler4");
		TimeUnit.SECONDS.sleep(4);
		System.out.println("4结束" + System.currentTimeMillis()+" ["+ Thread.currentThread().getName() +" ] UserEvent->" + event );
	}
}

```
### (6). 定义异常处理类
> 全局异常处理类(DefaultUserEventExceptionHandler)

```
package help.lixin.disruptor.exception;

import com.lmax.disruptor.ExceptionHandler;

import help.lixin.disruptor.event.UserEvent;

public class DefaultUserEventExceptionHandler implements ExceptionHandler<UserEvent> {

	@Override
	public void handleEventException(Throwable ex, long sequence, UserEvent event) {
		System.out.println("DefaultUserEventExceptionHandler.handleEventException ->" + event);
	}

	@Override
	public void handleOnStartException(Throwable ex) {
		System.out.println("DefaultUserEventExceptionHandler.handleOnStartException->" + ex.getMessage());
	}

	@Override
	public void handleOnShutdownException(Throwable ex) {
		System.out.println("DefaultUserEventExceptionHandler.handleOnShutdownException->" + ex.getMessage());
	}
}

```

> 为某个EventHandler定义异常处理类(UserEventExceptionHandler)

```
package help.lixin.disruptor.exception;

import com.lmax.disruptor.ExceptionHandler;

import help.lixin.disruptor.event.UserEvent;

public class UserEventExceptionHandler implements ExceptionHandler<UserEvent>{

	@Override
	public void handleEventException(Throwable ex, long sequence, UserEvent event) {
		System.out.println("UserEventExceptionHandler.handleEventException ->" + event);
	}

	@Override
	public void handleOnStartException(Throwable ex) {
		System.out.println("UserEventExceptionHandler.handleOnStartException->" + ex.getMessage());
	}

	@Override
	public void handleOnShutdownException(Throwable ex) {
		System.out.println("UserEventExceptionHandler.handleOnShutdownException->" + ex.getMessage());
	}
}

```
### (7). 组合测试
```
package help.lixin.disruptor;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;

import help.lixin.disruptor.consumer.UserEventHandler1;
import help.lixin.disruptor.consumer.UserEventHandler2;
import help.lixin.disruptor.consumer.UserEventHandler3;
import help.lixin.disruptor.consumer.UserEventHandler4;
import help.lixin.disruptor.event.UserEvent;
import help.lixin.disruptor.exception.DefaultUserEventExceptionHandler;
import help.lixin.disruptor.exception.UserEventExceptionHandler;
import help.lixin.disruptor.factory.UserEventFactory;
import help.lixin.disruptor.pojo.User;
import help.lixin.disruptor.producer.UserEventProducer;

public class UserEventTest1 {
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

		// 指定两个消费者(并行)
		disruptor.handleEventsWith(handler1);
		disruptor.handleEventsWith(handler2);

		// 设置全局的异常处理器类
		disruptor.setDefaultExceptionHandler(new DefaultUserEventExceptionHandler());

		// 为消费者(handler1)指定异常处理类为:UserEventExceptionHandler
		// 如果有为某个消费者指定异常处理类,则会覆盖全局的异常处理类.
		disruptor.handleExceptionsFor(handler1).with(new UserEventExceptionHandler());

		// 该方法只能调用一次,并且所有的EventHandler必须在start之前添加,包括:ExeceptionHandler
		disruptor.start();

		RingBuffer<UserEvent> ringBuffer = disruptor.getRingBuffer();
		UserEventProducer producer = new UserEventProducer(ringBuffer);

		ExecutorService executorService = Executors.newCachedThreadPool();
		int count = 5;
		for (int i = 0; i < count; i++) {
			final int id = i;
			executorService.submit(() -> {
				User user = new User();
				user.setId(id);
				user.setName("lixin " + id);
				user.setAge(id + 10);
				user.setAddress("深圳 " + id);
				producer.publish(user);
			});
		}
		TimeUnit.SECONDS.sleep(1000000);
		disruptor.shutdown();
	}
}

```
### (8). 总结
> Disruptor可以为(消费者)EventHandler指定异常处理类.可以全局(disruptor.setDefaultExceptionHandler)指定,也可以单独为EventHandler(disruptor.handleExceptionsFor(handler1).with(new UserEventExceptionHandler()))进行指定(单独指定的会覆盖全局的). 