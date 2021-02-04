---
layout: post
title: 'Seata 分支事务处理之RmBranchRollbackProcessor(七)'
date: 2021-01-29
author: 李新
tags: Seata-AT源码
---

### (1).  概述
> 这一节,主要分析:TC通知所有的参与者(RM),进行rollback.   

### (2). RmBranchRollbackProcessor
> RmNettyRemotingClient在初始化时(registerProcessor),是有指定code与RemotingProcessor的映射关系的.在TC有事件时,会触发回调.   
> TYPE_BRANCH_COMMIT(5) ==> RmBranchRollbackProcessor    

### (3). RmBranchRollbackProcessor.process
```
public void process(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
	String remoteAddress = NetUtil.toStringAddress(ctx.channel().remoteAddress());
	Object msg = rpcMessage.getBody();
	if (LOGGER.isInfoEnabled()) {
		LOGGER.info("rm handle branch rollback process:" + msg);
	}
	handleBranchRollback(rpcMessage, remoteAddress, (BranchRollbackRequest) msg);
} // end process

private void handleBranchRollback(RpcMessage request, String serverAddress, BranchRollbackRequest branchRollbackRequest) {
	BranchRollbackResponse resultMessage;
	// ***********************************************************************
	// 4. 委托给:DefaultRMHandler.onRequest方法
	// ***********************************************************************
	resultMessage = (BranchRollbackResponse) handler.onRequest(branchRollbackRequest, null);
	if (LOGGER.isDebugEnabled()) {
		LOGGER.debug("branch rollback result:" + resultMessage);
	}
	try {
		this.remotingClient.sendAsyncResponse(serverAddress, request, resultMessage);
	} catch (Throwable throwable) {
		LOGGER.error("send response error: {}", throwable.getMessage(), throwable);
	}
} // end handleBranchRollback
```
### (4). DefaultRMHandler.onRequest
```
public AbstractResultMessage onRequest(AbstractMessage request, RpcContext context) {
	if (!(request instanceof AbstractTransactionRequestToRM)) {
		throw new IllegalArgumentException();
	}
	// 对请求数据进行机制转换
	AbstractTransactionRequestToRM transactionRequest = (AbstractTransactionRequestToRM)request;
	// 设置Handler为this(DefaultRMHandler)
	transactionRequest.setRMInboundMessageHandler(this);
	// **********************************************************************
	// 5. 委托给:DefaultRMHandler.handler
	// **********************************************************************
	// 调用handler
	return transactionRequest.handle(context);
}// end  onRequest

```
### (5). DefaultRMHandler.handler
```
public BranchRollbackResponse handle(BranchRollbackRequest request) {
	// 6. 委托给:RMHandlerAT.handler方法
	return getRMHandler(request.getBranchType()).handle(request);
}// end handle
```
### (6). RMHandlerAT(AbstractRMHandler).handler
```
public BranchRollbackResponse handle(BranchRollbackRequest request) {
	BranchRollbackResponse response = new BranchRollbackResponse();
	
	// 6.1 构建AbstractCallback
	// 6.2 调用handler模块代码
	exceptionHandleTemplate(new AbstractCallback<BranchRollbackRequest, BranchRollbackResponse>() {
		
		// 6.4 execute方法
		@Override
		public void execute(BranchRollbackRequest request, BranchRollbackResponse response)
			throws TransactionException {
			 // *******************************************************************
			 // 7. 委托给:AbstractRMHandler.doBranchRollback
			 // *******************************************************************
			doBranchRollback(request, response);
		}
	}, request, response);
	return response;
} // end handle

public <T extends AbstractTransactionRequest, S extends AbstractTransactionResponse> void exceptionHandleTemplate(Callback<T, S> callback, T request, S response) {
	try {
		// 6.3 委托给: AbstractCallback.execute方法
		callback.execute(request, response);
		callback.onSuccess(request, response);
	} catch (TransactionException tex) {
		LOGGER.error("Catch TransactionException while do RPC, request: {}", request, tex);
		callback.onTransactionException(request, response, tex);
	} catch (RuntimeException rex) {
		LOGGER.error("Catch RuntimeException while do RPC, request: {}", request, rex);
		callback.onException(request, response, rex);
	}
} // end exceptionHandleTemplate
```
### (7). AbstractRMHandler.doBranchRollback
```
protected void doBranchRollback(BranchRollbackRequest request, BranchRollbackResponse response)
        throws TransactionException {
	String xid = request.getXid();
	long branchId = request.getBranchId();
	String resourceId = request.getResourceId();
	String applicationData = request.getApplicationData();
	if (LOGGER.isInfoEnabled()) {
		LOGGER.info("Branch Rollbacking: " + xid + " " + branchId + " " + resourceId);
	}
	
	// *******************************************************************
	// 8. 委托给:DefaultResourceManager.branchRollback方法
	// *******************************************************************
	BranchStatus status = getResourceManager().branchRollback(request.getBranchType(), xid, branchId, resourceId,
		applicationData);
	response.setXid(xid);
	response.setBranchId(branchId);
	response.setBranchStatus(status);
	if (LOGGER.isInfoEnabled()) {
		LOGGER.info("Branch Rollbacked result: " + status);
	}
}
```
### (8). DefaultResourceManager.branchRollback
```
public BranchStatus branchRollback(BranchType branchType, String xid, long branchId,
                                       String resourceId, String applicationData)
        throws TransactionException {
	// 9. 委托给:DataSourceManager.branchRollback方法
	return getResourceManager(branchType).branchRollback(branchType, xid, branchId, resourceId, applicationData);
}
```
### (9). DataSourceManager.branchRollback
```
public BranchStatus branchRollback(BranchType branchType, String xid, long branchId, String resourceId,
                                       String applicationData) throws TransactionException {
        DataSourceProxy dataSourceProxy = get(resourceId);
        if (dataSourceProxy == null) {
            throw new ShouldNeverHappenException();
        }
        try {
			// 10. 委托给:MySQLUndoLogManager.undo
            UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType())
			                     .undo(dataSourceProxy, xid, branchId);
        } catch (TransactionException te) {
            StackTraceLogger.info(LOGGER, te,
                "branchRollback failed. branchType:[{}], xid:[{}], branchId:[{}], resourceId:[{}], applicationData:[{}]. reason:[{}]",
                new Object[]{branchType, xid, branchId, resourceId, applicationData, te.getMessage()});
            if (te.getCode() == TransactionExceptionCode.BranchRollbackFailed_Unretriable) {
                return BranchStatus.PhaseTwo_RollbackFailed_Unretryable;
            } else {
                return BranchStatus.PhaseTwo_RollbackFailed_Retryable;
            }
        }
        return BranchStatus.PhaseTwo_Rollbacked;
    }
```
### (10). MySQLUndoLogManager.undo
```
public void undo(DataSourceProxy dataSourceProxy, String xid, long branchId) throws TransactionException {
	Connection conn = null;
	ResultSet rs = null;
	PreparedStatement selectPST = null;
	boolean originalAutoCommit = true;

	for (; ; ) {
		try {
			conn = dataSourceProxy.getPlainConnection();

			// The entire undo process should run in a local transaction.
			if (originalAutoCommit = conn.getAutoCommit()) {
				conn.setAutoCommit(false);
			}

			// Find UNDO LOG
			selectPST = conn.prepareStatement(SELECT_UNDO_LOG_SQL);
			selectPST.setLong(1, branchId);
			selectPST.setString(2, xid);
			rs = selectPST.executeQuery();

			boolean exists = false;
			while (rs.next()) {
				exists = true;
				
				int state = rs.getInt(ClientTableColumnsName.UNDO_LOG_LOG_STATUS);
				if (!canUndo(state)) { // 校验下undo_log的状态不能是:Normal
					if (LOGGER.isInfoEnabled()) {
						LOGGER.info("xid {} branch {}, ignore {} undo_log", xid, branchId, state);
					}
					return;
				}

				
				String contextString = rs.getString(ClientTableColumnsName.UNDO_LOG_CONTEXT);
				Map<String, String> context = parseContext(contextString);
				byte[] rollbackInfo = getRollbackInfo(rs);

				String serializer = context == null ? null : context.get(UndoLogConstants.SERIALIZER_KEY);
				// undo_log编码解码器
				UndoLogParser parser = serializer == null ? UndoLogParserFactory.getInstance()
					: UndoLogParserFactory.getInstance(serializer);
				BranchUndoLog branchUndoLog = parser.decode(rollbackInfo);

				try {
					// put serializer name to local
					setCurrentSerializer(parser.getName());
					
					List<SQLUndoLog> sqlUndoLogs = branchUndoLog.getSqlUndoLogs();
					if (sqlUndoLogs.size() > 1) {
						Collections.reverse(sqlUndoLogs);
					}
					
					for (SQLUndoLog sqlUndoLog : sqlUndoLogs) {
						TableMeta tableMeta = TableMetaCacheFactory.getTableMetaCache(dataSourceProxy.getDbType()).getTableMeta(
							conn, sqlUndoLog.getTableName(), dataSourceProxy.getResourceId());
						sqlUndoLog.setTableMeta(tableMeta);
						// **********************************************************************
						// 11. 根据SQL类型,获取相应的:AbstractUndoExecutor
						// 11.1 通过SPI,获得UndoExecutorHolder的所有实现类,我这里是:MySQLUndoExecutorHolder.   
						// 11.2 根据SQL类型,从MySQLUndoExecutorHolder获取相应的:AbstractUndoExecutor(MySQLUndoInsertExecutor/MySQLUndoUpdateExecutor/MySQLUndoDeleteExecutor)
						// **********************************************************************
						AbstractUndoExecutor undoExecutor = UndoExecutorFactory.getUndoExecutor(
							dataSourceProxy.getDbType(), sqlUndoLog);
						undoExecutor.executeOn(conn);
					}
				} finally {
					// remove serializer name
					removeCurrentSerializer();
				}
			}

			if (exists) {  // 清除undo_log
				deleteUndoLog(xid, branchId, conn);
				conn.commit();
				if (LOGGER.isInfoEnabled()) {
					LOGGER.info("xid {} branch {}, undo_log deleted with {}", xid, branchId,
						State.GlobalFinished.name());
				}
			} else {
				// 插入到undo_log表中
				insertUndoLogWithGlobalFinished(xid, branchId, UndoLogParserFactory.getInstance(), conn);
				conn.commit();
				if (LOGGER.isInfoEnabled()) {
					LOGGER.info("xid {} branch {}, undo_log added with {}", xid, branchId,
						State.GlobalFinished.name());
				}
			}

			return;
		} catch (SQLIntegrityConstraintViolationException e) {
			// Possible undo_log has been inserted into the database by other processes, retrying rollback undo_log
			if (LOGGER.isInfoEnabled()) {
				LOGGER.info("xid {} branch {}, undo_log inserted, retry rollback", xid, branchId);
			}
		} catch (Throwable e) {
			if (conn != null) {
				try {
					conn.rollback();
				} catch (SQLException rollbackEx) {
					LOGGER.warn("Failed to close JDBC resource while undo ... ", rollbackEx);
				}
			}
			throw new BranchTransactionException(BranchRollbackFailed_Retriable, String
				.format("Branch session rollback failed and try again later xid = %s branchId = %s %s", xid,
					branchId, e.getMessage()), e);

		} finally {
			try {
				if (rs != null) {
					rs.close();
				}
				if (selectPST != null) {
					selectPST.close();
				}
				if (conn != null) {
					if (originalAutoCommit) {
						conn.setAutoCommit(true);
					}
					conn.close();
				}
			} catch (SQLException closeEx) {
				LOGGER.warn("Failed to close JDBC resource while undo ... ", closeEx);
			}
		}
	}
}
```
### (11). AbstractUndoExecutor.executeOn
> 在MySQL下AbstractUndoExecutor的实现有:MySQLUndoInsertExecutor/MySQLUndoUpdateExecutor/MySQLUndoDeleteExecutor.   
> 在这里我以:MySQLUndoUpdateExecutor为例

