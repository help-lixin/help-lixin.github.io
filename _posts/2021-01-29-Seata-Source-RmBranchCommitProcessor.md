---
layout: post
title: 'Seata 分支事务处理之RmBranchCommitProcessor(六)'
date: 2021-01-29
author: 李新
tags: Seata-AT源码
---

### (1).  概述
> 这一节,主要分析:TC通知所有的参与者(RM),进行commit.   

### (2). RmBranchCommitProcessor
> RmNettyRemotingClient在初始化时(registerProcessor),是有指定code与RemotingProcessor的映射关系的.在TC有事件时,会触发回调.   
> TYPE_BRANCH_COMMIT(3) ==> RmBranchCommitProcessor   

### (3). 要先聊一下:DefaultRMHandler
> DefaultRMHandler是什么?    
> 答:DefaultRMHandler是TransactionMessageHandler的实现类,它主要负责接受TC发送过来的消息,并转交给业务处理.  
> DefaultRMHandler在什么时候初始化的?   
> 答:DefaultRMHandler是在RMClient.init方法时初始化的(DefaultRMHandler.get())   
> DefaultRMHandler初始化时做了什么?   
> 答:DefaultRMHandler在初始化时,会通过SPI加载:AbstractRMHandler的所有实现,并缓存在MAP容器里.    

```
public class DefaultRMHandler extends AbstractRMHandler {
	// *****************************************************
	// 所有AbstractRMHandler的实现容器集合
	// {
	//	TCC=io.seata.rm.tcc.RMHandlerTCC@78eaffd7, 
	//  AT=io.seata.rm.RMHandlerAT@762d0eba, 
	//  SAGA=io.seata.saga.rm.RMHandlerSaga@721294a7, 
	//  XA=io.seata.rm.RMHandlerXA@37100481
	// }
	// *****************************************************
	protected static Map<BranchType, AbstractRMHandler> allRMHandlersMap = new ConcurrentHashMap<>();
	
	// 3.1 DefaultRMHandler.get()
	public static AbstractRMHandler get() {
		return DefaultRMHandler.SingletonHolder.INSTANCE;
	} // end
	
	// 3.2 懒汉模式
	private static class SingletonHolder {
		private static AbstractRMHandler INSTANCE = new DefaultRMHandler();
	} //end
	
	// 3.3 构造器
	protected DefaultRMHandler() {
		initRMHandlers();
	} //end

	// 3.4 initRMHandlers
	protected void initRMHandlers() {
		// 3.5 通过SPI加载:AbstractRMHandler的所有实现类,并注册到:DefaultRMHandler.allRMHandlersMap容器里.
		List<AbstractRMHandler> allRMHandlers = EnhancedServiceLoader.loadAll(AbstractRMHandler.class);
		if (CollectionUtils.isNotEmpty(allRMHandlers)) {
			for (AbstractRMHandler rmHandler : allRMHandlers) {
				
				allRMHandlersMap.put(rmHandler.getBranchType(), rmHandler);
			}
		}
	}// end 
}
```

### (4). RmBranchCommitProcessor.process
```
public class RmBranchCommitProcessor implements RemotingProcessor {

    private static final Logger LOGGER = LoggerFactory.getLogger(RmBranchCommitProcessor.class);
	
	//  rmNettyRemotingClient.setTransactionMessageHandler(DefaultRMHandler.get());
	// 在RMClient初始化时通过静态工厂获取到的
	// io.seata.rm.DefaultRMHandler
    private TransactionMessageHandler handler;

	// RmNettyRemotingClient
    private RemotingClient remotingClient;

	
	// 3.1 在RmNettyRemotingClient.registerProcessor处构建的:RmBranchCommitProcessor
    public RmBranchCommitProcessor(TransactionMessageHandler handler, RemotingClient remotingClient) {
        this.handler = handler;
        this.remotingClient = remotingClient;
    }

	// 3.2 等待TC通知commit
    @Override
    public void process(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
		// 获得TC的远程地址
        String remoteAddress = NetUtil.toStringAddress(ctx.channel().remoteAddress());
		// 获得消息体
        Object msg = rpcMessage.getBody();
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("rm client handle branch commit process:" + msg);
        }
		
		// 3.3 委托给本地:handleBranchCommit
        handleBranchCommit(rpcMessage, remoteAddress, (BranchCommitRequest) msg);
    }

	// // 3.3 handleBranchCommit
    private void handleBranchCommit(RpcMessage request, String serverAddress, BranchCommitRequest branchCommitRequest) {
        // 定义返回给:TC的报文信息
		BranchCommitResponse resultMessage;
		
		// **********************************************************
		// 4. 把事件委派给:DefaultRMHandler.onRequest()方法
		// **********************************************************
        resultMessage = (BranchCommitResponse) handler.onRequest(branchCommitRequest, null);
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("branch commit result:" + resultMessage);
        }
        try {
			// **********************************************************
			// 向TC发送消息,告之commit状态.
			// **********************************************************
            this.remotingClient.sendAsyncResponse(serverAddress, request, resultMessage);
        } catch (Throwable throwable) {
            LOGGER.error("branch commit error: {}", throwable.getMessage(), throwable);
        }
    }
}
```
### (5). DefaultRMHandler.onRequest
> DefaultRMHandler属于:AbstractRMHandler的子类.

