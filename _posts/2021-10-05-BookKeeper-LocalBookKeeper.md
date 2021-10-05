---
layout: post
title: 'Apache BookKeeper源码之LocalBookKeeper(三)' 
date: 2021-10-05
author: 李新
tags:  BookKeeper
---

### (1). 概述
前面,通过运行脚本,测试运行了6个Bookies,在这里,我们开始剖析源码,让自己更深入的了解:Bookies是如何构建出来的.

### (2). 配置远程断点
```
# 为了支持远程断点,需要修改运行脚本内容:  /Users/lixin/GitRepository/bookkeeper/bin/bookkeeper

elif [ ${COMMAND} == "localbookie" ]; then
  NUMBER=$1
  shift
  # 增加断点,通过IDEA进行远程调试
  OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005 ${OPTS}"
  # *******************************************************
  # 通过这一行,我们能看出来,localbookie调用的是:LocalBookKeeper类
  # *******************************************************
  exec "${JAVA}" ${OPTS} ${JMX_ARGS} -Dzookeeper.4lw.commands.whitelist='*' org.apache.bookkeeper.util.LocalBookKeeper ${NUMBER} ${BOOKIE_CONF} $@
```
### (3). LocalBookKeeper.main
+ 读取参数中传递要创建Bookies的数量.      
+ 读取参数中传递的配置文件路径,并转换成业务模型(ServerConfiguration).      
+ 委托给内部静态方法(startLocalBookiesInternal),启动:Bookies

