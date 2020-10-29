---
layout: post
title: 'JDK NIO Selector'
date: 2018-06-21
author: 李新
tags: NIO
---

### (1).Selector介绍
> Selector称为:选择器,当然你也可以翻译为:多路复用器.它可以用**一个线程**,处理多个客户端的连接.
> **Selector能够检测多个注册的通道(Channel)上是否有事件发生**(多个Channel以事件的方式可以注册到同一个Selector),如果有事件发生,便获取事件,然后针对每个事件进行相应的处理.
> 只有在连接真正有读写事件发生时,才会进行读写,大大地减少了系统开销,并且不必为每个连接创建一个线程,不用去维护多个线程.
> 避免了多线程之间的上下文切换导致的开销.
### (2).Selector API

```
public abstract class Selector implements Closeable {

    // 得到一个选择器对象.
    public static Selector open();

    // 监控所有注册通道,当其中有IO操作可以进行时
    // 将对应的:SelectionKey加入到内部集合中并返回
    // 其参数:timeout用来设置超时时间.
    public abstract int selectNow();
    public abstract int select(long timeout);
    public abstract int select();

    // 从选择器(Selector)的内部集合中得到所有的事件:SelectionKey
    public abstract Set<SelectionKey> selectedKeys();
}
```
