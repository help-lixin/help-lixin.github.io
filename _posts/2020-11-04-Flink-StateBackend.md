---
layout: post
title: 'Flink StateBackend'
date: 2020-12-04
author: 李新
tags: Flink
---

### (1). StateBackend
> 在默认情况下,State会保存在TaskManager的内存中,Checkpoint会存储在JobManager的内存中,State和Checkpoint的存储位置取决于StateBackend的配置,Flink提供了3种StateBackend,以及RockDB作为存储介质的RocksDBState-Bacnend.

### (2). MemoryStateBackend
> State数据保存在java堆内存中,jobManager触发checkpoint的时候,会把TaskManager 中的state的快照数据保存到jobmanager的内存中,基于内存的state backend在生产环境下不建议使用.
> 启用代码如下:

```
// JobManager的内存大小(10M)
env.setStateBackend(new MemoryStateBackend(10 * 1024 * 1024))
```
### (3). FsStateBackend
> State数据保存在Taskmanager的内存中,执行checkpoint的时候,会把state的快照数据保存到配置的文件系统中,可以使用hdfs等分布式文件系统.
> 启用代码如下:

```
env.setStateBackend(new FsStateBackend("hdfs://hadoop:9000/checkpoint/cp1"))
```
### (4). RocksDBStateBackend
> RocksDB跟上面的都略有不同,它会在本地文件系统中维护状态,state会直接写入本地rocksdb中.同时它需要配置一个远端的filesystem uri（一般是HDFS),在做checkpoint的时候,会把本地的数据直接复制到filesystem中.fail over的时候从filesystem中恢复到本地.RocksDB克服了state受内存限制的缺点,同时又能够持久化到远端文件系统中,比较适合在生产中使用.
> 启用代码如下:

```
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-statebackend-rocksdb_2.12</artifactId>
    <version>1.11.2</version>
</dependency>
```

```
env.setStateBackend(new RocksDBStateBackend("hdfs://hadoop:9000/checkpoint/cp2"))
```
### (5). 全局配置(flink-conf.yaml)
```
# 支持: 'jobmanager', 'filesystem', 'rocksdb'
state.backend: filesystem

#保存chekcpoint的目录
state.checkpoints.dir: hdfs://namenode-host:port/flink-checkpoints
```

### (6). 案例
```
import org.apache.flink.contrib.streaming.state.RocksDBStateBackend
import org.apache.flink.runtime.state.filesystem.FsStateBackend
import org.apache.flink.streaming.api.CheckpointingMode
import org.apache.flink.streaming.api.environment.CheckpointConfig
import org.apache.flink.streaming.api.scala._

object StateFileStoreTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    // 指定chekcpoint间隔
    env.enableCheckpointing(6000)
    // 指定存储策略
    env.setStateBackend(new FsStateBackend("file:///Users/lixin/IDEAWorkspace/flink-example/flink-wordcount/target/checkpoint"))
    env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)
    env.getCheckpointConfig.setCheckpointTimeout(5000)
    // 中止job,会保留检查点数据
    env.getCheckpointConfig.enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION)

    val inpuDataStream = env.socketTextStream("localhost", 7000)

    val resultDataStream = inpuDataStream.flatMap(_.toUpperCase.split(" "))
      .filter(_.nonEmpty)
      .map((_, 1))
      .keyBy(0)
      .sum(1)

    resultDataStream.print
    env.execute()
  } // end main
}
```

### (7). 查看保存内容

!["Flink CheckPoint保存点"](/assets/flink/imgs/flink-savepoint.jpg)

### (8). 测试流程
> 启动Flink 
> 启动nc(nc -lk 7000)
> 向Flink提交任务  
> nc添加数据 
> 停止任务
> 重新提交任务并指定savepoint(/Users/lixin/IDEAWorkspace/flink-example/flink-wordcount/target/checkpoint/checkpoint/95ddab2699f4573944174dade0f10acc/chk-6)