```
public void executeOn(Connection conn) throws SQLException {
	// IS_UNDO_DATA_VALIDATION_ENABLE = true
	// ************************************************************
	// 12. 委托:AbstractUndoExecutor.dataValidationAndGoOn进行数据校验
	// ************************************************************
	if (IS_UNDO_DATA_VALIDATION_ENABLE && !dataValidationAndGoOn(conn)) {
		return;
	}
	
	try {
		// UPDATE %s SET %s WHERE %s 
		String undoSQL = buildUndoSQL();
		PreparedStatement undoPST = conn.prepareStatement(undoSQL);
		TableRecords undoRows = getUndoRows();
		for (Row undoRow : undoRows.getRows()) {
			ArrayList<Field> undoValues = new ArrayList<>();
			List<Field> pkValueList = getOrderedPkList(undoRows, undoRow, getDbType(conn));
			for (Field field : undoRow.getFields()) {
				if (field.getKeyType() != KeyType.PRIMARY_KEY) {
					undoValues.add(field);
				}
			}
			// 设置参数
			undoPrepare(undoPST, undoValues, pkValueList);
			
			// 执行Update语句
			undoPST.executeUpdate();
		}

	} catch (Exception ex) {
		if (ex instanceof SQLException) {
			throw (SQLException) ex;
		} else {
			throw new SQLException(ex);
		}
	}
} // end executeOn
```
### (12). AbstractUndoExecutor.dataValidationAndGoOn
```
protected boolean dataValidationAndGoOn(Connection conn) throws SQLException {
	// 镜像前
	TableRecords beforeRecords = sqlUndoLog.getBeforeImage();
	// 镜像后
	TableRecords afterRecords = sqlUndoLog.getAfterImage();

	// 镜像前后的数据进行比对
	// Compare current data with before data
	// No need undo if the before data snapshot is equivalent to the after data snapshot.
	Result<Boolean> beforeEqualsAfterResult = DataCompareUtils.isRecordsEquals(beforeRecords, afterRecords);
	if (beforeEqualsAfterResult.getResult()) {
		if (LOGGER.isInfoEnabled()) {
			LOGGER.info("Stop rollback because there is no data change " +
					"between the before data snapshot and the after data snapshot.");
		}
		// no need continue undo.
		return false;
	}

	// 查询当前最新的数据行锁定,并转换行业务模型:TableRecords
	// SELECT * FROM %s WHERE %s FOR UPDATE
	// Validate if data is dirty.
	TableRecords currentRecords = queryCurrentRecords(conn);
	
	// 比较最新的数据行和afterRecords
	// 如果不通过,代表数据已经出现了脏写,需要人工干预了.
	// compare with current data and after image.
	Result<Boolean> afterEqualsCurrentResult = DataCompareUtils.isRecordsEquals(afterRecords, currentRecords);
	if (!afterEqualsCurrentResult.getResult()) {
		// If current data is not equivalent to the after data, then compare the current data with the before 
		// data, too. No need continue to undo if current data is equivalent to the before data snapshot
		Result<Boolean> beforeEqualsCurrentResult = DataCompareUtils.isRecordsEquals(beforeRecords, currentRecords);
		if (beforeEqualsCurrentResult.getResult()) {
			if (LOGGER.isInfoEnabled()) {
				LOGGER.info("Stop rollback because there is no data change " +
						"between the before data snapshot and the current data snapshot.");
			}
			// no need continue undo.
			return false;
		} else {
			if (LOGGER.isInfoEnabled()) {
				if (StringUtils.isNotBlank(afterEqualsCurrentResult.getErrMsg())) {
					LOGGER.info(afterEqualsCurrentResult.getErrMsg(), afterEqualsCurrentResult.getErrMsgParams());
				}
			}
			if (LOGGER.isDebugEnabled()) {
				LOGGER.debug("check dirty datas failed, old and new data are not equal," +
						"tableName:[" + sqlUndoLog.getTableName() + "]," +
						"oldRows:[" + JSON.toJSONString(afterRecords.getRows()) + "]," +
						"newRows:[" + JSON.toJSONString(currentRecords.getRows()) + "].");
			}
			throw new SQLException("Has dirty records when undo.");
		}
	}
	return true;
}
```
### (13). RmBranchRollbackProcessor执行流程图解
!["RmBranchRollbackProcessor执行流程"](/assets/seata/imgs/seata-RmBranchRollbackProcessor-Sequence-Diagram.jpg)

### (14). 总结
> RmBranchRollbackProcessor会执行rollback,在rollback之前会对数据进行校验,然后还原.