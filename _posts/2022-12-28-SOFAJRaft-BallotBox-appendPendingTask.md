---
layout: post
title: 'SOFAJRaft源码之BallotBox之追加任务(十六)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
本来是要剖析:Replicator的,但是,发现Replicator,它依赖很多的其余组件,所以,这一篇开始,先解决内部组件依赖的剖析,再分析Replicator,这一篇的重头戏是:BallotBox

### (2). BallotBox UML图
!["BallotBox UML图"](/assets/jraft/imgs/BallotBox-ClassDiagram.jpg)

### (3). BallotBoxTest
```
@RunWith(value = MockitoJUnitRunner.class)
public class BallotBoxTest {
    private static final String GROUP_ID = "group001";
    private BallotBox           box;
    @Mock
    private FSMCaller           waiter;
    private ClosureQueueImpl    closureQueue;

    @Before
    public void setup() {
        BallotBoxOptions opts = new BallotBoxOptions();
        this.closureQueue = new ClosureQueueImpl(GROUP_ID);
        opts.setClosureQueue(this.closureQueue);
        opts.setWaiter(this.waiter);

        // ********************************************************************
        // BallotBox初始化需要依赖:ClosureQueue和FSMCaller
		// ********************************************************************
        box = new BallotBox();
        assertTrue(box.init(opts));
    } // end setup
	
	
	// ********************************************************************
	// RAFT的要求是这样的:
	// 在复制日志时,要求过半的节点都把数据复制完成,这条日志才算是完成,才可以应用到状态机,所以,commitAt是肯定会调用状态机的.
	// ********************************************************************
	@Test
	public void testCommitAt() {
		// ********************************************************************
		// 仅第一次需要重置下:pendingIndex
		// ********************************************************************
		assertTrue(box.resetPendingIndex(1));
		
		// ********************************************************************
		// 添加一个任务
		// 注意:在这里指定了三个节点(localhost:8081,localhost:8082,localhost:8083)来着的
		// ********************************************************************
		assertTrue(this.box.appendPendingTask(
			JRaftUtils.getConfiguration("localhost:8081,localhost:8082,localhost:8083"),
			JRaftUtils.getConfiguration("localhost:8081"), new Closure() {

				@Override
				public void run(Status status) {

				}
		}));
			
		// *********************************************************	
		// 这一部份,涉及到日志提交,留到下一篇进行剖析	
		// *********************************************************	
		// ... ...
	} // end testCommitAt
}	
```
### (4). BallotBox属性
```
// 状态机
private FSMCaller                 waiter;

// 前面分析过,里面存储的是:Closure,可以队列尾添加Closure,并且,可以从队首弹出N个Closure.
private ClosureQueue              closureQueue;

private final StampedLock         stampedLock        = new StampedLock();

// 记录:过半节点确认复制完日志的索引
private long                      lastCommittedIndex = 0;

// *************************************************************************
// pendingIndex会随着commitAt进行递增
// resetPendingIndex/clearPendingTasks方法 会对pendingIndex进行重置
// *************************************************************************
private long                      pendingIndex;

// *************************************************************************
// SegmentList 暂时不管它,就当他是一个队列,下一篇,再对这个类进行剖析.
// *************************************************************************
private final SegmentList<Ballot> pendingMetaQueue   = new SegmentList<>(false);
```
### (5). BallotBox.resetPendingIndex
```
public boolean resetPendingIndex(final long newPendingIndex) {
	final long stamp = this.stampedLock.writeLock();
	try {
        
		// 如果:pendingIndex不为零,并且,pendingMetaQueue里有数据的情况下,是不允许进行:resetPendingIndex调用的.
		if (!(this.pendingIndex == 0 && this.pendingMetaQueue.isEmpty())) {
			LOG.error("resetPendingIndex fail, pendingIndex={}, pendingMetaQueueSize={}.", this.pendingIndex, this.pendingMetaQueue.size());
			return false;
		}
		
		// lastCommittedIndex:代表着过半节点确认的index,这时进行重置的index是不能小于:lastCommittedIndex
		if (newPendingIndex <= this.lastCommittedIndex) {
			LOG.error("resetPendingIndex fail, newPendingIndex={}, lastCommittedIndex={}.", newPendingIndex,
				this.lastCommittedIndex);
			return false;
		}
		
	    // 比较简单,赋值
		this.pendingIndex = newPendingIndex;
		this.closureQueue.resetFirstIndex(newPendingIndex);
		return true;
	} finally {
		this.stampedLock.unlockWrite(stamp);
	}
}
```
### (6). BallotBox.appendPendingTask
```
public boolean appendPendingTask(final Configuration conf, final Configuration oldConf, final Closure done) {
	final Ballot bl = new Ballot();
	// *********************************************************************************
	// Ballot辅助类,当调用:Ballot.init方法时:
	// 会把conf的PeerId收集,并,计算出投票数(即:过半以上的节点确认)
	// 会把oldConf的PeerId收集,并,计算出投票数(即:过半以上的节点确认)
	// *********************************************************************************
	if (!bl.init(conf, oldConf)) {
		LOG.error("Fail to init ballot.");
		return false;
	}

	final long stamp = this.stampedLock.writeLock();
	try {
		// 如果:pendingIndex为0以下,则返回Pending任务失败,所以,在添加任务之前,需要让pendingIndex向前推动下.
		if (this.pendingIndex <= 0) {
			LOG.error("Fail to appendingTask, pendingIndex={}.", this.pendingIndex);
			return false;
		}
		
		// ***********************************************************************
		// 添加到队列里
		// ***********************************************************************
		this.pendingMetaQueue.add(bl);
		this.closureQueue.appendPendingClosure(done);
		return true;
	} finally {
		this.stampedLock.unlockWrite(stamp);
	}
}
```
### (7). Ballot.init
> Ballot是一个辅助投票的类,初始化方法会根据节点的数量计算出:投票数. 

