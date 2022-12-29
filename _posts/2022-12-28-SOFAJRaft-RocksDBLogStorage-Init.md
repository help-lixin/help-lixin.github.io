---
layout: post
title: 'SOFAJRaft源码之RocksDBLogStorage初始化(二)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
在看源码,我建议还是拆开来看,以点向面去扩展,在这里,主要剖析日志的存储这一块,重点要剖析的对象是:RocksDBLogStorage. 
### (2). UML图
> 在开始看源码之前,先看下与RocksDBLogStorage类相关的UML图


!["RocksDBLogStorage UML图解"](/assets/jraft/imgs/LogStorage-ClassDiagram.jpg)
### (3). RocksDBLogStorage相关package介绍
> 从这里能看出来,在编写JRAFT之前,肯定是有先做设计的.  

```
com.alipay.sofa.jraft.entity          :  RAFT底层最终的存储对象(LogEntry)
com.alipay.sofa.jraft.entity.codec    :  对LogEntry进行编码解码
com.alipay.sofa.jraft.storage         :  存储层的接口定义(LogStorage)
com.alipay.sofa.jraft.storage.impl    :  存储层的实现(RocksDBLogStorage)
```
### (4). RocksDBLogStorage.init
```
// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_5560395931697

public boolean init(final LogStorageOptions opts) {
	Requires.requireNonNull(opts.getConfigurationManager(), "Null conf manager");
	Requires.requireNonNull(opts.getLogEntryCodecFactory(), "Null log entry codec factory");
	// group001
	this.groupId = opts.getGroupId();
	// **********************************************************
	// 进行加锁,注意是:写锁
	// **********************************************************
	this.writeLock.lock();
	try {
		
		if (this.db != null) {
			LOG.warn("RocksDBLogStorage init() in {} already.", this.path);
			return true;
		}
		
		// 从配置信息中,取出编码器与解码器
		this.logEntryDecoder = opts.getLogEntryCodecFactory().decoder();
		this.logEntryEncoder = opts.getLogEntryCodecFactory().encoder();
		
		Requires.requireNonNull(this.logEntryDecoder, "Null log entry decoder");
		Requires.requireNonNull(this.logEntryEncoder, "Null log entry encoder");
		
		// 创建默认的数据库配置信息
		this.dbOptions = createDBOptions();
		
		if (this.openStatistics) { // true
			this.statistics = new DebugStatistics();
			this.dbOptions.setStatistics(this.statistics);
		}

		// 写配置信息
		this.writeOptions = new WriteOptions();
		this.writeOptions.setSync(this.sync);
		
		// 读配置信息
		this.totalOrderReadOptions = new ReadOptions();
		this.totalOrderReadOptions.setTotalOrderSeek(true);

        // *************************************************************
		// 委托给了initAndLoad方法
		// *************************************************************
		return initAndLoad(opts.getConfigurationManager());
	} catch (final RocksDBException e) {
		LOG.error("Fail to init RocksDBLogStorage, path={}.", this.path, e);
		return false;
	} finally {
		this.writeLock.unlock();
	}

} // end init
```
### (5). RocksDBLogStorage.initAndLoad
```
private boolean initAndLoad(final ConfigurationManager confManager) throws RocksDBException {
	this.hasLoadFirstLogIndex = false;
	this.firstLogIndex = 1;
	final List<ColumnFamilyDescriptor> columnFamilyDescriptors = new ArrayList<>();
	
	// 创建列族的配置
	final ColumnFamilyOptions cfOption = createColumnFamilyOptions();
	this.cfOptions.add(cfOption);
	
	// **************************************************************************
	// 创建列族
	// **************************************************************************
	// 第一个列族名称为:Configuration(用于存储配置信息,可以理解它是第一个列族数据的索引来着的)
	// Column family to store configuration log entry.
	columnFamilyDescriptors.add(new ColumnFamilyDescriptor("Configuration".getBytes(), cfOption));
	
	// 第二个列族名称为:default(用于存储真实的数据)
	// Default column family to store user data log entry.
	columnFamilyDescriptors.add(new ColumnFamilyDescriptor(RocksDB.DEFAULT_COLUMN_FAMILY, cfOption));

	// **************************************************************************
	// 操作RocketsDB
	// **************************************************************************
	openDB(columnFamilyDescriptors);
	load(confManager);
	return onInitLoaded();
} // end 
```
### (6). RocksDBLogStorage.openDB
```
private void openDB(final List<ColumnFamilyDescriptor> columnFamilyDescriptors) throws RocksDBException {
	final List<ColumnFamilyHandle> columnFamilyHandles = new ArrayList<>();
	
	final File dir = new File(this.path);
	if (dir.exists() && !dir.isDirectory()) {
		throw new IllegalStateException("Invalid log path, it's a regular file: " + this.path);
	}
	
	// *****************************************************************************
	// RocksDB操作
	// 1. 打开数据库
	// 2. 创建列族(Configuration/default)
	// 3. 通过columnFamilyHandles持有列族(Configuration/default)相应的:ColumnFamilyHandle,后面所有的操作,都要用到它们.
	// *****************************************************************************
	this.db = RocksDB.open(this.dbOptions, this.path, columnFamilyDescriptors, columnFamilyHandles);

	assert (columnFamilyHandles.size() == 2);
	
	// *****************************************************************************
	// Hold住:ColumnFamilyHandle
	// *****************************************************************************
	this.confHandle = columnFamilyHandles.get(0);
	this.defaultHandle = columnFamilyHandles.get(1);
}
```
### (7). 总结
由于RocksDBLogStorage间接实现了Lifecycle,而Lifecycle的init方法是入口,init方法的作用是操作RocksDB,Hold住两个列族(ColumnFamilyHandle),后面所有与DB的操作了用到这两个列族,为什么是两个?那是因为JRAFT想通过一个列族存储数据,另一个列族对数据进行索引. 
