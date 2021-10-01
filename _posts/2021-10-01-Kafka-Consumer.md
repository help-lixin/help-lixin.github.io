---
layout: post
title: 'Kafka消费者案例(五)' 
date: 2021-10-01
author: 李新
tags:  Kafka
---

### (1). 概述
在这一小节,主要学习Kakfa对消息的消费,并注意几个配置参数(offset提交/拉取的最大批次).

### (2). 消费者案例
```
package help.lixin.kafka.example.producer;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.junit.Before;
import org.junit.Test;

import java.time.Duration;
import java.util.Arrays;
import java.util.Properties;

public class TestConsumer {
    private Properties kafkaProperties = null;
    private String topic = "hello";
    private String groupName = "test-service";

    @Before
    public void initProperties() {
        kafkaProperties = new Properties();
        kafkaProperties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9092,127.0.0.1:9093,127.0.0.1:9094");
        kafkaProperties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        kafkaProperties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // 定义消费者组
        kafkaProperties.put(ConsumerConfig.GROUP_ID_CONFIG, groupName);
    }


    /**
     * 测试每次拉取消息的批次为2条记录
     */
    @Test
    public void testConsumerDefinMaxRecord() {
        // 定义每次拉取消息的最大数量.
        kafkaProperties.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 2);

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(kafkaProperties);
        // 消息订阅
        consumer.subscribe(Arrays.asList(topic));

        while (true) {
            // 每隔1秒,拉取一批消息(ConsumerRecords).
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            System.out.println("recods:" + records.count());
            // ConsumerRecord 为具体的消息
            for (ConsumerRecord<String, String> record : records) {
                String format = String.format("topic:[%s],partition:[%d],offset:[%d],value:[%s]", record.topic(), record.partition(), record.offset(), record.value());
                System.out.println("RESULT:" + format);
            }
        }
    }

    /**
     * 测试简单消费,默认订阅的是topic的最后的offset,消费消息后自动提交offset
     */
    @Test
    public void testConsumerAutoCommitOffset() {
        // *****************************消费者自动提交**************************************
        // 是否自动提交offset,默认是:true
        // kafkaProperties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,"true");
        // 自动提交offset的间隔时间
        // kafkaProperties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG,"1000");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(kafkaProperties);
        // 消息订阅
        consumer.subscribe(Arrays.asList(topic));


        while (true) {
            // 每隔1秒,拉取一批消息(ConsumerRecords).
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            // ConsumerRecord 为具体的消息
            for (ConsumerRecord<String, String> record : records) {
                String format = String.format("topic:[%s],partition:[%d],offset:[%d],value:[%s]", record.topic(), record.partition(), record.offset(), record.value());
                System.out.println("RESULT:" + format);
            }
        }
    }

    /**
     * 测试消费消息后,手动提交offset.
     */
    @Test
    public void testConsumerManualCommitOffset() {
        // *****************************消费者手动提交**************************************
        kafkaProperties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(kafkaProperties);
        // 消息订阅
        consumer.subscribe(Arrays.asList(topic));
        while (true) {
            // 每隔1秒,拉取一批消息(ConsumerRecords).
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            // ConsumerRecord 为具体的消息
            for (ConsumerRecord<String, String> record : records) {
                String format = String.format("topic:[%s],partition:[%d],offset:[%d],value:[%s]", record.topic(), record.partition(), record.offset(), record.value());
                System.out.println("RESULT:" + format);
            }

            if (records.count() > 0) {
                // 手动同步提交offset,会一直阻塞代码往下执行.
                consumer.commitSync();

                // 手动异步提交offset
                // consumer.commitAsync((offsets, ex) -> {
                //
                // });
            }
        }
    }

}
```