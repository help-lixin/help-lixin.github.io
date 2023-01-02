---
layout: post
title: 'SOFAJRaft源码之SnapshotWriter(七)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
本来是想剖析SnapshotStorage,但是在用UML对SnapshotStorage进行分解时,底层实际上还是依赖一个比较重的:Snapshot,所以,这一篇改成,剖析:Snapshot
### (2). Snapshot UML图解
!["Snapshot UML图解"](/assets/jraft/imgs/Snapshot.png)
### (3). LocalSnapshotWriter构建器
```
public class LocalSnapshotWriter extends SnapshotWriter {

    private static final Logger          LOG = LoggerFactory.getLogger(LocalSnapshotWriter.class);

    private final LocalSnapshotMetaTable metaTable;
    private final String                 path;
    private final LocalSnapshotStorage   snapshotStorage;

    public LocalSnapshotWriter(String path, LocalSnapshotStorage snapshotStorage, RaftOptions raftOptions) {
        super();
        this.snapshotStorage = snapshotStorage;
        this.path = path;
		// **********************************************************
		// 注意,引入了另一个类:LocalSnapshotMetaTable
		// **********************************************************
        this.metaTable = new LocalSnapshotMetaTable(raftOptions);
    } // end 构造器
}
```
### (4). LocalSnapshotWriter初始化
```
public boolean init(final Void v) {
	final File dir = new File(this.path);
	try {
		// 强制创建文件夹(/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_5774650759797)
		FileUtils.forceMkdir(dir);
	} catch (final IOException e) {
		LOG.error("Fail to create directory {}.", this.path, e);
		setError(RaftError.EIO, "Fail to create directory  %s", this.path);
		return false;
	}
	
	// ********************************************************
	// 判断元数据文件(__raft_snapshot_meta)是否存在,如果存在,则加载
	// ********************************************************
	// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_5774650759797/__raft_snapshot_meta
	final String metaPath = path + File.separator + JRAFT_SNAPSHOT_META_FILE;
	final File metaFile = new File(metaPath);
	try {
		if (metaFile.exists()) {
			// 加载元数据文件,先忽略,因为,第一次是不会有元数据文件的,咋位继续往下剖析.
			return metaTable.loadFromFile(metaPath);
		}
	} catch (final IOException e) {
		LOG.error("Fail to load snapshot meta from {}.", metaPath, e);
		setError(RaftError.EIO, "Fail to load snapshot meta from %s", metaPath);
		return false;
	}
	return true;
}
```
### (5). LocalSnapshotWriter.addFile
```
public boolean addFile(final String fileName, final Message fileMeta) {
	final Builder metaBuilder = LocalFileMeta.newBuilder();
	if (fileMeta != null) {
		metaBuilder.mergeFrom(fileMeta);
	}
	final LocalFileMeta meta = metaBuilder.build();
	// ********************************************************
	// 添加元数据文件,仅仅是把名称和meta放到缓存(map)中而已.
	// 在这里就有点迷茫了,不过我们继续剖析另一个方法(sync).
	// ********************************************************
	return this.metaTable.addFile(fileName, meta);
}
```
### (6). LocalSnapshotWriter.sync
```
public boolean sync() throws IOException {
	// 委托给了:LocalSnapshotMetaTable
	// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_6850709717664/__raft_snapshot_meta
	return this.metaTable.saveToFile(this.path + File.separator + JRAFT_SNAPSHOT_META_FILE);
}
```
### (7). LocalSnapshotMetaTable.saveToFile
```
public boolean saveToFile(String path) throws IOException {
	LocalSnapshotPbMeta.Builder pbMeta = LocalSnapshotPbMeta.newBuilder();
	if (hasMeta()) {
		pbMeta.setMeta(this.meta);
	}
	// *************************************************************************
	// 上面的addFile方法,就是把元数据添加到fileMap里
	// 在这里就是遍历map,添加到:pbMeta里
	// *************************************************************************
	for (Map.Entry<String, LocalFileMeta> entry : this.fileMap.entrySet()) {
		File f = File.newBuilder().setName(entry.getKey()).setMeta(entry.getValue()).build();
		pbMeta.addFiles(f);
	}
	
	// *************************************************************************
	// 又委托给了:ProtoBufFile进行序列化
	// *************************************************************************
	ProtoBufFile pbFile = new ProtoBufFile(path);
	return pbFile.save(pbMeta.build(), this.raftOptions.isSyncMeta());
}
```
### (8). ProtoBufFile.save
```
public boolean save(final Message msg, final boolean sync) throws IOException {
	// Write message into temp file
	final File file = new File(this.path + ".tmp");
	try (final FileOutputStream fOut = new FileOutputStream(file);
			final BufferedOutputStream output = new BufferedOutputStream(fOut)) {
		final byte[] lenBytes = new byte[4];

		// name len + name
		final String fullName = msg.getDescriptorForType().getFullName();
		final int nameLen = fullName.length();
		Bits.putInt(lenBytes, 0, nameLen);
		output.write(lenBytes);
		output.write(fullName.getBytes());
		
		// msg len + msg
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
### (9). LocalSnapshotPbMeta报文内容大概如下
```
files {
  name: "data2"
  meta {
  }
}
files {
  name: "data1"
  meta {
    source: FILE_SOURCE_LOCAL
    checksum: "test"
  }
}
```
### (10). 总结
> SnapshotWriter最终的目的是: 
> 1. 添加元数据文件(并未进行持久化).  
> 2. 删除元数据文件(并未进行持久化). 
> 3. 持久化(sync)元数据文件.  
> 4. 可以列出所有的元数据文件.