```
// AbstractRMHandler.onRequest
public AbstractResultMessage onRequest(AbstractMessage request, RpcContext context) {
	// 对消息进行判断,只处理:AbstractTransactionRequestToRM的消息
	if (!(request instanceof AbstractTransactionRequestToRM)) {
		throw new IllegalArgumentException();
	}
	// 把request强制转换成:AbstractTransactionRequestToRM
	AbstractTransactionRequestToRM transactionRequest = (AbstractTransactionRequestToRM)request;
	
	// ******************************************************************
	// 5. 设置:setRMInboundMessageHandler为:DefaultRMHandler对象
	//    transactionRequest.setRMInboundMessageHandler(DefaultRMHandler.this);
	// ******************************************************************
	transactionRequest.setRMInboundMessageHandler(this);
	
	// ******************************************************************
	// transactionRequest.handle是肯定会回调:DefaultRMHandler.handle方法的
	// ******************************************************************
	return transactionRequest.handle(context);
} // end onRequest

// rm接受到tc事件类型为(BranchCommitRequest)
public class BranchCommitRequest extends AbstractBranchEndRequest {

    @Override
    public short getTypeCode() {
        return MessageType.TYPE_BRANCH_COMMIT;
    }

    @Override
    public AbstractTransactionResponse handle(RpcContext rpcContext) {
		// handler == DefaultRMHandler
        return handler.handle(this);
    }
}
```
### (6). DefaultRMHandler.handle
```
// domain
// allRMHandlersMap = {
//	TCC=io.seata.rm.tcc.RMHandlerTCC@78eaffd7, 
//  AT=io.seata.rm.RMHandlerAT@762d0eba, 
//  SAGA=io.seata.saga.rm.RMHandlerSaga@721294a7, 
//  XA=io.seata.rm.RMHandlerXA@37100481
// }
protected static Map<BranchType, AbstractRMHandler> allRMHandlersMap = new ConcurrentHashMap<>();

// methods
protected AbstractRMHandler getRMHandler(BranchType branchType) {
	return allRMHandlersMap.get(branchType);
}

// allRMHandlersMap在DefaultRMHandler初始化时,通过SPI加载过了
// 
public BranchCommitResponse handle(BranchCommitRequest request) {
	// getRMHandler(request.getBranchType())
	return getRMHandler(request.getBranchType()).handle(request);
}
```
### (7). RMHandlerAT.handle
> RMHandlerAT属于:AbstractRMHandler的子类.

