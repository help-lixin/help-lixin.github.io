---
layout: post
title: 'Disruptor RingBuffer组装依赖'
date: 2019-10-07
author: 李新
tags: Disruptor
---

### (1). RingBuffer组装依赖

> 通过代码:RingBuffer来组装EventHandler.

### (2). 组装代码
```
package help.lixin.disruptor;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import com.lmax.disruptor.BatchEventProcessor;
import com.lmax.disruptor.BlockingWaitStrategy;
import com.lmax.disruptor.ExceptionHandler;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.SequenceBarrier;

import help.lixin.disruptor.consumer.UserEventHandler1;
import help.lixin.disruptor.consumer.UserEventHandler2;
import help.lixin.disruptor.event.UserEvent;
import help.lixin.disruptor.factory.UserEventFactory;
import help.lixin.disruptor.pojo.User;
import help.lixin.disruptor.producer.UserEventProducer;

public class UserEventTest3 {
	public static void main(String[] args) throws Exception {
		/**
		 * SequenceBarrier sequenceBarrier1 = ringBuffer.newBarrier(); <br/>
		 * SequenceBarrier sequenceBarrier2 = ringBuffer.newBarrier(eventProcessorH1.getSequence()); <br/>
		 * 最终无非不过就是两个EvenHandler串行处理,如下图:
		 * -> h1 -> h2 <br/>
		 */
		
		// 1. 初始化RingBuffer的大小,必须是2的指数
		int bufferSize = 1024;

		// 2.Event处理器(消费者)
		UserEventHandler1 h1 = new UserEventHandler1();
		UserEventHandler2 h2 = new UserEventHandler2();

		// 3. 创建RingBuffer
		RingBuffer<UserEvent> ringBuffer = RingBuffer.createMultiProducer( //
				new UserEventFactory(), //
				bufferSize, //
				new BlockingWaitStrategy());
		// 
		SequenceBarrier sequenceBarrier1 = ringBuffer.newBarrier();
		BatchEventProcessor<UserEvent> eventProcessorH1 = new BatchEventProcessor<UserEvent>(ringBuffer,
				sequenceBarrier1, h1);
		eventProcessorH1.setExceptionHandler(new ExceptionHandler<UserEvent>() {
			@Override
			public void handleEventException(Throwable ex, long sequence, UserEvent event) {
			}

			@Override
			public void handleOnStartException(Throwable ex) {
			}

			@Override
			public void handleOnShutdownException(Throwable ex) {
			}
		});

		SequenceBarrier sequenceBarrier2 = ringBuffer.newBarrier(eventProcessorH1.getSequence());
		BatchEventProcessor<UserEvent> eventProcessorH2 = new BatchEventProcessor<UserEvent>(ringBuffer,
				sequenceBarrier2, h2);
		eventProcessorH2.setExceptionHandler(new ExceptionHandler<UserEvent>() {
			@Override
			public void handleEventException(Throwable ex, long sequence, UserEvent event) {
			}

			@Override
			public void handleOnStartException(Throwable ex) {
			}

			@Override
			public void handleOnShutdownException(Throwable ex) {
			}
		});
		ringBuffer.addGatingSequences(eventProcessorH2.getSequence());
		
		// 5. 创建线程池
		ExecutorService pool = Executors.newCachedThreadPool();
		// 把消息处理器提交给线程池
		pool.execute(eventProcessorH1);
		pool.execute(eventProcessorH2);

		UserEventProducer producer = new UserEventProducer(ringBuffer);
		ExecutorService executorService = Executors.newCachedThreadPool();
		// 只定义一个,方便查看结果
		int count = 1;
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
	}
}
```
### (3). 输出结果
```
# 1处理完之后,交给2处理.
1开始1606320078710 [pool-1-thread-1 ] UserEvent->UserEvent [id=0, name=lixin 0, age=10, address=深圳 0]
1结束1606320079715 [pool-1-thread-1 ] UserEvent->UserEvent [id=0, name=lixin 0, age=10, address=深圳 0  Handler1]

2开始1606320079715 [pool-1-thread-2 ] UserEvent->UserEvent [id=0, name=lixin 0, age=10, address=深圳 0  Handler1]
2结束1606320079715 [pool-1-thread-2 ] UserEvent->UserEvent [id=0, name=lixin 0, age=10, address=深圳 0  Handler1  Handler2]

```
### (4). 总结
> 多个Barrier之间,若有依赖,则属于串行运行.
