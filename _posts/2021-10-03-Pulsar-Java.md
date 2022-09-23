---
layout: post
title: 'Pulsar Java代码调用(三)' 
date: 2021-10-03
author: 李新
tags:  Pulsar
---

### (1). 引入依赖(pom.xml)
```
<properties>
	<maven.compiler.source>8</maven.compiler.source>
	<maven.compiler.target>8</maven.compiler.target>
	<pulsar.version>2.8.1</pulsar.version>
</properties>

<dependencies>
	<dependency>
		<groupId>org.apache.pulsar</groupId>
		<artifactId>pulsar-client</artifactId>
		<version>${pulsar.version}</version>
	</dependency>
	<dependency>
		<groupId>junit</groupId>
		<artifactId>junit</artifactId>
		<version>4.12</version>
		<scope>test</scope>
	</dependency>
</dependencies>
```
### (2). 生产者
```
package help.lixin.pulsar.example;

import org.apache.pulsar.client.api.*;
import org.junit.Test;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class ProductTest {
    // 自己本地配置host(启动时,会去远程服务器,拉取集群列表出来,好像我的集群列表是机器名称,不是ip列表)
    // ./bin/pulsar-admin brokers list pulsar-cluster
    //
    // 10.211.55.100 pulsar.lixin.help
    // 10.211.55.101 pulsar.lixin.help
    // 10.211.55.102 pulsar.lixin.help
    // 10.211.55.100 erp-100
    // 10.211.55.101 erp-101
    // 10.211.55.102 erp-102
    private static final String SERVER_URL = "pulsar://pulsar.lixin.help:6650";

    @Test
    public void testProduct() throws Exception {
        // 构造Pulsar Client
        PulsarClient client = PulsarClient.builder()
                .serviceUrl(SERVER_URL)
                .enableTcpNoDelay(true)
                .build();
        // 构造生产者
        Producer<String> producer = client.newProducer(Schema.STRING)
                .producerName("my-producer")
                .topic("persistent://public/default/test")
                .batchingMaxMessages(1024)
                .batchingMaxPublishDelay(10, TimeUnit.MILLISECONDS)
                .enableBatching(true)
                .blockIfQueueFull(true)
                .maxPendingMessages(512)
                .sendTimeout(10, TimeUnit.SECONDS)
                .blockIfQueueFull(true)
                .create();
        // 同步发送消息
        MessageId messageId = producer.send("Hello World");
        String format = String.format("message id is %s", messageId);
        System.out.println(format);
        CompletableFuture<MessageId> asyncMessageId = producer.sendAsync("This is a async message");
        // 阻塞线程，直到返回结果
        String format2 = String.format("async message id is %s" , asyncMessageId.get());
        System.out.println(format2);
        // 配置发送的消息元信息，同步发送
        producer.newMessage()
                .key("my-message-key")
                .value("my-message")
                .property("my-key", "my-value")
                .property("my-other-key", "my-other-value")
                .send();
        producer.newMessage()
                .key("my-async-message-key")
                .value("my-async-message")
                .property("my-async-key", "my-async-value")
                .property("my-async-other-key", "my-async-other-value")
                .sendAsync();

        // 关闭producer的方式有两种：同步和异步
        // producer.closeAsync();
        producer.close();

        // 关闭licent的方式有两种，同步和异步
        // client.close();
        client.closeAsync();
    }
}

```
### (3). 消费者
```
package help.lixin.pulsar.example;

import org.apache.pulsar.client.api.Consumer;
import org.apache.pulsar.client.api.Message;
import org.apache.pulsar.client.api.PulsarClient;
import org.apache.pulsar.client.api.SubscriptionType;
import org.junit.Test;

import java.util.concurrent.TimeUnit;

public class ConsumerTest {

    // 自己本地配置host(启动时,会去远程服务器,拉取集群列表出来,好像我的集群列表是机器名称,不是ip列表)
    // ./bin/pulsar-admin brokers list pulsar-cluster
    //
    // 10.211.55.100 pulsar.lixin.help
    // 10.211.55.101 pulsar.lixin.help
    // 10.211.55.102 pulsar.lixin.help
    // 10.211.55.100 erp-100
    // 10.211.55.101 erp-101
    // 10.211.55.102 erp-102
    private static final String SERVER_URL = "pulsar://pulsar.lixin.help:6650";

    @Test
    public void testConsumer1() throws Exception {
        // 构造Pulsar Client
        PulsarClient client = PulsarClient.builder()
                .serviceUrl(SERVER_URL)
                .enableTcpNoDelay(true)
                .build();

        Consumer consumer = client.newConsumer()
                .consumerName("my-consumer")
                .topic("persistent://public/default/test")
                .subscriptionName("my-subscription")
                .ackTimeout(10, TimeUnit.SECONDS)
                .maxTotalReceiverQueueSizeAcrossPartitions(10)
                .subscriptionType(SubscriptionType.Exclusive)
                .subscribe();
        do {
            // 接收消息有两种方式：异步和同步
            // CompletableFuture<Message<String>> message = consumer.receiveAsync();
            Message message = consumer.receive();
            String format = String.format("%s", new String(message.getData()));
            System.err.println(format);

            // 消息确认机制
            consumer.acknowledge(message);
        } while (true);
    }
}
```
### (4). 总结
后面,会对Pulsar的基本命令进行学习,以及相关源码深入剖析.  