---
layout: post
title: 'Playwright源码之WriterThread和ReaderThread(五)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
在前面剖析了:PipeTransport它仅仅是所数据写到BlockingQueue里又或者从BlockingQueue里读取数据,在BlockingQueue里的数据是如何与进程通信?进程数据又是如何写入到BlockingQueue里的呢?答案就在:WriterThread和ReaderThread.  

### (2). WriterThread
```
class WriterThread extends Thread {
  final OutputStream out;
  private final BlockingQueue<String> queue;

  WriterThread(OutputStream out, BlockingQueue<String> queue) {
    this.out = out;
    this.queue = queue;
  }

  @Override
  public void run() {
    while (!isInterrupted()) {
      try {
        if (queue.isEmpty())
          out.flush();
		// ******************************************************************  
		// 1. take从队尾弹出一条信息并发送给进程(Process)
		// ******************************************************************  
        sendMessage(queue.take());
      } catch (IOException e) {
        if (!isInterrupted())
          e.printStackTrace();
        break;
      } catch (InterruptedException e) {
        break;
      }
    }
  }

  private void sendMessage(String message) throws IOException {
    byte[] bytes = message.getBytes(StandardCharsets.UTF_8);
	// 2. 发送消息的最长度和内容
    writeIntLE(out, bytes.length);
	// 3. 写出所有字节
    out.write(bytes);
  }
  
  private static void writeIntLE(OutputStream out, int v) throws IOException {
    out.write(v >>> 0 & 255);
    out.write(v >>> 8 & 255);
    out.write(v >>> 16 & 255);
    out.write(v >>> 24 & 255);
  }
}
```
### (3). ReaderThread
```
class ReaderThread extends Thread {
  private final DataInputStream in;
  private final BlockingQueue<JsonObject> queue;
  volatile boolean isClosing;
  volatile Exception exception;

  ReaderThread(DataInputStream in, BlockingQueue<JsonObject> queue) {
    this.in = in;
    this.queue = queue;
  }

  @Override
  public void run() {
    while (!isInterrupted()) {
      try {
		 // ******************************************************* 
		 // 2. 读取消息,通过GSON转换成:JsonObject,并添加到BlockingQueue里.
		 // ******************************************************* 
        JsonObject message = gson().fromJson(readMessage(), JsonObject.class);
        queue.put(message);
      } catch (IOException e) {
        if (!isInterrupted() && !isClosing) {
          exception = e;
        }
        break;
      } catch (InterruptedException e) {
        break;
      }
    }
  }

  // ******************************************************* 
  // 1. 从输入流中读取数据,并转换成:String
  // ******************************************************* 
  private String readMessage() throws IOException {
    int len = readIntLE(in);
    byte[] raw = new byte[len];
    in.readFully(raw, 0, len);
    return new String(raw, StandardCharsets.UTF_8);
  }
  
  private static int readIntLE(DataInputStream in) throws IOException {
    int ch1 = in.read();
    int ch2 = in.read();
    int ch3 = in.read();
    int ch4 = in.read();
    if ((ch1 | ch2 | ch3 | ch4) < 0) {
      throw new EOFException();
    } else {
      return (ch4 << 24) + (ch3 << 16) + (ch2 << 8) + (ch1 << 0);
    }
  }
  
}
```
### (4). 总结
+ ReaderThread的职责就是读取流(DataInputStream)里的数据,转换成:JsonObject并添加到BlockingQueue里. 
+ WriterThread的职责是读取BlockingQueue里的数据,往流(OutputStream)里写. 