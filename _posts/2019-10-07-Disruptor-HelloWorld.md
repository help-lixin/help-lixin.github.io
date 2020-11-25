---
layout: post
title: 'Disruptor 入门'
date: 2019-10-07
author: 李新
tags: Disruptor
---

### (1). Disruptor开发模型
> 参考网站(https://www.cnblogs.com/crazymakercircle/p/13909235.html)  

> 1. 定义Event,代表Disruptor所能处理的数据单元.  
> 2. 定义Event工厂,实现EventFactory<?>接口,用来填充RingBuffer容器.  
> 3. 定义Event处理器(消费者),实现:EventHandler<?>接口,用来从RingBuffer中取出数据并处理.   
> 4. 组合(1~3步)   

### (2). 定义Event/Event工厂/Event消费者
```
import com.lmax.disruptor.EventFactory;
import com.lmax.disruptor.EventHandler;

// 1. 定义Event
public class LongEvent {
	private long value;

	public void set(long value) {
		this.value = value;
	}

	public long get() {
		return value;
	}

	// 2.定义Event工厂
	static EventFactory<LongEvent> EVENT_FACTORY = () -> {
		return new LongEvent();
	};
}

// 3.定义Event处理器(消费者)
class LongEventHandler implements EventHandler<LongEvent> {

	/**
	 * event:发布到RingBuffer中的事件. <br/>
	 * sequence:当前正在处理的事件序列号.<br/>
	 * endOfBatch:是否为RingBuffer的最后一个.<br/>
	 */
	@Override
	public void onEvent(LongEvent event, long sequence, boolean endOfBatch) throws Exception {
		System.out.println("[" + Thread.currentThread().getName() + "] LongEventHandler consumer->" + event);
	}
}
```
### (3). 组合测试
```
import java.util.Random;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;

public class DisruptorTest1 {
	public static void main(String[] args) {
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

		// 获取RingBuffer
		RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
		// 获取RingBuffer的下一个序号,序列相关于指针
		long sequence = ringBuffer.next();
		try {
			// 取出下标(指针)对应的数据
			LongEvent longEvent = ringBuffer.get(sequence);
			// 填充数据
			longEvent.set(new Random(1000).nextLong());
		} finally {
			// 发送数据对应的序号(指针)
			ringBuffer.publish(sequence);
		}
	}
}
```

### (3). 总结
> 看到了没有?生产者生产数据时:   
> 1. 是拿出sequence下标.   
> 2. 取出sequence对应的模板数据.     
> 3. <font color='red'>对sequence下标进行发布(发布的是sequence,而不是发布所谓的业务模型(LongEvent))</font>.    
>  所以,Disruptor,在启动时,会根据bufferSize创建N个LongEvent,放在环形队列里,而环形队列的下标(long)放在CPU 的缓存行中,每次生产消息时,根据下标(CPU缓存行)取出业务模型(LongEvent),对业务模型进行数据填充后,通知Disruptor对下标进行发布.   