```
public class Ballot {	
	// *************************************************************************
	// 所有的节点
	// *************************************************************************
	private final List<UnfoundPeerId> peers    = new ArrayList<>();
	// *************************************************************************
	// 应投票的数量(要求:过半),呆会看init方法就知道了
	// *************************************************************************
	private int                       quorum;
	
	
	private final List<UnfoundPeerId> oldPeers = new ArrayList<>();
	private int                       oldQuorum;
	
	public boolean init(final Configuration conf, final Configuration oldConf) {
		// 由于是new出来的,所以,是线程安全,先清空两个集合和quorum
		this.peers.clear();
		this.oldPeers.clear();
		this.quorum = this.oldQuorum = 0;
		
		
		int index = 0;
		if (conf != null) {
			for (final PeerId peer : conf) {
				this.peers.add(new UnfoundPeerId(peer, index++, false));
			}
		}
		// *************************************************************************
		// peers = localhost:8081,localhost:8082,localhost:8083
		// peers.size() = 3
		// quorum = 3 / 2 + 1
		// 也就是说投票数量要达到:2
		// *************************************************************************
		this.quorum = this.peers.size() / 2 + 1;


		if (oldConf == null) {
			return true;
		}
		index = 0;
		for (final PeerId peer : oldConf) {
			this.oldPeers.add(new UnfoundPeerId(peer, index++, false));
		}
		this.oldQuorum = this.oldPeers.size() / 2 + 1;
		
		return true;
	}// end init
	
} // end 	Ballot
```
### (8). Ballot大体数据结构如下
> 根据上面的参数,大体推测出的JSON结构体,主要是分便理解和分析. 

```
{
   peers: [   
	   {   peerId: { endpoint: { ip: "localhost" , port: 8081 } } , found : false , index : 0  },
	   {   peerId: { endpoint: { ip: "localhost" , port: 8082 } } , found : false , index: 1  },
	   {   peerId: { endpoint: { ip: "localhost" , port: 8083 } } , found : false , index: 2  }
   ],
   quorum: 2,
   oldPeers: [
   	   {   peerId: { endpoint: { ip: "localhost" , port: 8081 } } , found : false , index : 0  }
   ],
   oldQuorum: 1
}
```
### (9). 总结
BallotBox追加任务时,会根据节点的数量计算出投票数(quorum). 