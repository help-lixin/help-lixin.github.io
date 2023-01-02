---
layout: post
title: 'SOFAJRaft源码之SnapshotReader(八)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
上一篇对SnapshotWriter的源码进行了剖析,在这篇,对SnapshotReader进行剖析(相比SnapshotWriter,SnapshotReader还是比较复杂一点)

### (2). Snapshot UML图解
!["Snapshot UML图解"](/assets/jraft/imgs/Snapshot.png)

### (3). LocalSnapshotReader构造器
```
public class LocalSnapshotReader extends SnapshotReader {

    private static final Logger          LOG = LoggerFactory.getLogger(LocalSnapshotReader.class);

    /** Generated reader id*/
    private long                         readerId;
    /** remote peer addr */
    private final Endpoint               addr;
    private final LocalSnapshotMetaTable metaTable;
    private final String                 path;
	// 依赖SnapshotStorage,暂时不理会它
    private final LocalSnapshotStorage   snapshotStorage;
	// 限流,暂时也不理它
    private final SnapshotThrottle       snapshotThrottle;

	
    public LocalSnapshotReader(LocalSnapshotStorage snapshotStorage, 
						       SnapshotThrottle snapshotThrottle, 
							   Endpoint addr,
                               RaftOptions raftOptions, 
							   String path) {
        super();
        this.snapshotStorage = snapshotStorage;
        this.snapshotThrottle = snapshotThrottle;
        this.addr = addr;
		// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_8977291920811/snapshot_99
        this.path = path;
        this.readerId = 0;
        this.metaTable = new LocalSnapshotMetaTable(raftOptions);
    } // end 
}	
```
### (4).  LocalSnapshotReader初始化(LocalSnapshotReader.init)
```
public boolean init(final Void v) {
	// **************************************************************************************
	// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_8977291920811/snapshot_99
	// **************************************************************************************
	final File dir = new File(this.path);
	if (!dir.exists()) {
		LOG.error("No such path {} for snapshot reader.", this.path);
		setError(RaftError.ENOENT, "No such path %s for snapshot reader", this.path);
		return false;
	}
	
	// **************************************************************************************
	// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_8977291920811/snapshot_99/__raft_snapshot_meta
	// **************************************************************************************
	final String metaPath = this.path + File.separator + JRAFT_SNAPSHOT_META_FILE;
	try {
		// **************************************************************************************
		// 读取磁盘上已经序列化了的文件
		// **************************************************************************************
		return this.metaTable.loadFromFile(metaPath);
	} catch (final IOException e) {
		LOG.error("Fail to load snapshot meta {}.", metaPath, e);
		setError(RaftError.EIO, "Fail to load snapshot meta from path %s", metaPath);
		return false;
	}
}
```
### (5). LocalSnapshotReader生成可读的URL(LocalSnapshotReader.generateURIForCopy)
```
public String generateURIForCopy() {
	// 验证下Endpoint
	if (this.addr == null || this.addr.equals(new Endpoint(Utils.IP_ANY, 0))) {
		LOG.error("Address is not specified");
		return null;
	}
	
	if (this.readerId == 0) {
		final SnapshotFileReader reader = new SnapshotFileReader(this.path, this.snapshotThrottle);
		reader.setMetaTable(this.metaTable);
		
		// ********************************************************************
		// 委托给:SnapshotFileReader尝试打开一下文件
		// ********************************************************************
		if (!reader.open()) {
			LOG.error("Open snapshot {} failed.", this.path);
			return null;
		}
		
		// ********************************************************************
		// 通过:FileService生成一个读取文件的唯一id
		// ********************************************************************
		this.readerId = FileService.getInstance().addReader(reader);
		if (this.readerId < 0) {
			LOG.error("Fail to add reader to file_service.");
			return null;
		}
	}
	
	// 最终返回的是网络+读取id
	// remote://localhost:8081/57782045674793870
	return String.format(REMOTE_SNAPSHOT_URI_SCHEME + "%s/%d", this.addr, this.readerId);
} // end 
```
### (6). FileService.addReader
```
// Long : readerId
// FileReader : 文件读取
private final ConcurrentMap<Long, FileReader> fileReaderMap = new ConcurrentHashMap<>();

// 自增数
private final AtomicLong                      nextId        = new AtomicLong();

public long addReader(final FileReader reader) {
	// 生成一个自增数
	final long readerId = this.nextId.getAndIncrement();
	// 通过map Hold住:FileReader
	if (this.fileReaderMap.putIfAbsent(readerId, reader) == null) {
		return readerId;
	} else {
		return -1L;
	}
}
```
### (7). FileService.handleGetFile
```
// 前面分析仅仅只是一个生产可读的URL请求,在这里,是读取URL请求,进行处理来着的. 
public Message handleGetFile(final GetFileRequest request, final RpcRequestClosure done) {
	if (request.getCount() <= 0 || request.getOffset() < 0) {
		return RpcFactoryHelper //
			.responseFactory() //
			.newResponse(GetFileResponse.getDefaultInstance(), RaftError.EREQUEST, "Invalid request: %s", request);
	}
	
	// ******************************************************************************
	// 根据readerId来读取快照
	// ******************************************************************************
	final FileReader reader = this.fileReaderMap.get(request.getReaderId());
	if (reader == null) {
		return RpcFactoryHelper //
			.responseFactory() //
			.newResponse(GetFileResponse.getDefaultInstance(), RaftError.ENOENT, "Fail to find reader=%d",
				request.getReaderId());
	}

	if (LOG.isDebugEnabled()) {
		LOG.debug("GetFile from {} path={} filename={} offset={} count={}", done.getRpcCtx().getRemoteAddress(),
			reader.getPath(), request.getFilename(), request.getOffset(), request.getCount());
	}

	final ByteBufferCollector dataBuffer = ByteBufferCollector.allocate();
	final GetFileResponse.Builder responseBuilder = GetFileResponse.newBuilder();
	try {
		final int read = reader
			.readFile(dataBuffer, request.getFilename(), request.getOffset(), request.getCount());
		responseBuilder.setReadSize(read);
		responseBuilder.setEof(read == FileReader.EOF);
		final ByteBuffer buf = dataBuffer.getBuffer();
		BufferUtils.flip(buf);
		if (!buf.hasRemaining()) {
			// skip empty data
			responseBuilder.setData(ByteString.EMPTY);
		} else {
			// TODO check hole
			responseBuilder.setData(ZeroByteStringHelper.wrap(buf));
		}
		return responseBuilder.build();
	} catch (final RetryAgainException e) {
		return RpcFactoryHelper //
			.responseFactory() //
			.newResponse(GetFileResponse.getDefaultInstance(), RaftError.EAGAIN,
				"Fail to read from path=%s filename=%s with error: %s", reader.getPath(), request.getFilename(),
				e.getMessage());
	} catch (final IOException e) {
		LOG.error("Fail to read file path={} filename={}", reader.getPath(), request.getFilename(), e);
		return RpcFactoryHelper //
			.responseFactory() //
			.newResponse(GetFileResponse.getDefaultInstance(), RaftError.EIO,
				"Fail to read from path=%s filename=%s", reader.getPath(), request.getFilename());
	}
}
```
### (8). 总结
从SnapshotReader接口签名上来看就是生成一个可读取的URL,而SnapshotReader它最主要的目的是:产生一个readId,然后,通过readId与FileReader进行关联,下次,发起请求时,根据这个readId找到:FileReader读取文件来着的. 