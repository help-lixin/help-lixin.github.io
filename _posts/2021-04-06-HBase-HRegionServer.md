---
layout: post
title: 'HBase HRegionServer(二)'
date: 2021-04-06
author: 李新
tags:  HBase源码
---

### (1). 概述
> 我只关心WAL的复制功能,所以,只挑部份源码查看.

### (2). HRegionServer.run
```

public void run() {
	// ... ...
	// 1. 判断是否集群模式
	while (keepLooping()) {
		
		// 2. 向HMaster发送请求,返回MAP内容如下:
		// name: "hbase.rootdir"  value: "hdfs://lixin-macbook.local:9000/hbase"
		// name: "fs.defaultFS" value: "hdfs://lixin-macbook.local:9000"
		// name: "hbase.master.info.port" value: "16010"
		// name: "hbase.regionserver.hostname.seen.by.master" value: "localhost"
		RegionServerStartupResponse w = reportForDuty();
		if (w == null) {
		  LOG.warn("reportForDuty failed; sleeping and then retrying.");
		  this.sleeper.sleep();
		} else {
		  // 3. 处理结果
		  handleReportForDutyResponse(w);
		  break;
		}
	}
	// ... ...
}
```
### (3). HRegionServer.handleReportForDutyResponse
```
protected void handleReportForDutyResponse(final RegionServerStartupResponse c) throws IOException {
	// ... ...
	
	// 设置WAL和复制功能
	this.walFactory = setupWALAndReplication();
	
	// ... ...
}
```
### (4). HRegionServer.setupWALAndReplication
```
private WALFactory setupWALAndReplication() throws IOException {
	// 创建复制实例
	createNewReplicationInstance(conf, this, this.walFs, logDir, oldLogDir);
}
```
### (5). HRegionServer.createNewReplicationInstance
```
static private void createNewReplicationInstance(
		   // Configuration: core-default.xml, core-site.xml, mapred-default.xml, mapred-site.xml, yarn-default.xml, yarn-site.xml, hdfs-default.xml, hdfs-site.xml, hbase-default.xml, hbase-site.xml
           Configuration conf,
		   // 
		   HRegionServer server, 
		   // DFS[DFSClient[clientName=DFSClient_NONMAPREDUCE_1886280369_1, ugi=lixin (auth:SIMPLE)]]
		   FileSystem walFs, 
		   // hdfs://lixin-macbook.local:9000/hbase/WALs/localhost,16201,1617955056347
		   Path walDir, 
		   // hdfs://lixin-macbook.local:9000/hbase/oldWALs
		   Path oldWALDir) throws IOException{
	
	// ... 
	// 1. 读取配置:hbase.replication.source.service,如果没有配置,则返回默认值:org.apache.hadoop.hbase.replication.regionserver.Replication
	String sourceClassname = conf.get(HConstants.REPLICATION_SOURCE_SERVICE_CLASSNAME,
	                               HConstants.REPLICATION_SERVICE_CLASSNAME_DEFAULT);
	
	// 2. 读取配置:hbase.replication.sink.service,如果没有配置,则返回默认值:org.apache.hadoop.hbase.replication.regionserver.Replication
	String sinkClassname = conf.get(HConstants.REPLICATION_SINK_SERVICE_CLASSNAME,
	                             HConstants.REPLICATION_SERVICE_CLASSNAME_DEFAULT);

	// 如果:sourceClassname和sinkClassname相同,replicationSinkHandler=replicationSourceHandler
	if (sourceClassname.equals(sinkClassname)) { //true
	
	  // 给HRegionServer设置复制SourceHandler/SinkHandler
	  // HRegionServer.replicationSourceHandler
	  server.replicationSourceHandler = (ReplicationSourceService)
										 newReplicationInstance(sourceClassname,
										 conf, server, walFs, walDir, oldWALDir);
	  server.replicationSinkHandler = (ReplicationSinkService)
										 server.replicationSourceHandler;
	} else {
	  // 通过反射构造各自的Hnadler	
	  server.replicationSourceHandler = (ReplicationSourceService)
										 newReplicationInstance(sourceClassname,
										 conf, server, walFs, walDir, oldWALDir);
	  server.replicationSinkHandler = (ReplicationSinkService)
										 newReplicationInstance(sinkClassname,
										 conf, server, walFs, walDir, oldWALDir);
	} // end else
}
```
### (6). HRegionServer.newReplicationInstance
```
static private ReplicationService newReplicationInstance(
	//  org.apache.hadoop.hbase.replication.regionserver.Replication
	String classname,
    Configuration conf, 
	HRegionServer server, 
	FileSystem walFs, 
	Path walDir,
    Path oldLogDir) throws IOException{

	// 1. 通过反射加载:org.apache.hadoop.hbase.replication.regionserver.Replication
	Class<?> clazz = null;
	try {
	  ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
	  clazz = Class.forName(classname, true, classLoader);
	} catch (java.lang.ClassNotFoundException nfe) {
	  throw new IOException("Could not find class for " + classname);
	}
	
	// 2. 调用:Replication类的无参构造器,返回:ReplicationService实例
	// create an instance of the replication object.
	ReplicationService service = (ReplicationService)
							  ReflectionUtils.newInstance(clazz, conf);

	// 3. ReplicationService.initialize.
	service.initialize(server, walFs, walDir, oldLogDir);
	return service;
}
```
### (7). 总结
> 1. HRegionServer启动时,会与HMaster进行交互,获得:WAL所在的目录.  
> 2. 构建:Replication实例,并初始化.  
> 3. Replication的内容,另开一篇来分析.  