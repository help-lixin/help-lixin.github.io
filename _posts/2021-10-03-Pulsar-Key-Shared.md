---
layout: post
title: 'Pulsar Key Shared案例(四)' 
date: 2021-10-03
author: 李新
tags:  Pulsar
---

### (1). 概述
在这里对Key Sahred模式进行测试,以对Key Shared有更进一步的了解.

### (2). 案例代码
```
package help.lixin.pulsar.example;

import org.apache.pulsar.client.api.*;

import java.util.concurrent.TimeUnit;

public class KeySharedTest {
    private static final String SERVER_URL = "pulsar://localhost:6650";
    private static final String TOPIC = "persistent://public/default/test_2";

    public static void main(String[] args) throws Exception {
        Thread product = new ProductThread();
        product.setName("product");
        product.start();

        Thread consumer1 = new ConsumerThread();
        consumer1.setName("consumer-1");
        consumer1.start();

        Thread consumer2 = new ConsumerThread();
        consumer2.setName("consumer-2");
        consumer2.start();

        Thread consumer3 = new ConsumerThread();
        consumer3.setName("consumer-3");
        consumer3.start();

        TimeUnit.HOURS.sleep(1);
    }

    static class ProductThread extends Thread {
        @Override
        public void run() {
            try {
                PulsarClient client = PulsarClient.builder()
                        //
                        .serviceUrl(SERVER_URL)
                        //
                        .enableTcpNoDelay(true)
                        //
                        .build();
                Producer<String> producer = client.newProducer(Schema.STRING)
                        //
                        .producerName("my-producer")
                        // 指定生产者模式做测试
                        // .accessMode(ProducerAccessMode.Shared)
                        .topic(TOPIC)
                        //
                        .sendTimeout(10, TimeUnit.SECONDS)
                        //
                        .create();
                for (int i = 0; i < 500; i++) {
                    try {
                        String key = "00001";
                        if (i % 2 == 0) {
                            key = "00007";
                        }
                        // 同步发送消息,并指定:key
                        MessageId messageId = producer.newMessage().key(key).value("Hello World! " + key + " -- " + i).send();
                        // System.out.println(Thread.currentThread().getName() + " Send Message: " + messageId);
                        TimeUnit.SECONDS.sleep(3);
                    } catch (Exception ignore) {
                        System.out.println(Thread.currentThread().getName() + " Send Fail " + ignore.getMessage());
                    }
                }// end for
            } catch (Exception ignore) {
                System.out.println(Thread.currentThread().getName() + " Error " + ignore.getMessage());
            }// end catch
        }
    } // end


    static class ConsumerThread extends Thread {
        @Override
        public void run() {
            try {

                // 构造Pulsar Client
                PulsarClient client = PulsarClient.builder()
                        //
                        .serviceUrl(SERVER_URL)
                        //
                        .enableTcpNoDelay(true).build();

                Consumer consumer = client.newConsumer()
                        // 消息费者名称
                        .consumerName("my-consumer")
                        // 主题
                        .topic(TOPIC)
                        // 订阅名称
                        .subscriptionName("my-subscription")
                        //
                        .ackTimeout(10, TimeUnit.SECONDS)
                        //
                        .maxTotalReceiverQueueSizeAcrossPartitions(10)
                        //
                        .subscriptionType(SubscriptionType.Key_Shared)
                        //
                        .subscribe();
                do {
                    // 接收消息有两种方式：异步和同步
                    Message message = consumer.receive();
                    String key = message.getKey();
                    String format = String.format("%s", new String(message.getData()));
                    System.err.println(Thread.currentThread().getName() + " Revice Msg: " + format + " key: " + key);
                    // 消息确认机制
                    consumer.acknowledge(message);
                } while (true);
            } catch (Exception ignore) {
                System.out.println(Thread.currentThread().getName() + " Revice Msg ERROR: " + ignore.getMessage());
            }
        }
    } // end
}
```
### (3). 测试结果
```
consumer-2 Revice Msg: Hello World! 00007 -- 0 key: 00007
consumer-3 Revice Msg: Hello World! 00001 -- 1 key: 00001
consumer-2 Revice Msg: Hello World! 00007 -- 2 key: 00007
consumer-3 Revice Msg: Hello World! 00001 -- 3 key: 00001
consumer-2 Revice Msg: Hello World! 00007 -- 4 key: 00007
consumer-3 Revice Msg: Hello World! 00001 -- 5 key: 00001
consumer-2 Revice Msg: Hello World! 00007 -- 6 key: 00007
consumer-3 Revice Msg: Hello World! 00001 -- 7 key: 00001
consumer-2 Revice Msg: Hello World! 00007 -- 8 key: 00007
consumer-3 Revice Msg: Hello World! 00001 -- 9 key: 00001
consumer-2 Revice Msg: Hello World! 00007 -- 10 key: 00007
```
### (4). 总结
我创建三个线程订阅消息,从结果能看出来:仅有两个线程在消费,得到的结论就是:当线程数大于key的数量时,线程数是闲出来的. 

### (5). Pulsar与RocketMQ比较后的缺点
> 在RocketMQ里,不仅可以指定Topic,还可以指定Tag(Tag相当于Pulsar里的Key),而,Pulsar在Key Shared模式下,维度只能到Topic级别.   
> 假如,有这样一个场景:应用部署是区分物理分区的:华东区和华南区,华东区和华南区的数据源是隔离的,那么问题来了,这种情况下,如果华南区消费了华东区的MQ数据,在进行数据源操作时,是会连接不到华东区的数据源的(比较MQ的消费不是那么纯粹,会伴随业务操作的数据落地),解决方案还是有的:    
> 1. 不同的物理分区(华南和华东)用不同的namespace.    
> 2. 在不区分namespace的情况下,大家在一起消费消息,把不属于自己分区的消息给拒绝掉,只是这样做的情况下,这条消息会被拒N-1次.      