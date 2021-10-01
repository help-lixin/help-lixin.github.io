---
layout: post
title: 'Kafka生产者配置和案例(四)' 
date: 2021-10-01
author: 李新
tags:  Kafka
---

### (1). 概述
在这一小节,主要学习下Kafka生产者的使用与配置.

### (2). 生产者ACK配置

["Kafka官网配置参考"](https://kafka.apache.org/documentation/#configuration)                  

> 在同步发送的前提下,生产者在获得集群返回的ack之前会一直阻塞,集群什么时候返回ack,此时,ack有三个配置项:  

+ ack=0
  - 生产者发送消息后,不需要等待kafka多副本中任何一个broder持久化消息.性能得到了提升了,但是,会丢失消息.
+ ack=1
  - 生产者发送消息后,要等到kafka多副本之间的leader已经收到消息,并把消息持久化到磁盘上了,但是,leader挂了之后,数据也真会丢失.
+ ack=-1
  - 生产者发送消息后,要等到kafka多副本之间的follwer已经收到消息,并把消息持久化到磁盘上了,这样提供者才能继续发送下一条信息,但是,吞吐量得不到提升.  
  - min.insync.replicas=2,等待多少个follwer确认消息已经持久化,才能继续发送下一条消息,建议大于2个.     

### (3). 生产者缓冲区配置
+ buffer.memory(定义缓冲区大小)
  - Kafka默认会创建一个消息缓冲区,用来存放要发送的消息,默认是32M
+ batch.size(定义从缓冲区一定拉取数据大小)
  - Kafka生产者,会有一个线程,每次从缓冲区中获取:16KB数据,并发送给Brokder.
+ linger.ms(当数据达不到16KB时,定义隔多长时间,向Brokder发送消息)
  - 在缓冲区中的消息,就算没有达到16KB,也会在指定的毫秒内发送给Brokder.

### (4). 引入依赖
```
<!-- 注意:要与kafka安装版本保持一致 -->
<dependency>
	<groupId>org.apache.kafka</groupId>
	<artifactId>kafka-clients</artifactId>
	<version>2.8.1</version>
</dependency>

<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>fastjson</artifactId>
	<version>1.2.4</version>
</dependency>
<dependency>
	<groupId>junit</groupId>
	<artifactId>junit</artifactId>
	<version>4.12</version>
	<scope>test</scope>
</dependency>
```
### (5). 生产者发送消息案例
```
package help.lixin.kafka.example.producer;

import com.alibaba.fastjson.JSON;
import com.sun.tools.corba.se.idl.StringGen;
import help.lixin.kafka.example.producer.model.User;
import org.apache.kafka.clients.producer.*;
import org.junit.Before;
import org.junit.Test;

import java.util.Properties;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

public class TestProducer {
    private Properties kafkaProperties = null;
    private String topic = "hello";

    @Before
    public void initProperties() {
        kafkaProperties = new Properties();
        kafkaProperties.put("bootstrap.servers", "127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094");
        kafkaProperties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        kafkaProperties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    }

    /**
     * 测试同步发送消息(默认重试3次)
     * RESULT:topic:hello,partition:1,offset:8
     * RESULT:topic:hello,partition:0,offset:3
     * RESULT:topic:hello,partition:1,offset:9
     * RESULT:topic:hello,partition:1,offset:10
     */
    @Test
    public void testSync() {
        Producer<String, String> producer = new KafkaProducer(kafkaProperties);

        int condition = 5;
        for (int i = 1; i <= 1000; i++) {
            if (i == condition) {
                break;
            }

            User user = User.newBuilder()
                    .setId(i)
                    .setName("张三")
                    .setAge(25)
                    .builder();
            // hash(key) % partitionNum
            ProducerRecord<String, String> record = new ProducerRecord(
                    topic,                     // topic
                    user.getId().toString(),  // key
                    JSON.toJSONString(user)); // value
            try {
                RecordMetadata metadata = producer.send(record).get();
                String format = String.format("topic:%s,partition:%d,offset:%d", metadata.topic(), metadata.partition(), metadata.offset());
                System.out.println("RESULT:" + format);
            } catch (InterruptedException e) {
                e.printStackTrace();
                // 同步发送,默认会重试3次,当3次失败后,会抛出异常.
                // 可以记录到日志里,人工干预处理.
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 测试异步发送消息()
     * RESULT:topic:hello,partition:0,offset:2
     * RESULT:topic:hello,partition:1,offset:5
     * RESULT:topic:hello,partition:1,offset:6
     * RESULT:topic:hello,partition:1,offset:7
     *
     * @throws Exception
     */
    @Test
    public void testAsync() {
        Producer<String, String> producer = new KafkaProducer(kafkaProperties);

        int condition = 10;
        for (int i = 6; i <= 1000; i++) {
            if (i == condition) {
                break;
            }
            User user = User.newBuilder()
                    .setId(i)
                    .setName("张三")
                    .setAge(25)
                    .builder();

            // 异步发送
            // hash(key) % partitionNum
            ProducerRecord<String, String> record = new ProducerRecord(
                    topic,                     // topic
                    user.getId().toString(),  // key
                    JSON.toJSONString(user)); // value

            // Callback
            producer.send(record, (metadata, ex) -> {

                if (null != metadata) {
                    String format = String.format("topic:%s,partition:%d,offset:%d", metadata.topic(), metadata.partition(), metadata.offset());
                    System.out.println("RESULT:" + format);
                }

                if (null != ex) {
                    System.out.println(ex.getMessage());
                }
            });
        }

        try {
            // sleep , wait callback end
            TimeUnit.SECONDS.sleep(30);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }


    /**
     * 测试消息保存在指定的分区下
     * RESULT:topic:hello,partition:1,offset:11
     * RESULT:topic:hello,partition:1,offset:12
     * RESULT:topic:hello,partition:1,offset:13
     * RESULT:topic:hello,partition:1,offset:14
     *
     * @throws Exception
     */
    @Test
    public void testPartition() throws Exception {
        Producer<String, String> producer = new KafkaProducer(kafkaProperties);

        int condition = 5;
        for (int i = 1; i <= 1000; i++) {
            if (i == condition) {
                break;
            }

            User user = User.newBuilder()
                    .setId(i)
                    .setName("张三")
                    .setAge(25)
                    .builder();
            // hash(key) % partitionNum
            ProducerRecord<String, String> record = new ProducerRecord(
                    topic,                     // topic
                    1,                // partition
                    user.getId().toString(),  // key
                    JSON.toJSONString(user)); // value
            RecordMetadata metadata = producer.send(record).get();
            String format = String.format("topic:%s,partition:%d,offset:%d", metadata.topic(), metadata.partition(), metadata.offset());
            System.out.println("RESULT:" + format);
        }
    }


    /**
     * 测试同步发送消息,并且配置ack
     */
    @Test
    public void testAckConfig() {
        // *****************************ACK配置***************************************
        /**
         * ack = 0  , 生产者发送消息后,不需要等待kafka多副本中任何一个broder持久化消息.性能得到了提升了,但是,会丢失消息.
         * ack = 1  , 生产者发送消息后,要等到kafka多副本之间的leader已经收到消息,并把消息持久化到磁盘上了,但是,leader挂了之后,数据也真会丢失.
         * ack = -1 , 生产者发送消息后,要等到kafka多副本之间的follwer已经收到消息,并把消息持久化到磁盘上了,这样提供者才能继续发送下一条信息,但是,吞吐量得不到提升.
         */
        kafkaProperties.put(ProducerConfig.ACKS_CONFIG, -1);

        // ******************************重试配置**************************************
        // 同步重试次数为:3
        kafkaProperties.put(ProducerConfig.RETRIES_CONFIG, 3);
        // 重试间隔(3秒后重试)
        kafkaProperties.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 3000);

        // *****************************缓冲区配置***************************************
        // 1. Message --> Buffer(32M)
        // 2. Kafka Producer Thread(Poll 16KB) Message --> Buffer(32M)
        // 3. Kafka Producer Thread --> Send Broder
        // Kafka的客户端发送数据到服务器,是经过缓冲(不是来一条就发一条),也就是说:通过KafkaProducer发送出去的消息都是先进入到客户端本地的内存缓冲里,然后把很多消息收集成一个一个的Batch,再发送到Broker上去的,这样性能才可能高.
        // buffer.memory的本质就是用来约束Kafka Producer能够使用的内存缓冲的大小的,默认值32MB.
        kafkaProperties.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        // 当一批消息体大小达到这个batch.size的时候会发送,默认是:16KB
        kafkaProperties.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);

        // 不论消息是否在缓存冲达到16K,都会在规定的时间内(linger.ms),把消息发送出去
        // 10毫秒内,batch没有满的情况下,也必须把消息发送出去,不能让消息发送延迟时间太长.
        kafkaProperties.put(ProducerConfig.LINGER_MS_CONFIG, 10);

        Producer<String, String> producer = new KafkaProducer(kafkaProperties);

        int condition = 10;
        for (int i = 6; i <= 1000; i++) {
            if (i == condition) {
                break;
            }
            User user = User.newBuilder()
                    .setId(i)
                    .setName("张三")
                    .setAge(25)
                    .builder();

            // 异步发送
            // hash(key) % partitionNum
            ProducerRecord<String, String> record = new ProducerRecord(
                    topic,                     // topic
                    user.getId().toString(),  // key
                    JSON.toJSONString(user)); // value

            // Callback
            producer.send(record, (metadata, ex) -> {

                if (null != metadata) {
                    String format = String.format("topic:%s,partition:%d,offset:%d", metadata.topic(), metadata.partition(), metadata.offset());
                    System.out.println("RESULT:" + format);
                }

                if (null != ex) {
                    System.out.println(ex.getMessage());
                }
            });
        }

        try {
            // sleep , wait callback end
            TimeUnit.SECONDS.sleep(30);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
