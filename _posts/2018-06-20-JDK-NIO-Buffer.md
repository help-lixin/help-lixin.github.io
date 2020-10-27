---
layout: post
title: 'JDK NIO Buffer'
date: 2018-06-20
author: 李新
tags: NIO
---

### (1). Buffer介绍
缓冲区本质上是一个可以读写数据的内存块,可以理解成是一个容器对象(含数组),该对象提供了一组方法,可以更轻松地使用内存块,缓冲区对象内置了一些机制,能够跟踪和记录缓冲区的状态变化情况.

### (2). Buffer类关系图
> java.nio.Buffer的相关子类
!["Buffer的子类"](/assets/nio/imgs/Buffer.jpg)

> java.nio.NioBuffer的相关子类
!["IntBuffer子类"](/assets/nio/imgs/IntBuffer-SubClass.jpg)

### (3).Buffer内部属性讲解
> mark:
    标记位置,用于记录某次读写的位置,可以通过reset()方法回到这里    
> position:
	下一个被读或写的位置   
> limit:
    可供读写的最大位置,**用于限制position**,position < limit    
> capacity:
	Buffer所能够存放的最大容量    


!["Buffer内部属性"](/assets/nio/imgs/Buffer-Inner-Properties.jpg)

### (4). 案例
```
public class BufferTest {
    public static void main(String[] args) {
        // 创建了一个Buffer,大小为:5,可以存放5个int
        // 刚创建Buffer时: position = 0 / limit = 5 / capacity = 5
        IntBuffer buf = IntBuffer.allocate(5);

        // position = 1 / limit = 5 / capacity = 5
        buf.put(1);
        // position = 2 / limit = 5 / capacity = 5
        buf.put(2);
        // position = 3 / limit = 5 / capacity = 5
        buf.put(3);
        // position = 4 / limit = 5 / capacity = 5
        buf.put(4);
        // position = 5 / limit = 5 / capacity = 5
        buf.put(5);

        // 将buffer转换,读写切换(把position设置为:0,limit为上次写入的最大position)
        buf.flip();

        // flip源码
        //  public final Buffer flip() {
        //    limit = position;   // 设置limit = position(5)
        //    position = 0;       // 设置position = 0
        //    mark = -1;          // 设置mark = -1
        //    return this;
        //  }

        // position = 0 / limit = 5 / capacity = 5
        System.out.println(buf.get());
        // position = 1 / limit = 5 / capacity = 5
        System.out.println(buf.get());
        // position = 2 / limit = 5 / capacity = 5
        System.out.println(buf.get());
        // position = 3 / limit = 5 / capacity = 5
        System.out.println(buf.get());
        // position = 4 / limit = 5 / capacity = 5
        System.out.println(buf.get());
    }
}
```
### (5).Buffer API
```
public abstract class Buffer {
	// 返回此缓冲区的容量  
    public final int capacity()
	// 返回此缓冲区的位置  
    public final int position()
	// 设置此缓冲区的位置  
    public final Buffer position (int newPositio)
	// 返回此缓冲区的限制  
    public final int limit()
	// 设置此缓冲区的限制  
    public final Buffer limit (int newLimit)
	// 在此缓冲区的位置设置标记  
    public final Buffer mark()
	// 将此缓冲区的位置重置为以前标记的位置  
    public final Buffer reset()  
	// 清除此缓冲区, 即将各个标记恢复到初始状态,但是数据并没有真正擦除,后面操作会覆盖  
    public final Buffer clear()
	// 反转此缓冲区  
    public final Buffer flip()
	// 重绕此缓冲区  
    public final Buffer rewind()
	// 返回当前位置与限制之间的元素数  
    public final int remaining()
	// 告知在当前位置和限制之间是否有元素  
    public final boolean hasRemaining()
	// 告知此缓冲区是否为只读缓冲区  
    public abstract boolean isReadOnly()
	// 告知此缓冲区是否具有可访问的底层实现数组  
    public abstract boolean hasArray()
	// 返回此缓冲区的底层实现数组  
    public abstract Object array()
	// 返回此缓冲区的底层实现数组中第一个缓冲区元素的偏移量
    public abstract int arrayOffset()
	// 告知此缓冲区是否为直接缓冲区
    public abstract boolean isDirect()
}

```
