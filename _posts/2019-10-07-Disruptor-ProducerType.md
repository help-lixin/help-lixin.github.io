---
layout: post
title: 'Disruptor 生产者类型配置'
date: 2019-10-07
author: 李新
tags: Disruptor
---

### (1). Disruptor ProducerType配置
> Disruptor可以根据业务场景(生产者)来指定:ProducerType.SINGLE / ProducerType.MULTI参数,用来控制序列器(sequence)的生成模式,默认为:ProducerType.MULTI   

### (2). 组合测试
```
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import com.lmax.disruptor.BlockingWaitStrategy;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;

public class DisruptorTest3 {
	public static void main(String[] args) throws Exception {
		AtomicInteger inc = new AtomicInteger(1);
		// 1. 初始化线程池工厂s
		ThreadFactory factory = (r) -> {
			int index = inc.getAndIncrement();
			Thread t = new Thread(r);
			t.setName("disruptor " + index);
			return t;
		};

		// 2. 初始化RingBuffer的大小,必须是2的指数
		int bufferSize = 1024;

		// 3.Event处理器(消费者)
		LongEventHandler consumerHandler = new LongEventHandler();

		// 默认生产者为:多线程模式
//		Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(LongEvent.EVENT_FACTORY, bufferSize, factory);
		// 多线程模式(多线程模式情况下,会存在吞吐量有所下降,但能保证不丢失数据)
//		Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(LongEvent.EVENT_FACTORY, bufferSize, factory,ProducerType.MULTI,new BlockingWaitStrategy());
		// 单线程模式(注意:单线程模式情况下,却开启了多线程,会存在丢数据的可能性)
		Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(LongEvent.EVENT_FACTORY, bufferSize, factory,
				ProducerType.SINGLE, new BlockingWaitStrategy());
		
		// 指定消费者
		disruptor.handleEventsWith(consumerHandler);
		// 该方法只能调用一次,并且所有的EventHandler必须在start之前添加,包括:ExeceptionHandler
		disruptor.start();

		RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
		// 自定义生产者
		LongEventProducer producer = new LongEventProducer(ringBuffer);

		int count = 10;
		final CyclicBarrier barrier = new CyclicBarrier(count);
		ExecutorService executor = Executors.newCachedThreadPool();
		for (long i = 0; i < count; i++) {
			final long data = i;
			executor.submit(() -> {
				try {
					System.out.println(data + " is ready...");
					barrier.await();
					producer.publish(data);
				} catch (Exception e) {
					e.printStackTrace();
				}
			});
		}
		TimeUnit.SECONDS.sleep(1000000);
		disruptor.shutdown();
	}
}

```
### (4). 总结
> 如果业务场景(生产者)是单线程模式下,生产者模式<font color='red'>可以</font>调整为:ProducerType.SINGLE,可以显著提高性能,因为不用处理并发模式下sequence的产生.   
> <font color='red'>如果业务场景(生产者)是多线程模式下,而生产者模式为:ProducerType.MULTI,则会出现sequence丢失的情况(数据丢失或者覆盖掉了).</font>   

> <font color='red'>总结:除非确保生产者是单线程模式,否则,尽量用多线程模式</font>  