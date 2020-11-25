---
layout: post
title: 'Disruptor 自定义生产者的模板代码'
date: 2019-10-07
author: 李新
tags: Disruptor
---


### (1). 自定义生产者,方案(1)
> 自己抽象出发布数据模板. 

```
import com.lmax.disruptor.RingBuffer;

public class LongEventProducer {

	private final RingBuffer<LongEvent> ringBuffer;

	public LongEventProducer(RingBuffer<LongEvent> ringBuffer) {
		this.ringBuffer = ringBuffer;
	}

	public void publish(Long data) {
		long sequence = ringBuffer.next();
		try {
			LongEvent event = ringBuffer.get(sequence);
			event.set(data);
			System.out.println("[" + Thread.currentThread().getName() + "]  LongEventProducer product ->" + event);
		} finally {
			ringBuffer.publish(sequence);
		}
	}
}
```
### (2). 自定义生产者,方案(2)
> 自定义:EventTranslatorOneArg/EventTranslatorXXXX   

```
import java.nio.ByteBuffer;

import com.lmax.disruptor.EventTranslatorOneArg;
import com.lmax.disruptor.RingBuffer;

public class LongEventProducerWithTranslator {
	private final RingBuffer<LongEvent> ringBuffer;

	public LongEventProducerWithTranslator(RingBuffer<LongEvent> ringBuffer) {
		this.ringBuffer = ringBuffer;
	}


	private static final EventTranslatorOneArg<LongEvent, ByteBuffer> TRANSLATOR = new EventTranslatorOneArg<LongEvent, ByteBuffer>() {
      // ByteBuffer 为入参
      // LongEvent  为返回类型
		@Override
		public void translateTo(LongEvent event, long sequence, ByteBuffer buffer) {
			event.set(buffer.getLong(0));
		}
	};

	public void publish(ByteBuffer buffer) {
		ringBuffer.publishEvent(TRANSLATOR, buffer);
	}
}
```
### (3). 组合测试
```
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;

public class DisruptorTest2 {
	public static void main(String[] args) throws Exception {
		// 1. 初始化线程池-用户执行Consumer
		Executor executor = Executors.newCachedThreadPool();
		// 2. 初始化RingBuffer的大小,必须是2的指数
		int bufferSize = 1024;

		// 3.Event处理器(消费者)
		LongEventHandler consumerHandler = new LongEventHandler();

		Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(LongEvent.EVENT_FACTORY, bufferSize, executor);
		// 指定消费者
		disruptor.handleEventsWith(consumerHandler);
		// 该方法只能调用一次,并且所有的EventHandler必须在start之前添加,包括:ExeceptionHandler
		disruptor.start();

		RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
		LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);

		ByteBuffer bb = ByteBuffer.allocate(8);
		for (long l = 0; true; l++) {
			bb.putLong(0, l);
			producer.publish(bb);
			Thread.sleep(1000);
		}
	}
}
```

### (3). 总结
> 因为生产者代码都是模板样式,Disruptor允许自定义生产者.