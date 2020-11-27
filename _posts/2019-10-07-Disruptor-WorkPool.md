---
layout: post
title: 'Disruptor 在 Canal中是如何使用的'
date: 2019-10-07
author: 李新
tags: Disruptor Canal
---

### (1). 概述

> 下面的代码是来自于Canal,因为剖析Canal发现:Canal用到了Disruptor.    
> 去除掉业务代码,纯粹只是分析数据流转过程.   
> 疑问:针对同一个张表的操作,并行解析如何保证不乱序( 比如:a线程针对一条数据(price)增加了100,b线程针对这条数据执行了删除. )    
### (2).案例代码 
```
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
	public static void main(String[] args) {
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
		BatchEventProcessor<MessageEvent> sinkStoreStage = new BatchEventProcessor<MessageEvent>( 
				disruptorMsgBuffer,
				sinkSequenceBarrier, 
				new SinkStoreStage());
		sinkStoreStage.setExceptionHandler(exceptionHandler);
		disruptorMsgBuffer.addGatingSequences(sinkStoreStage.getSequence());

		// 定义:SimpleParserStage和SinkStoreStage(EventHandler)交给哪个线程处理
		stageExecutor.submit(simpleParserStage);
		stageExecutor.submit(sinkStoreStage);
		
		// 定义:DmlParserStage(WorkHandler)交给哪个线程池处理.
		workerPool.start(parserExecutor);
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
		// 消费者处理
		// TODO
	}
}

/**
 * DML解析(WorkHandler:为多线程消费者需要定义的基类)
 * @author lixin
 */
class DmlParserStage implements WorkHandler<MessageEvent> {
	@Override
	public void onEvent(MessageEvent event) throws Exception {
		// TODO
	}
}

class SinkStoreStage implements EventHandler<MessageEvent> {

	@Override
	public void onEvent(MessageEvent event, long sequence, boolean endOfBatch) throws Exception {
		// 消费者处理
		// TODO
	}
}

/**
 * 业务模型工厂.Disruptor在构建时,会创建一个环形队列,队列里的业务模型就是通过该工厂创建的
 * @author lixin
 */
class MessageEventFactory implements EventFactory<MessageEvent> {
	@Override
	public MessageEvent newInstance() {
		return new MessageEvent();
	}
}

/**
 * 业务模型
 * @author lixin
 */
class MessageEvent {
}

/**
 * 异常处理
 * @author lixin
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


### (4). 总结
> 1. SimpleParserStage(单线程解析)   
> 2. DmlParserStage(多线程解析)   
> 3. SinkStoreStage(单线程解析)    