```
// AbstractRMHandler.handle(BranchCommitRequest request)
public BranchCommitResponse handle(BranchCommitRequest request) {
	// 7.1 创建分支提交Response
	BranchCommitResponse response = new BranchCommitResponse();
	// 7.2 创建:匿名AbstractCallback
	// 7.3 调用:exceptionHandleTemplate(AbstractCallback)
	exceptionHandleTemplate(new AbstractCallback<BranchCommitRequest, BranchCommitResponse>() {
		@Override
		public void execute(BranchCommitRequest request, BranchCommitResponse response)
			throws TransactionException {
			 // ****************************************************************
			 // 8. doBranchCommit
			 // ****************************************************************
			doBranchCommit(request, response);
		}
	}, request, response);
	return response;
} // end handle


public <T extends AbstractTransactionRequest, S extends AbstractTransactionResponse> void exceptionHandleTemplate(Callback<T, S> callback, T request, S response) {
	try {
		// 7.4 先调用:execute
		callback.execute(request, response);
		// 7.6 然后调用:onSuccess
		callback.onSuccess(request, response);
	} catch (TransactionException tex) {
		LOGGER.error("Catch TransactionException while do RPC, request: {}", request, tex);
		callback.onTransactionException(request, response, tex);
	} catch (RuntimeException rex) {
		LOGGER.error("Catch RuntimeException while do RPC, request: {}", request, rex);
		callback.onException(request, response, rex);
	}
} //end exceptionHandleTemplate
```
### (8). AbstractRMHandler.doBranchCommit
```
protected void doBranchCommit(BranchCommitRequest request, BranchCommitResponse response)
        throws TransactionException {
	 // 获得TC发送过来的消息内容
	 // xid
	String xid = request.getXid();
	// branchdId
	long branchId = request.getBranchId();
	// resourceId
	String resourceId = request.getResourceId();
	// 应用程序数据
	String applicationData = request.getApplicationData();
	if (LOGGER.isInfoEnabled()) {
		LOGGER.info("Branch committing: " + xid + " " + branchId + " " + resourceId + " " + applicationData);
	}
	
	// ***************************************************************************
	// getResourceManager() == DefaultResourceManager
	// 委托给:DefaultResourceManager处理分支提交(DataSourceManager.branchCommit)
	// ***************************************************************************
	BranchStatus status = getResourceManager().branchCommit(request.getBranchType(), xid, branchId, resourceId,
		applicationData);
	
	// 构建分支提交返回信息.
	response.setXid(xid);
	response.setBranchId(branchId);
	response.setBranchStatus(status);
	
	if (LOGGER.isInfoEnabled()) {
		LOGGER.info("Branch commit result: " + status);
	}
}
```
### (9). DataSourceManager.branchCommit
```
public BranchStatus branchCommit(BranchType branchType, String xid, long branchId, String resourceId,
                                     String applicationData) throws TransactionException {
	
	// *****************************************************************
	// 10. 委托给:AsyncWorker类处理
	// *****************************************************************
	// Seata的设计是:commit/rollback是异步处理的,AsyncWorker把数据放到队列里.
	return asyncWorker.branchCommit(branchType, xid, branchId, resourceId, applicationData);
}
```
### (10). AsyncWorker.branchCommit
> AsyncWorker.branchCommit方法只是把数据放入到了队列,那么,在什么时候执行呢?   

```
// domain 
// 队列
private static final BlockingQueue<Phase2Context> ASYNC_COMMIT_BUFFER = new LinkedBlockingQueue<>(
        ASYNC_COMMIT_BUFFER_LIMIT);

// methods
public BranchStatus branchCommit(BranchType branchType, String xid, long branchId, String resourceId,
                                     String applicationData) throws TransactionException {
	// 10.1 把参数包装成:Phase2Context
	// 10.2 offer入队列.
	if (!ASYNC_COMMIT_BUFFER.offer(new Phase2Context(branchType, xid, branchId, resourceId, applicationData))) {
		LOGGER.warn("Async commit buffer is FULL. Rejected branch [{}/{}] will be handled by housekeeping later.", branchId, xid);
	}
	return BranchStatus.PhaseTwo_Committed;
}
```
### (11). AsyncWorker何时执行队列里的数据(commit)呢?

