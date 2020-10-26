---
layout: post
title: 'Netty源码(NioServerSocketChannel)'
date: 2020-10-26
author: 李新
tags: Netty
---

### (1).NioServerSocketChannel 类结构图
!["NioServerSocketChannel类结构图"](/assets/netty/imgs/NioServerSocketChannel.png)


### (2).NioServerSocketChannel初始化过程
```
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();


// 1.通过反射工厂创建该对象
public NioServerSocketChannel() {
    // 1.1 newSocket
    // 1.2 this
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}


// 1.2 构造器
public NioServerSocketChannel(ServerSocketChannel channel) {
    // channel = sun.nio.ch.ServerSocketChannelImpl
    // 3.调用父类(AbstractNioMessageChannel/AbstractNioChannel/AbstractChannel)的构造器
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}

// 1.1 创建:ServerSocketChannel
private static ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
         return provider.openServerSocketChannel();
    } catch (IOException e) {
        throw new ChannelException("Failed to open a server socket.", e);
    }
} //end newSocket
```

### (3).AbstractNioMessageChannel
```
protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    // parent = null
    // ch = sun.nio.ch.ServerSocketChannelImpl
    // readInterestOp = SelectionKey.OP_ACCEPT(16)
    
    // 4. 调用父类(AbstractNioChannel)的构造器
    super(parent, ch, readInterestOp);
}
```

### (4).AbstractNioChannel
```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    // parent = null
    // ch = sun.nio.ch.ServerSocketChannelImpl
    // readInterestOp = SelectionKey.OP_ACCEPT(16)
    
    // 5.调用父类(AbstractChannel)构造器
    super(parent);
    
    this.ch = ch;
   this.readInterestOp = readInterestOp;
    
    try {
        // 设置非阻塞
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            // 有异常的情况下关闭
            ch.close();
        } catch (IOException e2) {
            logger.warn("Failed to close a partially initialized socket.", e2);
        }
        throw new ChannelException("Failed to enter non-blocking mode.", e);
    } // end catch
}
```

### (5).AbstractChannel
```
protected AbstractChannel(Channel parent) {
    // parent = null
    this.parent = parent;
    // 创建ChannelId
    id = newId();
    
    // 6.调用AbstractNioMessageChannel.newUnsafe() 创建Channel$Unsafe
    unsafe = newUnsafe();
    
    // 创建DefaultChannelPipeline
    pipeline = newChannelPipeline();
}

// 创建默认的ChannelPipeline
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}

```

### (6).AbstractNioMessageChannel.newUnsafe
```
// AbstractNioMessageChannel
protected AbstractNioUnsafe newUnsafe() {
    return new NioMessageUnsafe();
}
```

### (7).NioServerSocketChannel 初始化过程图解
!["NioServerSocketChannel调用过程"](/assets/netty/imgs/NioServerSocketChannel-Invoker.png)

### (8).NioServerSocketChannel 初始化总结
    1. 创建:ServerSocketChannel(sun.nio.ch.ServerSocketChannelImpl)
    2. 创建:Unsafe对象(NioMessageUnsafe)
    3. 创建:DefaultChannelPipeline对象
