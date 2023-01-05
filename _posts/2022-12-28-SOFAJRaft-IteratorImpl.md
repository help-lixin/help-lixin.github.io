---
layout: post
title: 'SOFAJRaft源码之IteratorImpl(十四)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
本来应该要剖析StateMachine的,但是,在剖析FSMCaller(onCommitted方法)时,有涉及到一个类:IteratorImpl,所以,在这一篇主要剖析它了.

### (2). IteratorImplTest
```
@RunWith(value = MockitoJUnitRunner.class)
public class IteratorImplTest {

    private static final String GROUP_ID = "group001";
    private IteratorImpl        iter;
    @Mock
    private NodeImpl            node;
    @Mock
    private FSMCallerImpl       fsmCaller;
    @Mock
    private LogManager          logManager;
    private List<Closure>       closures;
    private AtomicLong          applyingIndex;

    @Before
    public void setup() {
        Mockito.when(this.node.getGroupId()).thenReturn(GROUP_ID);
        Mockito.when(this.fsmCaller.getNode()).thenReturn(node);
        this.applyingIndex = new AtomicLong(0);

        this.closures = new ArrayList<>();
        for (int i = 0; i < 11; i++) {
            this.closures.add(new MockClosure());
            final LogEntry log = new LogEntry(EnumOutter.EntryType.ENTRY_TYPE_NO_OP);
            log.getId().setIndex(i);
            log.getId().setTerm(1);
			// *****************************************************************
			// 创建12条日志,当调用:logManager.getEntry(i)时,返回:LogEntry
			// *****************************************************************
            Mockito.when(this.logManager.getEntry(i)).thenReturn(log);
        }
        
		// 10L 代表着最后commitIndex
        this.iter = new IteratorImpl(fsmCaller, logManager, closures, 0L, 0L, 10L, applyingIndex);
    } // end setup

    @Test
    public void testNext() {
        int i = 1;
        while (iter.isGood()) {
            assertEquals(i, iter.getIndex());
            assertNotNull(iter.done());
            final LogEntry log = iter.entry();
            assertEquals(i, log.getId().getIndex());
            assertEquals(1, log.getId().getTerm());
            iter.next();
            i++;
        }
        assertEquals(i, 11);
    } // end testNext
} // end 	IteratorImplTest
```
### (3). IteratorImpl构造器
```
// start  IteratorImpl 属性
private long                currentIndex;
private final long          committedIndex;
private LogEntry            currEntry = new LogEntry(); // blank entry

// start  IteratorImpl 构造器
public IteratorImpl(final FSMCallerImpl fsmCaller, final LogManager logManager, final List<Closure> closures,
                        final long firstClosureIndex, final long lastAppliedIndex, final long committedIndex,
                        final AtomicLong applyingIndex) {
	super();
	this.fsmCaller = fsmCaller;
	this.fsmCommittedIndex = -1L;
	this.logManager = logManager;
	this.closures = closures;
	// firstClosureIndex = 0
	this.firstClosureIndex = firstClosureIndex;
	// **********************************************************
	// 这个:currentIndex是一个游标来着的
	// 每调用:next()方法一次,currentIndex进行自增,自增之后,是不可以大于:committedIndex
	// currentIndex = 0
	// **********************************************************
	this.currentIndex = lastAppliedIndex;

	// **********************************************************
	// 这个:committedIndex代表着最后提交的index,也就是最大可读取的位置
	// committedIndex = 0
	// **********************************************************
	this.committedIndex = committedIndex;
	// applyingIndex = 0
	this.applyingIndex = applyingIndex;

	// **********************************************************
	// 获取下一个Entry数据
	// **********************************************************
	next();
}
```
### (4). IteratorImpl.next
> 也就是说:IteratorImpl的初始化的时候,会从LogManager加载一条Entry,加载条件是下一条(lastAppliedIndex + 1). 

```
public void next() {
	// 释放当前的Entry
	this.currEntry = null; //release current entry
	
	// currentIndex = 0
	// committedIndex = 10 
	//get next entry
	if (this.currentIndex <= this.committedIndex) {
		// currentIndex = 1 
		++this.currentIndex;
		
		// currentIndex = 1
		// committedIndex = 10 
		if (this.currentIndex <= this.committedIndex) {
			try {
				// 从logManager中获取:Entry
				this.currEntry = this.logManager.getEntry(this.currentIndex);
				if (this.currEntry == null) {
					// ... ... 
				}
			} catch (final LogEntryCorruptedException e) {
				// ... ... 
			}
			// 修改:applyingIndex为:currentIndex(1)
			this.applyingIndex.set(this.currentIndex);
		}
	}
}
```
### (5). IteratorImpl.isGood
> isGood用来验证:currentIndex要小于committedIndex,只有这样,才能继续调用:next获取数据

```
public boolean isGood() {
	// currentIndex = 1
	// committedIndex = 10 
	return this.currentIndex <= this.committedIndex && !hasError();
}
```
### (6). IteratorImpl.entry
```
public LogEntry entry() {
	return this.currEntry;
}
```
### (7). 总结
IteratorImpl的目的是:允许加载:currentIndex~committedIndex之间的Entry,每调用一次next方法,游标(currentIndex)进行移动,然后,通过LogManager加载游标(currentIndex)处的数据(Entry),再通过currEntry进行Hold住这个Entry. 