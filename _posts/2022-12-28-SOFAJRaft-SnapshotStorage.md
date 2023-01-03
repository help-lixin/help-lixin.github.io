---
layout: post
title: 'SOFAJRaft源码之SnapshotStorage(九)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
终于到分析的重头戏了,这一小篇主要剖析:SnapshotStorage,它组合了:SnapshotWriter和SnapshotReader的功能,所以,它的职责应该是产生快照以及读取快照. 

### (2). SnapshotStorage UML图解

!["SnapshotStorage UML图解"](/assets/jraft/imgs/SnapshotStorage-ClassDiagram.jpg)

### (3). SnapshotStorage接口签名
```
public interface SnapshotStorage extends Lifecycle<Void>, Storage {

    // 创建写快照
    SnapshotWriter create();

    // 创建读快照
    SnapshotReader open();

    // 读取URL,拷贝数据
    SnapshotReader copyFrom(final String uri, final SnapshotCopierOptions opts);

    // 创建一个异步任务,读取URL,并拷贝数据
    SnapshotCopier startToCopyFrom(final String uri, final SnapshotCopierOptions opts);
}
```
### (4). LocalSnapshotStorage学习入口(LocalSnapshotStorage.setup)
```
public class LocalSnapshotStorageTest extends BaseStorageTest {
    private LocalSnapshotStorage   snapshotStorage;
    private LocalSnapshotMetaTable table;
    private int                    lastSnapshotIndex = 99;

    // ******************************************************************************
	// 方法调用之前先进行测试
	// ******************************************************************************
    @Override
    @Before
    public void setup() throws Exception {
        super.setup();
        // 1. 指定快照存储目录(snapshot_/99)
        String snapshotPath = this.path + File.separator + Snapshot.JRAFT_SNAPSHOT_PREFIX + lastSnapshotIndex;
        FileUtils.forceMkdir(new File(snapshotPath));

        // 2. 创建快照元数据存储表(LocalSnapshotMetaTable)
        this.table = new LocalSnapshotMetaTable(new RaftOptions());
        // 3. 为快照存储元数据表(LocalSnapshotMetaTable)配置元数据(最后快照的索引和term)
        this.table.setMeta(RaftOutter.SnapshotMeta.newBuilder().setLastIncludedIndex(this.lastSnapshotIndex)
            .setLastIncludedTerm(1).build());
        // 4. 持久化快照存储元数据(__raft_snapshot_meta)
        this.table.saveToFile(snapshotPath + File.separator + Snapshot.JRAFT_SNAPSHOT_META_FILE);

        // 5. 创建快照存储对象(LocalSnapshotStorage),需要指定快照存储目录
        this.snapshotStorage = new LocalSnapshotStorage(path, new RaftOptions());
        // 6.LocalSnapshotStorage初始化
        assertTrue(this.snapshotStorage.init(null));
    }
}	
```
### (5). LocalSnapshotStorage.init
> 初始化方法的主要作用如下:     
> 1. 如果目录不存在,则,强制创建快照存储目录.     
> 2. 删除存储快照目录下的临时文件夹.    
> 3. 删除存储快照目录下所有文件夹,留下最大的那个.    

