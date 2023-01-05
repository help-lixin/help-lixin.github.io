---
layout: post
title: 'SOFAJRaft源码之StateMachine(十五)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
终于轮到剖析StateMachine了,其实,剖析完:FSMCaller之后,StateMachine要不要剖析,也没多大的意义了,只是,还是要让整个源码剖析过程比较完整一些,此处,以:CounterStateMachine为例. 
### (2). StateMachine UML图

!["StateMachine UML图"](/assets/jraft/imgs/StateMachine-ClassDiagram.jpg)

### (3). 快照保存(CounterStateMachine.onSnapshotSave)
```
public void onSnapshotSave(final SnapshotWriter writer, final Closure done) {
	// currVal为业务数据来着的
	final long currVal = this.value.get();
	
	executor.submit(() -> {
		final CounterSnapshotFile snapshot = new CounterSnapshotFile(writer.getPath() + File.separator + "data");
		// *************************************************************************
		// 自己实现数据的写出(CounterSnapshotFile)
		// *************************************************************************
		if (snapshot.save(currVal)) {
			
			// *************************************************************************
			// 添加元数据信息,在前面剖析过源码来着,这里不详解了.
			// 我现在感觉元数据仅仅只是一个标记而已,先不管了,后再剖析KV的时候再看它的实现是咋回事
			// *************************************************************************
			if (writer.addFile("data")) {
				done.run(Status.OK());
			} else {
				done.run(new Status(RaftError.EIO, "Fail to add file to writer"));
			}
		} else {
			done.run(new Status(RaftError.EIO, "Fail to save counter snapshot %s", snapshot.getPath()));
		}
	});
}
```
### (4). CounterSnapshotFile.save
```
public boolean save(final long value) {
	try {
		// 将数据转换成字符串,写出到指定的目录
		FileUtils.writeStringToFile(new File(path), String.valueOf(value));
		return true;
	} catch (IOException e) {
		LOG.error("Fail to save snapshot", e);
		return false;
	}
}
```
### (5). 快照加载(CounterStateMachine.onSnapshotLoad)
```
public boolean onSnapshotLoad(final SnapshotReader reader) {
	if (isLeader()) { // leader 不允许加载快照
		LOG.warn("Leader is not supposed to load snapshot");
		return false;
	}
	
	// 验证下元数据是否存在
	if (reader.getFileMeta("data") == null) {
		LOG.error("Fail to find data file in {}", reader.getPath());
		return false;
	}
	

	final CounterSnapshotFile snapshot = new CounterSnapshotFile(reader.getPath() + File.separator + "data");
	try {
		// ********************************************************************************************
		//  委托给:CounterSnapshotFile加载数据
		// ********************************************************************************************
		this.value.set(snapshot.load());
		return true;
	} catch (final IOException e) {
		LOG.error("Fail to load snapshot from {}", snapshot.getPath());
		return false;
	}

}
```
### (6). CounterSnapshotFile.load
```
public long load() throws IOException {
	// 从磁盘加载数据,转换成long类型
	final String s = FileUtils.readFileToString(new File(path));
	if (!StringUtils.isBlank(s)) {
		return Long.parseLong(s);
	}
	throw new IOException("Fail to load snapshot from " + path + ",content: " + s);
}
```
### (7). 应用日志(CounterStateMachine.onApply)
```
// ***************************************************************************
// onApply是由FSMCaller.onCommitted触发回调的
// ***************************************************************************

public void onApply(final Iterator iter) {
	while (iter.hasNext()) {
		long current = 0;
		CounterOperation counterOperation = null;

		CounterClosure closure = null;

		// 从Iterator里获得数据.
		if (iter.done() != null) {
			// 从Closure中获取数据
			// This task is applied by this node, get value from closure to avoid additional parsing.
			closure = (CounterClosure) iter.done();
			counterOperation = closure.getCounterOperation();
		} else {
			// Have to parse FetchAddRequest from this user log.
			final ByteBuffer data = iter.getData();
			try {
				// 反序列化操作
				counterOperation = SerializerManager.getSerializer(SerializerManager.Hessian2).deserialize(
					data.array(), CounterOperation.class.getName());
			} catch (final CodecException e) {
				LOG.error("Fail to decode IncrementAndGetRequest", e);
			}
			// follower ignore read operation
			if (counterOperation != null && counterOperation.isReadOp()) {
				iter.next();
				continue;
			}
		}

		if (counterOperation != null) {
			switch (counterOperation.getOp()) {
				case GET: // 获取操作
					current = this.value.get();
					LOG.info("Get value={} at logIndex={}", current, iter.getIndex());
					break;
				case INCREMENT: // 自增操作
					final long delta = counterOperation.getDelta();
					final long prev = this.value.get();
					current = this.value.addAndGet(delta);
					LOG.info("Added value={} by delta={} at logIndex={}", prev, delta, iter.getIndex());
					break;
			}

			if (closure != null) {
				closure.success(current);
				closure.run(Status.OK());
			}
		}

		iter.next();
	}
}
```
### (8). 总结
StateMachine的所有方法,都是由FSMCaller进行触发的,相当于StateMachine只是一个回调模板,留给我们开发人员而已. 