```
public synchronized void init() {
	LOGGER.info("Async Commit Buffer Limit: {}", ASYNC_COMMIT_BUFFER_LIMIT);
	ScheduledExecutorService timerExecutor = new ScheduledThreadPoolExecutor(1, new NamedThreadFactory("AsyncWorker", 1, true));
	timerExecutor.scheduleAtFixedRate(() -> {
		try {
			// 11.1 调用本地:doBranchCommits
			doBranchCommits();
		} catch (Throwable e) {
			LOGGER.info("Failed at async committing ... {}", e.getMessage());
		}
	}, 10, 1000 * 1, TimeUnit.MILLISECONDS);  // 每一秒执行一次
}// end init

private void doBranchCommits() {
	if (ASYNC_COMMIT_BUFFER.isEmpty()) {
		return;
	}
	
	Map<String, List<Phase2Context>> mappedContexts = new HashMap<>(DEFAULT_RESOURCE_SIZE);
	List<Phase2Context> contextsGroupedByResourceId;
	
	// 11.2 批量获取队列里的数据
	while (!ASYNC_COMMIT_BUFFER.isEmpty()) {
		Phase2Context commitContext = ASYNC_COMMIT_BUFFER.poll();
		contextsGroupedByResourceId = CollectionUtils.computeIfAbsent(mappedContexts, commitContext.resourceId, key -> new ArrayList<>());
		contextsGroupedByResourceId.add(commitContext);
	}


	for (Map.Entry<String, List<Phase2Context>> entry : mappedContexts.entrySet()) {
		Connection conn = null;
		DataSourceProxy dataSourceProxy;
		try {
			
			try {
				// 11.3 确保Connection是存在的
				DataSourceManager resourceManager = (DataSourceManager) DefaultResourceManager.get()
					.getResourceManager(BranchType.AT);
				dataSourceProxy = resourceManager.get(entry.getKey());
				if (dataSourceProxy == null) {
					throw new ShouldNeverHappenException("Failed to find resource on " + entry.getKey());
				}
				conn = dataSourceProxy.getPlainConnection();
			} catch (SQLException sqle) {
				LOGGER.warn("Failed to get connection for async committing on " + entry.getKey(), sqle);
				continue;
			}
			contextsGroupedByResourceId = entry.getValue();
			Set<String> xids = new LinkedHashSet<>(UNDOLOG_DELETE_LIMIT_SIZE);
			Set<Long> branchIds = new LinkedHashSet<>(UNDOLOG_DELETE_LIMIT_SIZE);
			
			
			for (Phase2Context commitContext : contextsGroupedByResourceId) {
				xids.add(commitContext.xid);
				branchIds.add(commitContext.branchId);
				int maxSize = Math.max(xids.size(), branchIds.size());
				if (maxSize == UNDOLOG_DELETE_LIMIT_SIZE) {
					try {
						// *********************************************************
						// 12. 批量:清除undo_log表的日志信息(xid,branchId)
						// *********************************************************
						UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType()).batchDeleteUndoLog(
							xids, branchIds, conn);
					} catch (Exception ex) {
						LOGGER.warn("Failed to batch delete undo log [" + branchIds + "/" + xids + "]", ex);
					}
					xids.clear();
					branchIds.clear();
				}
			}

			if (CollectionUtils.isEmpty(xids) || CollectionUtils.isEmpty(branchIds)) {
				return;
			}

			try {
				UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType()).batchDeleteUndoLog(xids,
					branchIds, conn);
			} catch (Exception ex) {
				LOGGER.warn("Failed to batch delete undo log [" + branchIds + "/" + xids + "]", ex);
			}

			if (!conn.getAutoCommit()) {
				conn.commit();
			}
		} catch (Throwable e) {
			LOGGER.error(e.getMessage(), e);
			try {
				if (conn != null) {
					conn.rollback();
				}
			} catch (SQLException rollbackEx) {
				LOGGER.warn("Failed to rollback JDBC resource while deleting undo_log ", rollbackEx);
			}
		} finally {
			if (conn != null) {
				try {
					conn.close();
				} catch (SQLException closeEx) {
					LOGGER.warn("Failed to close JDBC resource while deleting undo_log ", closeEx);
				}
			}
		}
	}
} // end doBranchCommits

```
### (12). UndoLogManager.batchDeleteUndoLog
> AbstractUndoLogManager.batchDeleteUndoLog

```
public void batchDeleteUndoLog(Set<String> xids, Set<Long> branchIds, Connection conn) throws SQLException {
	if (CollectionUtils.isEmpty(xids) || CollectionUtils.isEmpty(branchIds)) {
		return;
	}
	
	int xidSize = xids.size();
	int branchIdSize = branchIds.size();
	// 12.1 构建删除的SQL
	// DELETE FROM undo_log WHERE branch_id =   IN (?)  AND xid IN (?)
	String batchDeleteSql = toBatchDeleteUndoLogSql(xidSize, branchIdSize);
	
	try (PreparedStatement deletePST = conn.prepareStatement(batchDeleteSql)) {
		int paramsIndex = 1;
		for (Long branchId : branchIds) {
			deletePST.setLong(paramsIndex++, branchId);
		}
		for (String xid : xids) {
			deletePST.setString(paramsIndex++, xid);
		}
		
		int deleteRows = deletePST.executeUpdate();
		if (LOGGER.isDebugEnabled()) {
			LOGGER.debug("batch delete undo log size {}", deleteRows);
		}
	} catch (Exception e) {
		if (!(e instanceof SQLException)) {
			e = new SQLException(e);
		}
		throw (SQLException) e;
	}
} // end batchDeleteUndoLog

// 12.2 构建删除的SQL
protected static String toBatchDeleteUndoLogSql(int xidSize, int branchIdSize) {
	StringBuilder sqlBuilder = new StringBuilder(64);
	sqlBuilder.append("DELETE FROM ").append(UNDO_LOG_TABLE_NAME).append(" WHERE  ").append(
		ClientTableColumnsName.UNDO_LOG_BRANCH_XID).append(" IN ");
	appendInParam(branchIdSize, sqlBuilder);
	sqlBuilder.append(" AND ").append(ClientTableColumnsName.UNDO_LOG_XID).append(" IN ");
	appendInParam(xidSize, sqlBuilder);
	return sqlBuilder.toString();
} // end toBatchDeleteUndoLogSql
```
### (12). RmBranchCommitProcessor执行流程图解
!["RmBranchCommitProcessor执行流程图解"](/assets/seata/imgs/seata-RmBranchCommitProcessor-Sequence.jpg.jpg)
### (13). 总结
> AT模式下的第二阶段(commit)的分析的内容还是比较多的.  
> 通过源码分析,AT模式下commit只要入队列(offer)成功了,就直接告之TC成功了.   
