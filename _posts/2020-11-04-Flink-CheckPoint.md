---
layout: post
title: 'Flink CheckPoint'
date: 2020-11-04
author: 李新
tags: Flink
---

### (1). Checkpoint开启和时间间隔指定
> 开启检查点,并且指定检查点时间间隔为1000ms.
```
env.enableCheckpointing(1000);
```
### (2). exactly-ance和at-least-once语义选择
> 

```
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.AT_LEAST_ONCE);

```
### (3). Checkpoint超时时间
> 指定Checkpoint执行过程的上限时间范围,一旦Checkpoint执行时间超过该阀值,Flink将会中断Checkpoint过程,并按照超时时间处理(**默认是10分钟**)

```
env.getCheckpointConfig().setCheckpointTimeout(50000);
```
### (4). 检查点之间最小时间间隔
> 设定两个Checkpoint之间的最小间隔,防止出现Checkpoint积压过多.
```
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);
```
### (5). 最大并行执行的检查点数量
> 能够最大同时执行的Checkpoint数量,在默认情况下只有一个检查点可以运行.

```
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
```
### (6). 是否删除Checkpoint中保存数据
> RETAIN_ON_CANCELLATION : Flink被Cancel后,会**保留**Checkpoint数据
> DELETE_ON_CANCELLATION : Flink被Cancel后,会**删除**Checkpoint数据,

```
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);

env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION);
```
### (7). TolerableCheckpointFailureNumber
> 设置可以容忍的检查次数,超过这个数量Flink将会自动关闭和停止任务.

```
env.getCheckpointConfig.setTolerableCheckpointFailureNumber(1)
```
