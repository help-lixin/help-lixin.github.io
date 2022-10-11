---
layout: post
title: 'Playwright源码之PipeTransport(四)'
date: 2022-10-09
author: 李新
tags:  Playwright
---

### (1). 概述
在前一小篇,剖析到,Playwright会通过Process Hold住进程,然后,通过:PipeTransport包裹着Process,代表:PipeTransport是一个与进程通信的中间层. 

### (2). Transport
> 最喜欢的一件事情就是看接口了,因为:接口代表了这个实现类的大概功能. 

```
public interface Transport {
  //  发送消息
  void send(JsonObject message);
  // 拉取消息
  JsonObject poll(Duration timeout);
  // 关闭
  void close() throws IOException;
}
```
### (3). PipeTransport构造器
```
public class PipeTransport implements Transport {
	// **********************************************************
	// 1. 创建阻塞队列
	// 主要用于,当向浏览器发送消息以及浏览器回复消息时的载体.
	// **********************************************************
	private final BlockingQueue<JsonObject> incoming = new ArrayBlockingQueue<>(1000);
	private final BlockingQueue<String> outgoing= new ArrayBlockingQueue<>(1000);

	private final ReaderThread readerThread;
	private final WriterThread writerThread;

	private boolean isClosed;

	PipeTransport(InputStream input, OutputStream output) {
		
		DataInputStream in = new DataInputStream(new BufferedInputStream(input));
		
		// **********************************************************
		// 2. 创建读写线程,并启动.
		// **********************************************************
		readerThread = new ReaderThread(in, incoming);
		readerThread.start();
		writerThread = new WriterThread(output, outgoing);
		writerThread.start();
	} // end 
	
	public void send(JsonObject message) {
		if (isClosed) {
		  throw new PlaywrightException("Playwright connection closed");
		}
		
		try {
          // **************************************************************
		  // 3. 把对象消息转换成JSON字符串,并,插入到队列(BlockingQueue<String>)里.
		  // **************************************************************
		  outgoing.put(gson().toJson(message));
		} catch (InterruptedException e) {
		  throw new PlaywrightException("Failed to send message", e);
		}
	} // end send

  @Override
  public JsonObject poll(Duration timeout) {
    if (isClosed) {
      throw new PlaywrightException("Playwright connection closed");
    }
	
    try {
	   // **************************************************************
	   // 4. 读取队列(BlockingQueue<JsonObject>)里的数据
	   // **************************************************************
      JsonObject message = incoming.poll(timeout.toMillis(), TimeUnit.MILLISECONDS);
      if (message == null && readerThread.exception != null) {
        try {
          close();
        } catch (IOException e) {
          e.printStackTrace(System.err);
        }
        throw new PlaywrightException("Failed to read message from driver, pipe closed.", readerThread.exception);
      }
      return message;
    } catch (InterruptedException e) {
      throw new PlaywrightException("Failed to read message", e);
    }
  }// end 	poll
}	
```
### (4). 总结
PipeTransport的主要职责是:发送消息,拉取消息,但是,这两者都只是把数据塞到队列里,并没有看到与进程通信,原因在于:Playwright通过:ReaderThread和WriterThread进行了解藕. 