```
public static void main(String[] args) {
	// 要运行的Bookies数量和配置文件路径
	// args = {
	// 	6,
	//  /Users/lixin/GitRepository/bookkeeper/conf/bk_server.conf
	// }
	try { 
		//  没有参数的情况下退出
		if (args.length < 1) {
			usage();
			System.exit(-1);
		}

        // 解析出第一个参数:6
		int numBookies = 0;
		try {
			numBookies = Integer.parseInt(args[0]);
		} catch (NumberFormatException nfe) {
			LOG.error("Unrecognized number-of-bookies: {}", args[0]);
			usage();
			System.exit(-1);
		}
        
		// ********************************************************************
		// 配置文件转化成业务模型:ServerConfiguration
		// ********************************************************************
		ServerConfiguration conf = new ServerConfiguration();
		conf.setAllowLoopback(true);
		if (args.length >= 2) {
			// /Users/lixin/GitRepository/bookkeeper/conf/bk_server.conf
			String confFile = args[1];
			try {
				// ********************************************************************
				// 解析出配置文件到业务模型:ServerConfiguration
				// ********************************************************************
				conf.loadConf(new File(confFile).toURI().toURL());
				LOG.info("Using configuration file {}", confFile);
			} catch (Exception e) {
				// load conf failed
				LOG.warn("Error loading configuration file {}", confFile, e);
			}
		}
		
		// 第3个参数可以配置zk数据目录
		String zkDataDir = null;
		if (args.length >= 3) {
			zkDataDir = args[2];
		}
		
		// 默认的:BookiesConfig目录
		// defaultLocalBookiesConfigDir = /tmp/localbookies-config
		String localBookiesConfigDirName = defaultLocalBookiesConfigDir;
		if (args.length >= 4) {
			localBookiesConfigDirName = args[3];
		}

		// ******************************************************************************
		// 在单个jvm进程里启动6个Bookies
		// ******************************************************************************
		startLocalBookiesInternal(conf, zooKeeperDefaultHost, zooKeeperDefaultPort, numBookies, true,
				bookieDefaultInitialPort, false, "test", zkDataDir, localBookiesConfigDirName);
	} catch (Exception e) {
		LOG.error("Exiting LocalBookKeeper because of exception in main method", e);
		System.exit(-1);
	}
}
```
### (4). LocalBookKeeper.startLocalBookiesInternal
```
static void startLocalBookiesInternal(
			//
			ServerConfiguration conf,
			// 127.0.0.1
			String zkHost,
			// 2181
			int zkPort,
			// 6
			int numBookies,
			// true
			boolean shouldStartZK,
			// 5000
			int initialBookiePort,
			// false
			boolean stopOnExit,
			// test
			String dirSuffix,
			// null
			String zkDataDir,
			// /tmp/localbookies-config
			String localBookiesConfigDirName)
            throws Exception {
	
	conf.setMetadataServiceUri(
			// 设置元数据保存的URL地址
			// zk+null://127.0.0.1:2181/ledgers
			newMetadataServiceUri(
					zkHost,
					zkPort,
					conf.getLedgerManagerLayoutStringFromFactoryClass(),
					conf.getZkLedgersRootPath()));
	
	// ****************************************************************************
	// 创建:LocalBookKeeper
	// ****************************************************************************
	LocalBookKeeper lb = new LocalBookKeeper(numBookies, initialBookiePort, conf, localBookiesConfigDirName);
	ZooKeeperServerShim zks = null;
	File zkTmpDir = null;
	List<File> bkTmpDirs = null;
	try {
		if (shouldStartZK) { // true
			File zkDataDirFile = null;
			// zk数据目录
			if (zkDataDir != null) { // false
				zkDataDirFile = new File(zkDataDir);
				if (zkDataDirFile.exists() && zkDataDirFile.isFile()) {
					throw new IOException("Unable to create zkDataDir, since there is a file at "
							+ zkDataDirFile.getAbsolutePath());
				}
				if (!zkDataDirFile.exists() && !zkDataDirFile.mkdirs()) {
					throw new IOException("Unable to create zkDataDir - " + zkDataDirFile.getAbsolutePath());
				}
			}
			
			// *****************************************************************************
			// 创建ZK数据的临时目录(/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/zookeeper1221935550339855498test)
			// *****************************************************************************
			zkTmpDir = IOUtils.createTempDir("zookeeper", dirSuffix, zkDataDirFile);
			// 如果文件目录存在,则删除.
			zkTmpDir.deleteOnExit();
			
			// ********************************************************************
			// 值得学习:内部实际是构建出来了一个zookeeper.
			// ********************************************************************
			zks = LocalBookKeeper.runZookeeper(1000, zkPort, zkTmpDir);
		}
		
		// 此时ZK数据目录内容如下:
		// lixin-macbook:zookeeper1221935550339855498test lixin$ tree /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/zookeeper1221935550339855498test
		// /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/zookeeper1221935550339855498test
		// └── version-2
		//    ├── log.1
		//    └── snapshot.0
		
		// 调用:LocalBookKeeper.initializeZookeeper方法,对ZK进行Node的初始化(会在ZK中创建以下三个Node)
		// /ledgers
		// /ledgers/available/
		// /ledgers/available/readonly
		lb.initializeZookeeper(zkHost, zkPort);
		
		// ***************************************************************
		// 开始运行:Bookies
		// ***************************************************************
		bkTmpDirs = lb.runBookies(dirSuffix);

		try {
			while (true) {
				Thread.sleep(5000);
			}
		} catch (InterruptedException ie) {
			Thread.currentThread().interrupt();
			if (stopOnExit) {
				lb.shutdownBookies();

				if (null != zks) {
					zks.stop();
				}
			}
			throw ie;
		}
	} catch (Exception e) {
		LOG.error("Failed to run {} bookies : zk ensemble = '{}:{}'",
				numBookies, zkHost, zkPort, e);
		throw e;
	} finally {
		if (stopOnExit) {
			if (null != bkTmpDirs) {
				cleanupDirectories(bkTmpDirs);
			}
			if (null != zkTmpDir) {
				FileUtils.deleteDirectory(zkTmpDir);
			}
		}
	} // end finally
}
```
### (5). LocalBookKeeper.runBookies
```
private List<File> runBookies(String dirSuffix)
            throws Exception {
	List<File> tempDirs = new ArrayList<>();
	try {
		// dirSuffix = "test"
		// **********************************************************
		// 委托给了另一个重载的方法:runBookies
		// **********************************************************
		runBookies(tempDirs, dirSuffix);
		return tempDirs;
	} catch (Exception ioe) {
		cleanupDirectories(tempDirs);
		throw ioe;
	}
} // end runBookies
```
### (6). LocalBookKeeper.runBookies
+ 创建Bookies(BookieService),并启动HTTP监听
+ 底层用的是Vertx

