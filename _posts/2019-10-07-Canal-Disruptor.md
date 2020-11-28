---
layout: post
title: 'Canal中是如何使用Disruptor的'
date: 2019-10-07
author: 李新
tags: Disruptor Canal
---

### (1). 概述

> 下面的代码是来自于Canal,因为剖析Canal发现:Canal用到了Disruptor.    
> 去除掉业务代码,纯粹只是分析数据流转过程.   

### (2).案例代码 
```
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import com.alibaba.otter.canal.common.utils.NamedThreadFactory;
import com.lmax.disruptor.BatchEventProcessor;
import com.lmax.disruptor.BlockingWaitStrategy;
import com.lmax.disruptor.EventFactory;
import com.lmax.disruptor.EventHandler;
import com.lmax.disruptor.ExceptionHandler;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.Sequence;
import com.lmax.disruptor.SequenceBarrier;
import com.lmax.disruptor.WorkHandler;
import com.lmax.disruptor.WorkerPool;

public class DisruptorTest {
	public static void main(String[] args) throws Exception {
		// 定义解析线程总数量
		int parserThreadCount = 2;
		// 定义实例
		String destination = "example";
		int tc = parserThreadCount > 0 ? parserThreadCount : 1;
		RingBuffer<MessageEvent> disruptorMsgBuffer = RingBuffer.createSingleProducer(
				// 定义线程池工厂
				new MessageEventFactory(),
				// RingBuffer大小
				1024,
				// 生产者和消费者策略
				new BlockingWaitStrategy());

		// 定义:DmlParserStage线程池(WorkHandler)
		ExecutorService parserExecutor = Executors.newFixedThreadPool(tc,
				new NamedThreadFactory("MultiStageCoprocessor-Parser-" + destination));

		// 定义:SimpleParserStage和SinkStoreStage线程池(EventHandler)
		ExecutorService stageExecutor = Executors.newFixedThreadPool(2,
				new NamedThreadFactory("MultiStageCoprocessor-other-" + destination));

		// 创建异常处理
		ExceptionHandler exceptionHandler = new SimpleFatalExceptionHandler();

		// 创建屏障
		SequenceBarrier sequenceBarrier = disruptorMsgBuffer.newBarrier();
		// 定义事件处理器
		BatchEventProcessor<MessageEvent> simpleParserStage = new BatchEventProcessor<MessageEvent>(
				// RingBuffer
				disruptorMsgBuffer,
				// SequenceBarrier
				sequenceBarrier,
				// EventHandler
				new SimpleParserStage());

		// 创建屏障
		SequenceBarrier dmlParserSequenceBarrier = disruptorMsgBuffer.newBarrier(simpleParserStage.getSequence());

		// 创建多个消费者组(WorkHandler)
		WorkHandler<MessageEvent>[] workHandlers = new DmlParserStage[tc];
		for (int i = 0; i < tc; i++) {
			workHandlers[i] = new DmlParserStage();
		} // end for

		// 创建WorkPool
		WorkerPool<MessageEvent> workerPool = new WorkerPool<MessageEvent>(
				// DataProvider
				disruptorMsgBuffer,
				// SequenceBarrier
				dmlParserSequenceBarrier,
				// 异常处理
				exceptionHandler,
				// WorkHandler数组
				workHandlers);

		// 从WorkPool里获得所有的:Sequence
		Sequence[] sequence = workerPool.getWorkerSequences();
		// 添加:Sequence与RingBuffer的关系,因为:Sequence需要实时向:RingBuffer实时汇报处理情况.
		disruptorMsgBuffer.addGatingSequences(sequence);

		// stage 4
		SequenceBarrier sinkSequenceBarrier = disruptorMsgBuffer.newBarrier(sequence);
		BatchEventProcessor<MessageEvent> sinkStoreStage = new BatchEventProcessor<MessageEvent>(disruptorMsgBuffer,
				sinkSequenceBarrier, new SinkStoreStage());
		sinkStoreStage.setExceptionHandler(exceptionHandler);
		disruptorMsgBuffer.addGatingSequences(sinkStoreStage.getSequence());

		// 定义:SimpleParserStage和SinkStoreStage(EventHandler)交给哪个线程处理
		stageExecutor.submit(simpleParserStage);
		stageExecutor.submit(sinkStoreStage);

		// 定义:DmlParserStage(WorkHandler)交给哪个线程池处理.
		workerPool.start(parserExecutor);

		MessageEventProducer producer = new MessageEventProducer(disruptorMsgBuffer);
		CountDownLatch latch = new CountDownLatch(1);
		long count = 10;
		for (long i = 1; i <= count; i++) {
			producer.publish(i);
		}
		latch.await();
	}
}

/**
* 自定义生产者
*/
class MessageEventProducer {
	private final RingBuffer<MessageEvent> ringBuffer;

	public MessageEventProducer(RingBuffer<MessageEvent> ringBuffer) {
		this.ringBuffer = ringBuffer;
	}

	public void publish(Long value) {
		long sequence = ringBuffer.next();
		try {
			MessageEvent event = ringBuffer.get(sequence);
			event.setValue(value);
		} finally {
			ringBuffer.publish(sequence);
		}
	}
}

/**
 * 简单解析
 * 
 * @author lixin
 */
class SimpleParserStage implements EventHandler<MessageEvent> {
	@Override
	public void onEvent(MessageEvent event, long sequence, boolean endOfBatch) throws Exception {
		System.out.println("["+Thread.currentThread().getName()+"] process event before:[" + event + "]");
		event.setValue(event.getValue() + 1);
		System.out.println("["+Thread.currentThread().getName()+"] ++++SimpleParserStage++++ Process: " + event);
	}
}

/**
 * DML解析(WorkHandler:为多线程消费者需要定义的基类)
 * 
 * @author lixin
 *
 */
class DmlParserStage implements WorkHandler<MessageEvent> {
	@Override
	public void onEvent(MessageEvent event) throws Exception {
		event.setValue(event.getValue() + 1);
		System.out.println("["+Thread.currentThread().getName()+"] ----DmlParserStage---- Process: " + event);
	}
}

class SinkStoreStage implements EventHandler<MessageEvent> {

	@Override
	public void onEvent(MessageEvent event, long sequence, boolean endOfBatch) throws Exception {
		System.out.println("[" + Thread.currentThread().getName() + "] ****SinkStoreStage**** Event:" + event);
	}
}

/**
 * 业务模型工厂.Disruptor在构建时,会创建一个环形队列,队列里的业务模型就是通过该工厂创建的
 * 
 * @author lixin
 *
 */
class MessageEventFactory implements EventFactory<MessageEvent> {
	@Override
	public MessageEvent newInstance() {
		return new MessageEvent();
	}
}

/**
 * 业务模型
 * 
 * @author lixin
 *
 */
class MessageEvent {
	private Long value;

	public Long getValue() {
		return value;
	}

	public void setValue(Long value) {
		this.value = value;
	}

	@Override
	public String toString() {
		return "MessageEvent [value=" + value + "]";
	}
}

/**
 * 异常处理
 * 
 * @author lixin
 *
 */
class SimpleFatalExceptionHandler implements ExceptionHandler<MessageEvent> {
	@Override
	public void handleEventException(Throwable ex, long sequence, MessageEvent event) {
	}

	@Override
	public void handleOnStartException(Throwable ex) {
	}

	@Override
	public void handleOnShutdownException(Throwable ex) {

	}
}

```
### (3). 事件依赖图

