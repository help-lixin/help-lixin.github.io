---
layout: post
title: 'SOFAJRaft源码之RaftMetaStorage(六)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
固名词义,这个接口的主要功能是元数据的存储.

### (2). RaftMetaStorage UML图
!["RaftMetaStorage UML"](/assets/jraft/imgs/RaftMetaStorage-ClassDiagram.jpg)

### (3). RaftMetaStorage.init
```
public boolean init(final RaftMetaStorageOptions opts) {
	if (this.isInited) {
		LOG.warn("Raft meta storage is already inited.");
		return true;
	}
	this.node = opts.getNode();
	this.nodeMetrics = this.node.getNodeMetrics();
	try {
		FileUtils.forceMkdir(new File(this.path));
	} catch (final IOException e) {
		LOG.error("Fail to mkdir {}", this.path, e);
		return false;
	}
	// **********************************************************
	// 委派给load
	// **********************************************************
	if (load()) {
		this.isInited = true;
		return true;
	} else {
		return false;
	}
}
```
### (4). RaftMetaStorage.load
```
private ProtoBufFile newPbFile() {
	return new ProtoBufFile(this.path + File.separator + RAFT_META);
} // end newPbFile

private boolean load() {
	final ProtoBufFile pbFile = newPbFile();
	try {
		// 委派给:ProtoBufFile,加载元数据文件
		final StablePBMeta meta = pbFile.load();
		if (meta != null) {
			this.term = meta.getTerm();
			return this.votedFor.parse(meta.getVotedfor());
		}
		return true;
	} catch (final FileNotFoundException e) {
		return true;
	} catch (final IOException e) {
		LOG.error("Fail to load raft meta storage", e);
		return false;
	}
} // end
```
### (5). RaftMetaStorage.setTerm/setVotedFor
```
public boolean setTerm(final long term) {
	checkState();
	this.term = term;
	return save();
} // end setTerm

public boolean setVotedFor(final PeerId peerId) {
	checkState();
	this.votedFor = peerId;
	return save();
} // end setVotedFor
```
### (6). RaftMetaStorage.save
```
private boolean save() {
	final long start = Utils.monotonicMs();
	final StablePBMeta meta = StablePBMeta.newBuilder() //
		.setTerm(this.term) //
		.setVotedfor(this.votedFor.toString()) //
		.build();
	final ProtoBufFile pbFile = newPbFile();
	try {
		// ******************************************************************
		// 最终是委派给了:ProtoBufFile.save
		// ******************************************************************
		if (!pbFile.save(meta, this.raftOptions.isSyncMeta())) {
			reportIOError();
			return false;
		}
		return true;
	} catch (final Exception e) {
		// ... 
	} finally {
		// ... 
	}
}
```
### (7). ProtoBufFile.save
```
public boolean save(final Message msg, final boolean sync) throws IOException {
	// Write message into temp file
	// ******************************************************************
	// raft_meta.tmp
	// ******************************************************************
	final File file = new File(this.path + ".tmp");
	try (final FileOutputStream fOut = new FileOutputStream(file);
			final BufferedOutputStream output = new BufferedOutputStream(fOut)) {
		final byte[] lenBytes = new byte[4];

		// ******************************************************************
		// name len + name
		// ******************************************************************
		final String fullName = msg.getDescriptorForType().getFullName();
		final int nameLen = fullName.length();
		Bits.putInt(lenBytes, 0, nameLen);
		output.write(lenBytes);
		output.write(fullName.getBytes());
		
		// ******************************************************************
		// msg len + msg
		// ******************************************************************
		final int msgLen = msg.getSerializedSize();
		Bits.putInt(lenBytes, 0, msgLen);
		output.write(lenBytes);
		msg.writeTo(output);
		output.flush();
	}
	
	if (sync) {
		Utils.fsync(file);
	}
	return Utils.atomicMoveFile(file, new File(this.path), sync);
}
```
### (8). 总结
RaftMetaStorage的主要目的是持久化,term和votedFor,报文格式如下( class name length + class name + msg length + msg ).   