```
@Override
public boolean init(final Void v) {
	final File dir = new File(this.path);
	try {
		FileUtils.forceMkdir(dir);
	} catch (final IOException e) {
		LOG.error("Fail to create directory {}.", this.path, e);
		return false;
	}

	// delete temp snapshot
	if (!this.filterBeforeCopyRemote) {
		// tempSnapshotPath = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/temp
		final String tempSnapshotPath = this.path + File.separator + TEMP_PATH;
		final File tempFile = new File(tempSnapshotPath);
		if (tempFile.exists()) {
			try {
				FileUtils.forceDelete(tempFile);
			} catch (final IOException e) {
				LOG.error("Fail to delete temp snapshot path {}.", tempSnapshotPath, e);
				return false;
			}
		}
	}

	// delete old snapshot
	final List<Long> snapshots = new ArrayList<>();
	// dir = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244
	// 列出目录下所有的文件
	final File[] oldFiles = dir.listFiles();
	// 此时目上录下结构是这样的:
	// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/snapshot_99/__raft_snapshot_meta
	if (oldFiles != null) {
		for (final File sFile : oldFiles) {
			final String name = sFile.getName();
			// 忽略所有文件前缀不是:snapshot_开头的
			if (!name.startsWith(Snapshot.JRAFT_SNAPSHOT_PREFIX)) {
				continue;
			}
			// 截取数字99出来
			final long index = Long.parseLong(name.substring(Snapshot.JRAFT_SNAPSHOT_PREFIX.length()));
			// 添加到集合中
			snapshots.add(index);
		}
	}

	// TODO: add snapshot watcher

	// 获取最后一个快照
	// get last_snapshot_index
	if (!snapshots.isEmpty()) {
		// 排序
		Collections.sort(snapshots);
		// 统计出所有的快照数量
		final int snapshotCount = snapshots.size();

		// ******************************************************************************
		// 如果快照数量大于一个的情况下,把排序后最小的全部目录给删了.
		// ******************************************************************************
		for (int i = 0; i < snapshotCount - 1; i++) {
			final long index = snapshots.get(i);
			final String snapshotPath = getSnapshotPath(index);
			// ******************************************************************************
			// 删除快照
			// ******************************************************************************
			if (!destroySnapshot(snapshotPath)) {
				return false;
			}
		}

		// 取得排序后,最大的索引
		this.lastSnapshotIndex = snapshots.get(snapshotCount - 1);
		// ref实际是一个map(key:lastSnapshotIndex/value:0),每调用一次会进行自增,我感觉是一个引用计数量.
		// 到底是什以,后面再分析.
		ref(this.lastSnapshotIndex);
	}

	return true;
}
```
### (6). LocalSnapshotStorageTest.testCreateOpen
```
@Test
public void testCreateOpen() throws Exception {
	// **********************************************************************
	// create方法,实则注是在快照存储目录下,创建一个临时目录
	// **********************************************************************
	SnapshotWriter writer = this.snapshotStorage.create();


	// 创建快照的元数据(index/term)
	// LastIncludedIndex : 指定下一个index(100)
	RaftOutter.SnapshotMeta wroteMeta = RaftOutter.SnapshotMeta.newBuilder()
		.setLastIncludedIndex(this.lastSnapshotIndex + 1).setLastIncludedTerm(1).build();
	// SnapshotWriter方法配置元数据(未持久化)
	((LocalSnapshotWriter) writer).saveMeta(wroteMeta);
	// 添加一个文件(未持久化)
	writer.addFile("data");
	// 判断lastSnapshotIndex的引用计数器的值
	assertEquals(1, this.snapshotStorage.getRefs(this.lastSnapshotIndex).get());
	
	// **********************************************************************
	// 1. SnapshotWriter里的数据进行持久化
	// 2. 递增:lastSnapshotIndex,并创建数据
	// 3. 删除老的lastSnapshotIndex数据
	// **********************************************************************
	writer.close();
	
	
	// *************************************************************
	// 读取快照存储目录下最大的一个快照文件
	// *************************************************************
	SnapshotReader reader = this.snapshotStorage.open();
	assertTrue(reader.listFiles().contains("data"));
	RaftOutter.SnapshotMeta readMeta = reader.load();
}
```
### (7). SnapshotWriter.create
```
/**
 * create方法,好像就是在存储快照目录下,创建了一个临时目录而已.
 * @param fromEmpty
 * @return
 */
public SnapshotWriter create(final boolean fromEmpty) {
	LocalSnapshotWriter writer = null;
	// noinspection ConstantConditions
	do {
		
		// ************************************************************************************
		// snapshotPath = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/temp
		// 在存储快照目录下,创建temp目录
		// ************************************************************************************
		final String snapshotPath = this.path + File.separator + TEMP_PATH;
		// delete temp
		// TODO: Notify watcher before deleting
		// 如果临时目录存在,并且,传递的参数为:true的情况下,删除快照目录
		if (new File(snapshotPath).exists() && fromEmpty) {
			if (!destroySnapshot(snapshotPath)) {
				break;
			}
		}

		// ************************************************************************************
		// 创建本地写快照(LocalSnapshotWriter),实际指向的是:temp目录
		// ************************************************************************************
		writer = new LocalSnapshotWriter(snapshotPath, this, this.raftOptions);
		// 调用:LocalSnapshotWriter的初始化方法
		if (!writer.init(null)) {
			LOG.error("Fail to init snapshot writer.");
			writer = null;
			break;
		}
	} while (false);
	return writer;
}
```
### (8). SnapshotWriter.close
```
/**
 * close方法的作用:在create方法创建的临时目录里写元数据,并把temp目录改名为新的目录(snapshot_X)
 * @param writer
 * @param keepDataOnError
 * @throws IOException
 */
void close(final LocalSnapshotWriter writer, final boolean keepDataOnError) throws IOException {
	// 在没有设置Status时,code就是:0
	int ret = writer.getCode();
	IOException ioe = null;
	// noinspection ConstantConditions
	do {
		if (ret != 0) {
			break;
		}

		try {
			// **************************************************************
			// 调用LocalSnapshotWriter.sync,把metaTable里的数据进行持久化
			// 此时目录结构如下:
			// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/temp/__raft_snapshot_meta
			// **************************************************************
			if (!writer.sync()) {
				ret = RaftError.EIO.getNumber();
				break;
			}
		} catch (final IOException e) {
			LOG.error("Fail to sync writer {}.", writer.getPath(), e);
			ret = RaftError.EIO.getNumber();
			ioe = e;
			break;
		}

		// lastSnapshotIndex在init方法时,就已经确定了,取的是快照目录下面最大的一个
		final long oldIndex = getLastSnapshotIndex();
		// 当LocalSnapshotWriter的metaTable.meta里没有配置LastIncludedIndex时,那默认值就是:0,
		// 而我这里的值为:100
		final long newIndex = writer.getSnapshotIndex();
		// 新的index与老的index肯定是不能相同来着
		if (oldIndex == newIndex) {
			ret = RaftError.EEXISTS.getNumber();
			break;
		}

		// 定义临时目录
		// tempPath = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/temp
		// rename temp to new
		final String tempPath = this.path + File.separator + TEMP_PATH;
		// newPath = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/snapshot_100
		final String newPath = getSnapshotPath(newIndex);

		// 尝试删除一下新的目录是否存在,如果,存在,就抛出异常
		if (!destroySnapshot(newPath)) {
			LOG.warn("Delete new snapshot path failed, path is {}.", newPath);
			ret = RaftError.EIO.getNumber();
			ioe = new IOException("Fail to delete new snapshot path: " + newPath);
			break;
		}

		LOG.info("Renaming {} to {}.", tempPath, newPath);

		// 强制进行重命令(元子操作)
		// tempPath = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/temp
		// newPath = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/snapshot_100
		if (!Utils.atomicMoveFile(new File(tempPath), new File(newPath), true)) {
			LOG.error("Renamed temp snapshot failed, from path {} to path {}.", tempPath, newPath);
			ret = RaftError.EIO.getNumber();
			ioe = new IOException("Fail to rename temp snapshot from: " + tempPath + " to: " + newPath);
			break;
		}

		// 对新的index进行计数引用
		ref(newIndex);
		this.lock.lock();
		try {
			// 验证新的index与老的index不能相同
			Requires.requireTrue(oldIndex == this.lastSnapshotIndex);
			// 重新赋值lastSnapshotIndex,此时为:100
			this.lastSnapshotIndex = newIndex;
		} finally {
			this.lock.unlock();
		}

		// 对老的index进行计数器递减,如果,递减到了0时,从map中移除.
		unref(oldIndex);
	} while (false);

	if (ret != 0) {
		LOG.warn("Close snapshot writer {} with exit code: {}.", writer.getPath(), ret);
		if (!keepDataOnError) {
			destroySnapshot(writer.getPath());
		}
	}

	if (ioe != null) {
		throw ioe;
	}
}
```
### (9). LocalSnapshotStorage.open
```
public SnapshotReader open() {
	long lsIndex = 0;
	this.lock.lock();
	try {
		if (this.lastSnapshotIndex != 0) {
			lsIndex = this.lastSnapshotIndex;
			ref(lsIndex);
		}
	} finally {
		this.lock.unlock();
	}

	if (lsIndex == 0) {
		LOG.warn("No data for snapshot reader {}.", this.path);
		return null;
	}

	// snapshotPath = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/jraft_test_982705467244/snapshot_100
	final String snapshotPath = getSnapshotPath(lsIndex);
	final SnapshotReader reader = new LocalSnapshotReader(this, this.snapshotThrottle, this.addr, this.raftOptions,
		snapshotPath);
	// 加载元数据
	if (!reader.init(null)) {
		LOG.error("Fail to init reader for path {}.", snapshotPath);
		unref(lsIndex);
		return null;
	}
	return reader;
}
```
### (10). 总结
SnapshotWriter和SnapshotReader是对底层的文件进行操作,而SnapshotStorage则是在它俩之上进行业务组合和叠加. 