!["Canal Disruptor事件依赖图"](/assets/disruptor/images/canal-disruptor-dapper.png)


### (4). 运行结果
```
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=1]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=2]
[MultiStageCoprocessor-Parser-example-0] ----DmlParserStage---- Process: MessageEvent [value=3]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=3]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=2]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=3]
[MultiStageCoprocessor-Parser-example-1] ----DmlParserStage---- Process: MessageEvent [value=4]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=4]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=3]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=4]
[MultiStageCoprocessor-Parser-example-0] ----DmlParserStage---- Process: MessageEvent [value=5]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=5]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=4]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=5]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=5]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=6]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=6]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=7]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=7]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=8]
[MultiStageCoprocessor-Parser-example-1] ----DmlParserStage---- Process: MessageEvent [value=6]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=6]
[MultiStageCoprocessor-Parser-example-0] ----DmlParserStage---- Process: MessageEvent [value=7]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=7]
[MultiStageCoprocessor-Parser-example-1] ----DmlParserStage---- Process: MessageEvent [value=8]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=8]
[MultiStageCoprocessor-Parser-example-0] ----DmlParserStage---- Process: MessageEvent [value=9]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=9]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=8]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=9]
[MultiStageCoprocessor-Parser-example-1] ----DmlParserStage---- Process: MessageEvent [value=10]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=10]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=9]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=10]
[MultiStageCoprocessor-other-example-0] process event before:[MessageEvent [value=10]]
[MultiStageCoprocessor-other-example-0] ++++SimpleParserStage++++ Process: MessageEvent [value=11]
[MultiStageCoprocessor-Parser-example-1] ----DmlParserStage---- Process: MessageEvent [value=12]
[MultiStageCoprocessor-Parser-example-0] ----DmlParserStage---- Process: MessageEvent [value=11]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=11]
[MultiStageCoprocessor-other-example-1] ****SinkStoreStage**** Event:MessageEvent [value=12]

```

### (5). 总结
> 1. 我们根据日志来分析(如遇不相信,可运行多次):   
> 2. MultiStageCoprocessor-other-example中有两个线程分别是:DmlParserStage/SinkStoreStage来运行业务代码.   
> 3. MultiStageCoprocessor-Parser-example有两个线程是在<font color='red'>并发运行</font>,有人就会有疑问了,先串行,再并发,然后再串行,<font color='red'>数据不会乱序吗?</font>   
> 4. 结论:   
>    Canal利用了Disruptor的依赖关系(编排多个Stage).在生产者与RingBuffer,以及消费者与消费者之间是存在一个东西叫:SequenceBarrier.<font color='red'>最后一个消费者(SinkStoreStage)会按照sequence的顺序来执行.Canal巧妙的利用了Disruptor来解决了,既并行,又不乱序的问题.</font>    
      我在这里不做太多解释,请参考( http://ifeve.com/disruptor-writing-ringbuffer/  )   