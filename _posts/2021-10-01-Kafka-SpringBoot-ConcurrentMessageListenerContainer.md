---
layout: post
title: 'Spring Boot与Kafka集成源码之ConcurrentMessageListenerContainer(九)' 
date: 2021-10-01
author: 李新
tags:  Kafka
---

### (1). 概述
前面剖析到MessageListenerContainer接口,这个接口主要负责转化业务模型,并真正与Kafka进行通信,实现订阅功能,它有两个实现类,分别是:ConcurrentMessageListenerContainer和KafkaMessageListenerContainer.   
而,我比较好奇的一件事情就是:注解(@KafkaListener(concurrency = "2"))上配置了并发数,那么底层是线程池呢?还是怎么实现的?   

### (2). ConcurrentMessageListenerContainer
```
public class ConcurrentMessageListenerContainer<K, V> extends AbstractMessageListenerContainer<K, V> {
	// 内部持有N个KafkaMessageListenerContainer
	private final List<KafkaMessageListenerContainer<K, V>> containers = new ArrayList<>();
	// 并发数
	private int concurrency = 1;
	
	// 为了偷懒,我只看这个变量在哪些地方有使用即可.
	@Override
	protected void doStart() {
		if (!isRunning()) {
			checkTopics();
			ContainerProperties containerProperties = getContainerProperties();
			TopicPartitionInitialOffset[] topicPartitions = containerProperties.getTopicPartitions();
			if (topicPartitions != null
					&& this.concurrency > topicPartitions.length) {
				this.logger.warn("When specific partitions are provided, the concurrency must be less than or "
						+ "equal to the number of partitions; reduced from " + this.concurrency + " to "
						+ topicPartitions.length);
				this.concurrency = topicPartitions.length;
			}
			setRunning(true);

            // ******************************************************************
			// 哈哈哈,出乎我的猜测:底层没用线程池,而是直接new:KafkaMessageListenerContainer
			// 也就是说:ConcurrentMessageListenerContainer内部包裹着:KafkaMessageListenerContainer,典型的组合模式哈.
			// ******************************************************************
			for (int i = 0; i < this.concurrency; i++) {
				KafkaMessageListenerContainer<K, V> container;
				if (topicPartitions == null) {
					container = new KafkaMessageListenerContainer<>(this, this.consumerFactory,
							containerProperties);
				} else {
					container = new KafkaMessageListenerContainer<>(this, this.consumerFactory,
							containerProperties, partitionSubset(containerProperties, i));
				}
				
				String beanName = getBeanName();
				container.setBeanName((beanName != null ? beanName : "consumer") + "-" + i);
				if (getApplicationEventPublisher() != null) {
					container.setApplicationEventPublisher(getApplicationEventPublisher());
				}
				container.setClientIdSuffix("-" + i);
				container.setGenericErrorHandler(getGenericErrorHandler());
				container.setAfterRollbackProcessor(getAfterRollbackProcessor());
				// 委托给:KafkaMessageListenerContainer
				container.start();
				// 包裹着所有的:KafkaMessageListenerContainer
				this.containers.add(container);
			}
		}
	}
}	
```
### (3). 总结
Spring的设计还是挺巧妙的,通过组合模式来实现:并发数的消费,而不是通过一个线程池去配置.      