---
layout: post
title: 'JDK NIO MappedByteBuffer'
date: 2018-06-20
author: 李新
tags: NIO
---

### (1).MappedByteBuffer介绍
> NIO提供了MappedByteBuffer,可以让文件直接在内存(堆外内存)中进行修改,而如何同步到文件则是由NIO来完成

### (2).MappedByteBuffer类结构图
!["MappedByteBuffer类结构图"](/assets/nio/imgs/MappedByteBuffer.jpg)

### (3). MappedByteBuffer案例
```
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;
import java.util.concurrent.CountDownLatch;

/**
 * 1. MappedByteBuffer 可以文件直接在内存修改(堆外内存),操作系统不需要拷贝一次.
 */
public class MappedByteBufferTest {
    public static void main(String[] args) throws Exception {
        RandomAccessFile file =
                new RandomAccessFile("/Users/lixin/Downloads/尚硅谷Netty学习资料/代码/NettyPro/target/file01.txt","rw");
        FileChannel channel = file.getChannel();
        // MapMode     : 使用的模式,可选值(PRIVATE/READ_ONLY/READ_WRITE)
        // position    : 可以直接修改的起始位置
        // size        : 最多可映射到内存的大小.
        // 即可以修改的范围是position~size之间
        MappedByteBuffer mappedByteBuffer = channel.map(FileChannel.MapMode.READ_WRITE,0,5);
        mappedByteBuffer.putChar(0,'H');
        file.close();
    }
}

```
