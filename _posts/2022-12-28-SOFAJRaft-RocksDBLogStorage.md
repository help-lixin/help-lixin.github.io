---
layout: post
title: 'SOFAJRaft源码之RocksDBLogStorage常用方法剖析(三)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
在前面对RocksDBLogStorage的初始化进行了剖析,在这里,主要剖析追加日志和获取日志.

### (2). 追加日志(RocksDBLogStorage.appendEntry)

```
public boolean appendEntry(final LogEntry entry) {
	if (entry.getType() == EntryType.ENTRY_TYPE_CONFIGURATION) { // 判断日志类型是否为配置信息
		return executeBatch(batch -> addConfBatch(entry, batch)); // 针对列族(Configuration)进行操作.
	} else {
		this.readLock.lock();
		try {
			if (this.db == null) {
				LOG.warn("DB not initialized or destroyed in data path: {}.", this.path);
				return false;
			}
			
			// 这个不用理会,是一个钩子回调函数来着的
			final WriteContext writeCtx = newWriteContext();
			
			// 日志索引(100)
			final long logIndex = entry.getId().getIndex();
			// 对整个日志进行编码
			final byte[] valueBytes = this.logEntryEncoder.encode(entry);
			
			// 这个也不用管,返回的byte实际还是:valueBytes
			// 只是调用了一下WriteContext.finishJob()方法.
			final byte[] newValueBytes = onDataAppend(logIndex, valueBytes, writeCtx);
			// 调用了一下WriteContext.startJob()方法.
			writeCtx.startJob();
			// ***********************************************************************
			// 操作RoksDB,在列族(default)进行数据的存储
			// ***********************************************************************
			this.db.put(this.defaultHandle, this.writeOptions, getKeyBytes(logIndex), newValueBytes);
			// 调用了一下WriteContext.joinAll()方法.
			writeCtx.joinAll();
			
			if (newValueBytes != valueBytes) {
				doSync();
			}
			return true;
		} catch (final RocksDBException | IOException e) {
			LOG.error("Fail to append entry.", e);
			return false;
		} catch (final InterruptedException e) {
			Thread.currentThread().interrupt();
			return false;
		} finally {
			this.readLock.unlock();
		}
	}
}
```
### (3). 获取日志(RocksDBLogStorage.getEntry)
```
public LogEntry getEntry(final long index) {
	this.readLock.lock();
	try {
		if (this.hasLoadFirstLogIndex && index < this.firstLogIndex) {
			return null;
		}
		
		// ***************************************************************
		// 委托给了另一个方法
		// ***************************************************************
		return getEntryFromDB(index);
	} catch (final RocksDBException | IOException e) {
		LOG.error("Fail to get log entry at index {} in data path: {}.", index, this.path, e);
	} finally {
		this.readLock.unlock();
	}
	return null;
}
```
### (4). 获取日志(RocksDBLogStorage.getEntryFromDB)
```
LogEntry getEntryFromDB(final long index) throws IOException, RocksDBException {
  // 1. 把index转换成key数组
  final byte[] keyBytes = getKeyBytes(index);
  // 2. 委派给:getValueFromRocksDB方法,根据index获取数据
  final byte[] bs = onDataGet(index, getValueFromRocksDB(keyBytes));
  if (bs != null) {
	  // 3. bs数组实际就是:LogEntry编码存储的结果,在这里对数组进行解码而已
	  final LogEntry entry = this.logEntryDecoder.decode(bs);
	  if (entry != null) {
		  return entry;
	  } else {
		  LOG.error("Bad log entry format for index={}, the log data is: {}.", index, BytesUtil.toHex(bs));
		  // invalid data remove? TODO
		  return null;
	  }
  }
  return null;
}
```
### (5). 获取日志(RocksDBLogStorage.getValueFromRocksDB)
```
protected byte[] getValueFromRocksDB(final byte[] keyBytes) throws RocksDBException {
	checkState();
	// So Easy,指定列族(default)和key,获取value
	return this.db.get(this.defaultHandle, keyBytes);
}
```
### (6). 获取日志的索引信息(RocksDBLogStorage.getFirstLogIndex)
```
public long getFirstLogIndex() {
	this.readLock.lock();
	RocksIterator it = null;
	try {
		if (this.hasLoadFirstLogIndex) {
			return this.firstLogIndex;
		}
		checkState();
		// *************************************************************************
		// 1. 通过Iterator来遍历列族(default)的数据
		// *************************************************************************
		it = this.db.newIterator(this.defaultHandle, this.totalOrderReadOptions);
		
		// *************************************************************************
		// 2. 跳转到第一列数据
		// *************************************************************************
		it.seekToFirst();
		if (it.isValid()) { 
			// ret为:index
			final long ret = Bits.getLong(it.key(), 0);
			// *************************************************************************
			// 3. 操作列族(Configuration),保存:firstLogIndex
			// *************************************************************************
			saveFirstLogIndex(ret);
			setFirstLogIndex(ret);
			return ret;
		}
		return 1L;
	} finally {
		if (it != null) {
			it.close();
		}
		this.readLock.unlock();
	}
} 
```
### (7). 获取日志的索引信息(RocksDBLogStorage.saveFirstLogIndex)
```
private boolean saveFirstLogIndex(final long firstLogIndex) {
	this.readLock.lock();
	try {
		final byte[] vs = new byte[8];
		Bits.putLong(vs, 0, firstLogIndex);
		checkState();
		// ***************************************************************
		// 重点:
		// 看到没有?在这里居然会操纵列族(Configuration),所以,Configuration列族为元数据来着. 
		// FIRST_LOG_IDX_KEY = meta/firstLogIndex
		// vs = 100
		// ***************************************************************
		this.db.put(this.confHandle, this.writeOptions, FIRST_LOG_IDX_KEY, vs);
		return true;
	} catch (final RocksDBException e) {
		LOG.error("Fail to save first log index {} in {}.", firstLogIndex, this.path, e);
		return false;
	} finally {
		this.readLock.unlock();
	}
}
```
### (8). 总结
> RocksDBLogStorage会创建两个列族(Configuration/default).    
> 1. default列族主要是存储数据(kv存储).  
> 2. Configuration是对default列族的数据进行索引. 