---
layout: post
title: 'Liquibase源码之UpdateVisitor(三)' 
date: 2021-09-03
author: 李新
tags:  Liquibase
---

### (1). 概述
在前面,剖析了Liquibase会对XML进行解析,解析后数据保存在:DatabaseChangeLog对象里,DatabaseChangeLog承载着所有的业务模型数据.     
那么,Liquibase是何时对DatabaseChangeLog进行加工,做二次处理的呢(即转化成SQL,并应用于数据库的呢)?    

### (2). Liquibase入口在哪?
```
public class Liquibase implements AutoCloseable {
	public void update(Contexts contexts, LabelExpression labelExpression, boolean checkLiquibaseTables) throws LiquibaseException {
		runInScope(() -> {
			// ... ...
			Operation updateOperation = null;
			BufferedLogService bufferLog = new BufferedLogService();
			DatabaseChangeLog changeLog = null;
			
			// ************************************************************************
			// 1. 解析(XML/JSON/YAML/SQLFile),转换成业务模型:DatabaseChangeLog
			// ************************************************************************
			changeLog = getDatabaseChangeLog();

			// ... ...
			
			
			// 创建一个ChangeLog责任链.
			ChangeLogIterator runChangeLogIterator = getStandardChangelogIterator(contexts, labelExpression, changeLog);
			// 这里仅仅是打日志的一个服务而已
			CompositeLogService compositeLogService = new CompositeLogService(true, bufferLog);

			// ************************************************************************
			// 2. 重点:开始执行ChangeLog(SQL)内容了
			// ************************************************************************
			Scope.child(Scope.Attr.logService.name(), compositeLogService, () -> {
				//  *******************************************************************
				// 3. createUpdateVisitor()
				//    调用:ChangeLogIterator.run方法
				//  *******************************************************************
				runChangeLogIterator.run(createUpdateVisitor(), new RuntimeEnvironment(database, contexts, labelExpression));
			});
			
			// ... ...
		});
		
		//  *******************************************************************
		// 3. 创建了一个:UpdateVisitor,执行更新操作
		//  *******************************************************************
		protected UpdateVisitor createUpdateVisitor() {
			return new UpdateVisitor(database, changeExecListener);
		}
}
```
### (3). ChangeLogIterator.run
```
public class ChangeLogIterator {
	
	public void run(ChangeSetVisitor visitor, RuntimeEnvironment env) throws LiquibaseException {
		Logger log = Scope.getCurrentScope().getLog(getClass());
		databaseChangeLog.setRuntimeEnvironment(env);
		try {
			Scope.child(Scope.Attr.databaseChangeLog, databaseChangeLog, new Scope.ScopedRunner() {
				@Override
				public void run() throws Exception {

					List<ChangeSet> changeSetList = new ArrayList<>(databaseChangeLog.getChangeSets());
					if (visitor.getDirection().equals(ChangeSetVisitor.Direction.REVERSE)) {
						Collections.reverse(changeSetList);
					}
					for (ChangeSet changeSet : changeSetList) {
						boolean shouldVisit = true;
						Set<ChangeSetFilterResult> reasonsAccepted = new HashSet<>();
						Set<ChangeSetFilterResult> reasonsDenied = new HashSet<>();
						if (changeSetFilters != null) {
							for (ChangeSetFilter filter : changeSetFilters) {
								ChangeSetFilterResult acceptsResult = filter.accepts(changeSet);
								if (acceptsResult.isAccepted()) {
									reasonsAccepted.add(acceptsResult);
								} else {
									shouldVisit = false;
									reasonsDenied.add(acceptsResult);
									break;
								}
							}
						}

						boolean finalShouldVisit = shouldVisit;
						BufferedLogService bufferLog = new BufferedLogService();
						CompositeLogService compositeLogService = new CompositeLogService(true, bufferLog);
						Scope.child(Scope.Attr.changeSet.name(), changeSet, () -> {
							if (finalShouldVisit && !alreadySaw(changeSet)) {
								//
								// Go validate any change sets with an Executor
								//
								validateChangeSetExecutor(changeSet, env);

								//
								// Execute the visit call in its own scope with a new
								// CompositeLogService and BufferLogService in order
								// to capture the logging for just this change set.  The
								// log is sent to Hub if available
								//
								Map<String, Object> values = new HashMap<>();
								values.put(Scope.Attr.logService.name(), compositeLogService);
								values.put(BufferedLogService.class.getName(), bufferLog);
								Scope.child(values, () -> {
									// ***************************************************************
									// 委托给:UpdateVisitor.visit方法
									// ***************************************************************
									visitor.visit(changeSet, databaseChangeLog, env.getTargetDatabase(), reasonsAccepted);
								});
								markSeen(changeSet);
							} else {
								if (visitor instanceof SkippedChangeSetVisitor) {
									((SkippedChangeSetVisitor) visitor).skipped(changeSet, databaseChangeLog, env.getTargetDatabase(), reasonsDenied);
								}
							}
						});
					}
				}
			});
		} catch (Exception e) {
			throw new LiquibaseException(e);
		} finally {
			databaseChangeLog.setRuntimeEnvironment(null);
		}
	}
}	
```
### (4). UpdateVisitor.visit
```
public class UpdateVisitor implements ChangeSetVisitor {
	
	public void visit(ChangeSet changeSet, DatabaseChangeLog databaseChangeLog, Database database,
	                      Set<ChangeSetFilterResult> filterResults) throws LiquibaseException {
		ChangeSet.RunStatus runStatus = this.database.getRunStatus(changeSet);
		Scope.getCurrentScope().getLog(getClass()).fine("Running Changeset:" + changeSet);
		fireWillRun(changeSet, databaseChangeLog, database, runStatus);
		ExecType execType = null;
		ObjectQuotingStrategy previousStr = this.database.getObjectQuotingStrategy();
		try {
			// *******************************************************
			// 委托给了:ChangeSet.execute方法去执行所有的ChangeLog
			// *******************************************************
			execType = changeSet.execute(databaseChangeLog, execListener, this.database);
		} catch (MigrationFailedException e) {
			fireRunFailed(changeSet, databaseChangeLog, database, e);
			throw e;
		}
		if (!runStatus.equals(ChangeSet.RunStatus.NOT_RAN)) {
			execType = ChangeSet.ExecType.RERAN;
		}
		fireRan(changeSet, databaseChangeLog, database, execType);
		// reset object quoting strategy after running changeset
		this.database.setObjectQuotingStrategy(previousStr);
		this.database.markChangeSetExecStatus(changeSet, execType);

		this.database.commit();
	}
}
```
### (5). ChangeSet.execute
```
public class ChangeSet implements Conditional, ChangeLogChild {
	public ExecType execute(DatabaseChangeLog databaseChangeLog, ChangeExecListener listener, Database database)
	            throws MigrationFailedException {
		Logger log = Scope.getCurrentScope().getLog(getClass());

		if (validationFailed) {
			return ExecType.MARK_RAN;
		}

		long startTime = new Date().getTime();

		ExecType execType = null;

		boolean skipChange = false;

		Executor originalExecutor = setupCustomExecutorIfNecessary(database);
		try {
			Executor executor = Scope.getCurrentScope().getSingleton(ExecutorService.class).getExecutor("jdbc", database);
			// set object quoting strategy
			database.setObjectQuotingStrategy(objectQuotingStrategy);
			
			
			if (database.supportsDDLInTransaction()) {
				database.setAutoCommit(!runInTransaction);
			}

			// 处理注释部份.
			executor.comment("Changeset " + toString(false));
			if (StringUtil.trimToNull(getComments()) != null) {
				String comments = getComments();
				String[] lines = comments.split("\\n");
				for (int i = 0; i < lines.length; i++) {
					if (i > 0) {
						lines[i] = database.getLineComment() + " " + lines[i];
					}
				}
				executor.comment(StringUtil.join(Arrays.asList(lines), "\n"));
			}

			try {
				// 处理preconditions
				if (preconditions != null) {
					preconditions.check(database, databaseChangeLog, this, listener);
				}
			} catch (PreconditionFailedException e) {
				// ... ...
			} catch (PreconditionErrorException e) {
				// ... ...
			} finally {
				database.rollback();
			}
			
			
			
			if (!skipChange) {
				// ... ...

				// ***********************************************************************
				// 挨个遍历Change对象,并调用:Change.executeStatements方法
				// ***********************************************************************
				log.fine("Reading ChangeSet: " + toString());
				for (Change change : getChanges()) {
					if ((!(change instanceof DbmsTargetedChange)) || DatabaseList.definitionMatches(((DbmsTargetedChange) change).getDbms(), database, true)) {
						
						// 运行之前的钩子函数
						if (listener != null) {
							listener.willRun(change, this, changeLog, database);
						}
						if (change.generateStatementsVolatile(database)) {
							executor.comment("WARNING The following SQL may change each run and therefore is possibly incorrect and/or invalid:");
						}

						// **************************************************************
						// 调用每一个Change,生成SQL语句,并执行.
						// **************************************************************
						database.executeStatements(change, databaseChangeLog, sqlVisitors);
						log.info(change.getConfirmationMessage());
						
						// 运行之后的钩子函数
						if (listener != null) {
							listener.ran(change, this, changeLog, database);
						}
					} else {
						log.fine("Change " + change.getSerializedObjectName() + " not included for database " + database.getShortName());
					}
				} //end for
				
				// **************************************************************
				// commit是针对<changeSet></changeSet>内所有的操作,当成一个事务来着的.
				// **************************************************************
				if (runInTransaction) {
					database.commit();
				}
				log.info("ChangeSet " + toString(false) + " ran successfully in " + (new Date().getTime() - startTime + "ms"));
				if (execType == null) {
					execType = ExecType.EXECUTED;
				}
			} else {
				log.fine("Skipping ChangeSet: " + toString());
			}

		} catch (Exception e) {
			try {
				// ***************************************************************
				// rollback事务
				// ***************************************************************
				database.rollback();
			} catch (Exception e1) {
				throw new MigrationFailedException(this, e);
			}
			if ((getFailOnError() != null) && !getFailOnError()) {
				log.info("Change set " + toString(false) + " failed, but failOnError was false.  Error: " + e.getMessage());
				log.fine("Failure Stacktrace", e);
				execType = ExecType.FAILED;
			} else {
				if (e instanceof MigrationFailedException) {
					throw ((MigrationFailedException) e);
				} else {
					throw new MigrationFailedException(this, e);
				}
			}
		} finally {
			// ... ...
		}
		return execType;
	}
}	
```
### (6). 总结
Liquibase会遍历所有Change,Change会根据自己的行为,产生SQL语句,并执行.事务不是针对Change的,而是针对ChangeSet的.  