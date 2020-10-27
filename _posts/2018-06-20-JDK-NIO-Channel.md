---
layout: post
title: 'JDK NIO Channel'
date: 2018-06-20
author: 李新
tags: NIO
---

### (1). Channel介绍
NIO的通道类似于流,但有些区别如下:    
    1. 通道可以同时进行读写,**而流只能读或者只能写**.    
    2. 通道可以实现**异步读写数据**.    
    3. 通道可以从缓冲读数据,也可以写数据到缓冲.    
### (2). Channel实现类

!["Channel实现类结构图"](/assets/nio/imgs/Channel.png)

### (3). 案例
```
package com.atguigu.nio2;

import io.netty.buffer.ByteBufUtil;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.Channel;
import java.nio.channels.FileChannel;

public class NIOFileChannelTest {
    public static void main(String[] args) throws Exception {
        String outFileName = "/Users/xxx/Downloads/file01.txt";
        
        write(outFileName);
        read(outFileName);

        String targetFileName = "/Users/xxx/Downloads/file02.txt";
        copyTo(outFileName,targetFileName);
        // Channel之间直接传输文件
        transferFrom(outFileName,targetFileName);
    }

    public static void transferFrom(String source,String target) throws Exception{
        FileInputStream fis = new FileInputStream(source);
        FileChannel inChannel = fis.getChannel();

        FileOutputStream fos = new FileOutputStream(target);
        FileChannel outChannel = fos.getChannel();

        // 传输:inChannel的数据到:outChannel
        outChannel.transferFrom(inChannel,0,fis.available());
    }


    public static void copyTo(String source,String target) throws Exception{
        FileInputStream fis = new FileInputStream(source);
        FileChannel inChannel = fis.getChannel();

        FileOutputStream fos = new FileOutputStream(target);
        FileChannel outChannel = fos.getChannel();

        // 创建一个缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(10);

        while(true){
            // 先清空标志位,再进行读取
            buffer.clear();
            int read = inChannel.read(buffer);
            if(read == -1){
                break;
            }
            // 写之前先进行反转.
            buffer.flip();
            outChannel.write(buffer);
        }
        inChannel.close();
        outChannel.close();
    }

    public static void read(String path) throws Exception {
        FileInputStream fis = new FileInputStream(path);
        FileChannel channel = fis.getChannel();

        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        while (channel.read(byteBuffer) != -1) {
            // 反转
            byteBuffer.flip();

            byte[] bytes = byteBuffer.array();
            String str = new String(bytes, "UTF-8");
            System.out.println("内容为: " + str);
        }
        channel.close();
        fis.close();
    }

    public static void write(String path) throws Exception {
        String str = "hello world";
        // 创建一个输出流
        FileOutputStream fos = new FileOutputStream(path);
        // 获取对应的文件Channel
        FileChannel channel = fos.getChannel();
        // 创建一个缓存区
        ByteBuffer buffer = ByteBuffer.wrap(str.getBytes());
        // 把buffer内容写入到Channel
        channel.write(buffer);
        channel.close();
        fos.close();
    }
}

```

### (4). 