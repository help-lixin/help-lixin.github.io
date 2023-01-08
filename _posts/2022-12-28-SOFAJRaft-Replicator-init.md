---
layout: post
title: 'SOFAJRaft源码之Replicator初始化(十九)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述

### (2). 
```

@RunWith(value = MockitoJUnitRunner.class)
public class ReplicatorTest {

    private static final String GROUP_ID    = "test";
    private ThreadId            id;
    private final RaftOptions   raftOptions = new RaftOptions();
    private TimerManager        timerManager;
    @Mock
    private RaftClientService   rpcService;
    @Mock
    private NodeImpl            node;
    @Mock
    private BallotBox           ballotBox;
    @Mock
    private LogManager          logManager;
    @Mock
    private SnapshotStorage     snapshotStorage;
    private ReplicatorOptions   opts;
	// 服务器节点
    private final PeerId        peerId      = new PeerId("localhost", 8081);

    @Before
    public void setup() {
		// 定时任务管理器
        this.timerManager = new TimerManager(5);

		// 配置
        this.opts = new ReplicatorOptions();
        this.opts.setRaftRpcService(this.rpcService);
        this.opts.setPeerId(this.peerId);
        this.opts.setBallotBox(this.ballotBox);
        this.opts.setGroupId(GROUP_ID);
        this.opts.setTerm(1);
        this.opts.setServerId(new PeerId("localhost", 8082));
        this.opts.setNode(this.node);
        this.opts.setSnapshotStorage(this.snapshotStorage);
        this.opts.setTimerManager(this.timerManager);
        this.opts.setLogManager(this.logManager);
        this.opts.setDynamicHeartBeatTimeoutMs(100);
        this.opts.setElectionTimeoutMs(1000);
		
		// mock配置
        Mockito.when(this.logManager.getLastLogIndex()).thenReturn(10L);
        Mockito.when(this.logManager.getTerm(10)).thenReturn(1L);
        Mockito.when(this.rpcService.connect(this.peerId.getEndpoint())).thenReturn(true);
        Mockito.when(this.node.getNodeMetrics()).thenReturn(new NodeMetrics(true));

        // 对RaftClientService.appendEntries方法进行mock
        // mock send empty entries
        mockSendEmptyEntries();
        
		// ********************************************************************
		// 启动:Replicator
		// ********************************************************************
        this.id = Replicator.start(this.opts, this.raftOptions);
    } // end setup
	
	
    private void mockSendEmptyEntries() {
        this.mockSendEmptyEntries(false);
    }// end mockSendEmptyEntries

    private void mockSendEmptyEntries(final boolean isHeartbeat) {
        final RpcRequests.AppendEntriesRequest request = createEmptyEntriesRequest(isHeartbeat);
        Mockito.when(this.rpcService.appendEntries(eq(this.peerId.getEndpoint()), eq(request), eq(-1), Mockito.any()))
            .thenReturn(new FutureImpl<>());
    } // end mockSendEmptyEntries

    private RpcRequests.AppendEntriesRequest createEmptyEntriesRequest() {
        return this.createEmptyEntriesRequest(false);
    } // end createEmptyEntriesRequest

    private RpcRequests.AppendEntriesRequest createEmptyEntriesRequest(final boolean isHeartbeat) {
        RpcRequests.AppendEntriesRequest.Builder rb = RpcRequests.AppendEntriesRequest.newBuilder() //
            .setGroupId("test") //
            .setServerId(new PeerId("localhost", 8082).toString()) //
            .setPeerId(this.peerId.toString()) //
            .setTerm(1) //
            .setPrevLogIndex(10) //
            .setPrevLogTerm(1) //
            .setCommittedIndex(0);
        if (!isHeartbeat) {
            rb.setData(ByteString.EMPTY);
        }
        return rb.build();
    } // end createEmptyEntriesRequest
}
```
### (3). Replicator.start
```
public static ThreadId start(final ReplicatorOptions opts, final RaftOptions raftOptions) {
	// Replicator的初始化是强制要求:LogManager/BallotBox/Node
	if (opts.getLogManager() == null || opts.getBallotBox() == null || opts.getNode() == null) {
		throw new IllegalArgumentException("Invalid ReplicatorOptions.");
	}
    
	// 通过构造器创建:Replicator
	final Replicator r = new Replicator(opts, raftOptions);
	// localhost:8081
	if (!r.rpcService.connect(opts.getPeerId().getEndpoint())) {
		LOG.error("Fail to init sending channel to {}.", opts.getPeerId());
		// Return and it will be retried later.
		return null;
	}

    // 监控相关的,咱先不理它
	// Register replicator metric set.
	final MetricRegistry metricRegistry = opts.getNode().getNodeMetrics().getMetricRegistry();
	if (metricRegistry != null) {
		try {
			if (!metricRegistry.getNames().contains(r.metricName)) {
				metricRegistry.register(r.metricName, new ReplicatorMetricSet(opts, r));
			}
		} catch (final IllegalArgumentException e) {
			// ignore
		}
	}

    // 通过ThreadId Hold住Replicator
	// ThreadId的方法好像比较简单,加锁,解锁.
	// Start replication
	r.id = new ThreadId(r, r);
	// 加锁,并返回:Replicator对象
	r.id.lock();
	// 监听器
	notifyReplicatorStatusListener(r, ReplicatorEvent.CREATED);
	LOG.info("Replicator={}@{} is started", r.id, r.options.getPeerId());
	r.catchUpClosure = null;
	r.lastRpcSendTimestamp = Utils.monotonicMs();
	
	// ********************************************************************
	// 启动定时任务
	// ********************************************************************
	r.startHeartbeatTimer(Utils.nowMs());
	
	// ****************************************************************************
	// id.unlock in sendEmptyEntries
	// 发送探针请求
	// ****************************************************************************
	r.sendProbeRequest();
	return r.id;
}
```
### (4). Replicator构造器
```
public Replicator(final ReplicatorOptions replicatorOptions, final RaftOptions raftOptions) {
	this.options = replicatorOptions;
	this.nodeMetrics = this.options.getNode().getNodeMetrics();
	// ****************************************************************
	// nextIndex为日志最后提交索引 + 1
	// ****************************************************************
	this.nextIndex = this.options.getLogManager().getLastLogIndex() + 1;
	this.timerManager = replicatorOptions.getTimerManager();
	this.raftOptions = raftOptions;
	this.rpcService = replicatorOptions.getRaftRpcService();
	this.metricName = getReplicatorMetricName(replicatorOptions);
	// **********************************************************
	// 配置状态为创建
	// **********************************************************
	setState(State.Created);
}
```
### (5). Replicator.sendProbeRequest
```
private void sendProbeRequest() {
	sendEmptyEntries(false);
}
```
### (6). Replicator.sendEmptyEntries
```
private void sendEmptyEntries(final boolean isHeartbeat) {
	sendEmptyEntries(isHeartbeat, null);
}
```
### (7). Replicator.sendEmptyEntries
```
private void sendEmptyEntries(final boolean isHeartbeat,
                                  final RpcResponseClosure<AppendEntriesResponse> heartBeatClosure) {
	
	// ************************************************************************
	// 构建追加日志请求.
	// ************************************************************************
	final AppendEntriesRequest.Builder rb = AppendEntriesRequest.newBuilder();
	
	// ... ...
	
	try {
		final long monotonicSendTimeMs = Utils.monotonicMs();

		if (isHeartbeat) { // 是否为心跳请求
			// ... ...
		} else {
			// 
			rb.setData(ByteString.EMPTY);
			final AppendEntriesRequest request = rb.build();
			
			// Sending a probe request.
			this.statInfo.runningState = RunningState.APPENDING_ENTRIES;
			this.statInfo.firstLogIndex = this.nextIndex;
			this.statInfo.lastLogIndex = this.nextIndex - 1;
			this.probeCounter++;
			setState(State.Probe);
			
			
			final int stateVersion = this.version;
			final int seq = getAndIncrementReqSeq();
			
			// **********************************************************************************
			// 追加日志
			// options.getPeerId().getEndpoint() = localhost:8081
			// **********************************************************************************
			final Future<Message> rpcFuture = this.rpcService.appendEntries(
			    this.options.getPeerId().getEndpoint(),
				request, 
				-1, 
				new RpcResponseClosureAdapter<AppendEntriesResponse>() {
					@Override
					public void run(final Status status) {
						onRpcReturned(Replicator.this.id, RequestType.AppendEntries, status, request, getResponse(), seq, stateVersion, monotonicSendTimeMs);
					}
				});
            
			
			addInflight(RequestType.AppendEntries, this.nextIndex, 0, 0, seq, rpcFuture);
		}
		LOG.debug("Node {} send HeartbeatRequest to {} term {} lastCommittedIndex {}", this.options.getNode().getNodeId(), this.options.getPeerId(), this.options.getTerm(), rb.getCommittedIndex());
	} finally {
			unlockId();
	}
}
```
### (8). Replicator.onRpcReturned
```
static void onRpcReturned(final ThreadId id, final RequestType reqType, final Status status, final Message request,
                              final Message response, final int seq, final int stateVersion, final long rpcSendTime) {
	if (id == null) {
		return;
	}
	
	final long startTimeMs = Utils.nowMs();
	Replicator r;
	if ((r = (Replicator) id.lock()) == null) {
		return;
	}

	if (stateVersion != r.version) {
		LOG.debug(
			"Replicator {} ignored old version response {}, current version is {}, request is {}\n, and response is {}\n, status is {}.",
			r, stateVersion, r.version, request, response, status);
		id.unlock();
		return;
	}

	final PriorityQueue<RpcResponse> holdingQueue = r.pendingResponses;
	holdingQueue.add(new RpcResponse(reqType, seq, status, request, response, rpcSendTime));

	if (holdingQueue.size() > r.raftOptions.getMaxReplicatorInflightMsgs()) {
		LOG.warn("Too many pending responses {} for replicator {}, maxReplicatorInflightMsgs={}", holdingQueue.size(), r.options.getPeerId(), r.raftOptions.getMaxReplicatorInflightMsgs());
		r.resetInflights();
		r.setState(State.Probe);
		r.sendProbeRequest();
		return;
	}

	boolean continueSendEntries = false;

	final boolean isLogDebugEnabled = LOG.isDebugEnabled();
	StringBuilder sb = null;
	if (isLogDebugEnabled) {
		sb = new StringBuilder("Replicator ") //
			.append(r) //
			.append(" is processing RPC responses, ");
	}
	try {
		int processed = 0;
		while (!holdingQueue.isEmpty()) {
			final RpcResponse queuedPipelinedResponse = holdingQueue.peek();

			// Sequence mismatch, waiting for next response.
			if (queuedPipelinedResponse.seq != r.requiredNextSeq) {
				if (processed > 0) {
					if (isLogDebugEnabled) {
						sb.append("has processed ") //
							.append(processed) //
							.append(" responses, ");
					}
					break;
				} else {
					// Do not processed any responses, UNLOCK id and return.
					continueSendEntries = false;
					id.unlock();
					return;
				}
			}
			holdingQueue.remove();
			processed++;
			final Inflight inflight = r.pollInflight();
			if (inflight == null) {
				// The previous in-flight requests were cleared.
				if (isLogDebugEnabled) {
					sb.append("ignore response because request not found: ") //
						.append(queuedPipelinedResponse) //
						.append(",\n");
				}
				continue;
			}
			if (inflight.seq != queuedPipelinedResponse.seq) {
				// reset state
				LOG.warn(
					"Replicator {} response sequence out of order, expect {}, but it is {}, reset state to try again.",
					r, inflight.seq, queuedPipelinedResponse.seq);
				r.resetInflights();
				r.setState(State.Probe);
				continueSendEntries = false;
				r.block(Utils.nowMs(), RaftError.EREQUEST.getNumber());
				return;
			}
			try {
				switch (queuedPipelinedResponse.requestType) {
					case AppendEntries:
						continueSendEntries = onAppendEntriesReturned(id, inflight, queuedPipelinedResponse.status,
							(AppendEntriesRequest) queuedPipelinedResponse.request,
							(AppendEntriesResponse) queuedPipelinedResponse.response, rpcSendTime, startTimeMs, r);
						break;
					case Snapshot:
						continueSendEntries = onInstallSnapshotReturned(id, r, queuedPipelinedResponse.status,
							(InstallSnapshotRequest) queuedPipelinedResponse.request,
							(InstallSnapshotResponse) queuedPipelinedResponse.response);
						break;
				}
			} finally {
				if (continueSendEntries) {
					// Success, increase the response sequence.
					r.getAndIncrementRequiredNextSeq();
				} else {
					// The id is already unlocked in onAppendEntriesReturned/onInstallSnapshotReturned, we SHOULD break out.
					break;
				}
			}
		}
	} finally {
		if (isLogDebugEnabled) {
			sb.append("after processed, continue to send entries: ") //
				.append(continueSendEntries);
			LOG.debug(sb.toString());
		}
		if (continueSendEntries) {
			// unlock in sendEntries.
			r.sendEntries();
		}
	}
}
```
### (9). 

### (10). 
 
### (11). 

### (12). 

### (13). 

### (14). 

### (15). 

### (16). 

### (17). 

### (18). 
 
### (19). 