```
private void runBookies(List<File> tempDirs, String dirSuffix)
            throws Exception {
	LOG.info("Starting Bookie(s)");
	// Create Bookie Servers (B1, B2, B3)
	
	// 定义存放WAL日志的数组
	journalDirs = new File[numberOfBookies];
	// 定义BookieServer数组
	bs = new BookieServer[numberOfBookies];
	// 定义BookieServer相应的配置(ServerConfiguration)数组
	bsConfs = new ServerConfiguration[numberOfBookies];
	
	// localBookiesConfigDir = /tmp/localbookies-config,存放的是每个:BookieServer对应的配置文件.
	// 验证目录(/tmp/localbookies-config)
	if (localBookiesConfigDir.exists() && localBookiesConfigDir.isFile()) {
		throw new IOException("Unable to create LocalBookiesConfigDir, since there is a file at "
				+ localBookiesConfigDir.getAbsolutePath());
	}
	
	// 创建目录(/tmp/localbookies-config)
	if (!localBookiesConfigDir.exists() && !localBookiesConfigDir.mkdirs()) {
		throw new IOException(
				"Unable to create LocalBookiesConfigDir - " + localBookiesConfigDir.getAbsolutePath());
	}

	
	for (int i = 0; i < numberOfBookies; i++) {
		
		// 如果没有指定WAL目录,则创建一个临时的
		if (null == baseConf.getJournalDirNameWithoutDefault()) { // true
			//  /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/localbookkeeper03300424156333625743test
			journalDirs[i] = IOUtils.createTempDir("localbookkeeper" + Integer.toString(i), dirSuffix);
			tempDirs.add(journalDirs[i]);
		} else {
			journalDirs[i] = new File(baseConf.getJournalDirName(), "bookie" + Integer.toString(i));
		}
		
		// 检查下新创建的:WAL目录是否存在
		if (journalDirs[i].exists()) { // true
			// 判断是否为目录
			if (journalDirs[i].isDirectory()) { // true
				// 重新删除目录
				FileUtils.deleteDirectory(journalDirs[i]);
			} else if (!journalDirs[i].delete()) {
				throw new IOException("Couldn't cleanup bookie journal dir " + journalDirs[i]);
			}
		}
		
		// 重新创建目录,如果创建不成功的话,就抛出异常,这样做的目的是要代码当前进程对磁盘有读写的权限.
		if (!journalDirs[i].mkdirs()) {
			throw new IOException("Couldn't create bookie journal dir " + journalDirs[i]);
		}

		// 数据目录
		// ledgerDirs = /tmp/bk-data
		String [] ledgerDirs = baseConf.getLedgerDirWithoutDefault();
		
		if ((null == ledgerDirs) || (0 == ledgerDirs.length)) { // false
			ledgerDirs = new String[] { journalDirs[i].getPath() };
		} else {
			// 循环遍历,创建目录
			for (int l = 0; l < ledgerDirs.length; l++) {
				// /tmp/bk-data/bookie0
				// /tmp/bk-data/bookie1
				// /tmp/bk-data/bookie2
				// /tmp/bk-data/bookie3
				// /tmp/bk-data/bookie4
				// /tmp/bk-data/bookie5
				File dir = new File(ledgerDirs[l], "bookie" + Integer.toString(i));
				
				if (dir.exists()) {
					if (dir.isDirectory()) {
						FileUtils.deleteDirectory(dir);
					} else if (!dir.delete()) {
						throw new IOException("Couldn't cleanup bookie ledger dir " + dir);
					}
				}
				if (!dir.mkdirs()) {
					throw new IOException("Couldn't create bookie ledger dir " + dir);
				}
				ledgerDirs[l] = dir.getPath();
			}
		}

		// clone一份配置文件.
		bsConfs[i] = new ServerConfiguration((ServerConfiguration) baseConf.clone());

		// initialPort = 5000
		PortManager.initPort(initialPort);
		if (0 == initialPort) { // false
			bsConfs[i].setBookiePort(0);
		} else { //true
			// 通过nextFreePort产生一个随机可用的端口,并设置到配置对象(ServerConfiguration)里.
			bsConfs[i].setBookiePort(PortManager.nextFreePort());
		}
		
		
		if (null == baseConf.getMetadataServiceUriUnchecked()) { // false
			bsConfs[i].setMetadataServiceUri(baseConf.getMetadataServiceUri());
		}

		// journalDirs[i].getPath() = /var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/localbookkeeper03300424156333625743test
		bsConfs[i].setJournalDirName(journalDirs[i].getPath());
		// ledgerDirs = /tmp/bk-data/bookieN
		bsConfs[i].setLedgerDirNames(ledgerDirs);
		
		// write config into file before start so we can know what's wrong if start failed
		// fileName = 172.17.14.140:5000.conf
		String fileName = BookieImpl.getBookieId(bsConfs[i]).toString() + ".conf";
		// 把配置内容序列化到本地磁盘上(/tmp/localbookies-config/172.17.14.140:5000.conf)
		serializeLocalBookieConfig(bsConfs[i], fileName);
		
		// 典型的发布与订阅模型
		// Mimic BookKeeper Main
		final ComponentInfoPublisher componentInfoPublisher = new ComponentInfoPublisher();
		final Supplier<BookieServiceInfo> bookieServiceInfoProvider = () -> buildBookieServiceInfo(componentInfoPublisher);
		
		// ******************************************************************************
		// 看了半天,好像这才是重点,创建:BookieService对象
		// ******************************************************************************
		BookieService bookieService = new BookieService(new BookieConfiguration(bsConfs[i]),
				NullStatsLogger.INSTANCE,
				bookieServiceInfoProvider
		);
		
		bs[i] = bookieService.getServer();
		// 发布事件
		bookieService.publishInfo(componentInfoPublisher);
		componentInfoPublisher.startupFinished();
		// 启动:BookieService
		bookieService.start();
	}

	// 创建:/tmp/localbookies-config/baseconf.conf,存放基础配置
	// /tmp/localbookies-config
	// lixin-macbook:localbookies-config lixin$ cat baseconf.conf
	// allowLoopback=true
	// bookiePort=3181
	// extraServerComponents=
	// httpServerEnabled=false
	// httpServerPort=8080
	// httpServerClass=org.apache.bookkeeper.http.vertx.VertxHttpServer
	// journalDirectories=/tmp/bk-txn
	// ledgerDirectories=/tmp/bk-data
	// zkServers=localhost:2181
	// zkTimeout=10000
	// zkEnableSecurity=false
	// storageserver.grpc.port=4181
	// dlog.bkcEnsembleSize=3
	// dlog.bkcWriteQuorumSize=2
	// dlog.bkcAckQuorumSize=2
	// storage.range.store.dirs=data/bookkeeper/ranges
	// storage.serve.readonly.tables=false
	// storage.cluster.controller.schedule.interval.ms=30000
	// metadataServiceUri=zk+null://127.0.0.1:2181/ledgers
	ServerConfiguration baseConfWithCorrectZKServers = new ServerConfiguration(
			(ServerConfiguration) baseConf.clone());
	if (null == baseConf.getMetadataServiceUriUnchecked()) {
		baseConfWithCorrectZKServers.setMetadataServiceUri(baseConf.getMetadataServiceUri());
	}
	serializeLocalBookieConfig(baseConfWithCorrectZKServers, "baseconf.conf");
}
```
### (7). 总结
本来是想找到调用Bookeeper操作日志的API来着的,结果,看完源码后,才发现:LocalBookKeeper的目的是构建N个Bookies(BookieService)容器,并启动.底层用到了:VertxHttpServer.    