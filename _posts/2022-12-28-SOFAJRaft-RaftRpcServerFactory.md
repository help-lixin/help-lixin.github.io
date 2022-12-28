---
layout: post
title: 'SOFAJRaft源码之RaftRpcServerFactory(一)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
最近一直在看RAFT相关的源码,Atomix突然开始转型(用Go重写),不得不看其它RAFT的实现,从蚂蚁金服官网clone RAFT演示代码,进行源码剖析,在这里先看RPC初始化处理.

### (2). CounterServer
```
// serverId.getEndpoint() == 127.0.0.1:8081
RpcServer rpcServer = RaftRpcServerFactory.createRaftRpcServer(serverId.getEndpoint());
```

### (3). RaftRpcServerFactory静态代码块
> 我们知道,初始化一个类,首先会初始化静态代码块. 


```
public class RaftRpcServerFactory {
    static {
		// PROTOBUF处理,在这里,不进行深究
        ProtobufMsgFactory.load();
    }
}	
```
### (4). RaftRpcServerFactory.createRaftRpcServer
```
public static RpcServer createRaftRpcServer(final Endpoint endpoint) {
	return createRaftRpcServer(endpoint, null, null);
}
```
### (5). RaftRpcServerFactory.createRaftRpcServer
```
public static RpcServer createRaftRpcServer(final Endpoint endpoint, final Executor raftExecutor,
                                                final Executor cliExecutor) {
	// 委托给:RpcFactoryHelper处理
	// RpcFactoryHelper内部通过SPI加载:RaftRpcFactory,然后,创建:RpcServer
	final RpcServer rpcServer = RpcFactoryHelper.rpcFactory().createRpcServer(endpoint);
	
	// ****************************************************************
	// 为RpcServer配置RAFT处理,RpcServer是接受网络请求,最终要承载业务处理.
	// ****************************************************************
	addRaftRequestProcessors(rpcServer, raftExecutor, cliExecutor);
	return rpcServer;
}
```
### (6). RpcFactoryHelper.rpcFactory
```
public class RpcFactoryHelper {
	// ***************************************************************************************
	// 通过SPI加载RaftRpcFactory的实现类(com.alipay.sofa.jraft.rpc.impl.GrpcRaftRpcFactory).
	// ***************************************************************************************
    private static final RaftRpcFactory RPC_FACTORY = JRaftServiceLoader.load(RaftRpcFactory.class) //
                                                        .first();
    public static RaftRpcFactory rpcFactory() {
        return RPC_FACTORY;
    } // end 
}	
```
### (7). RaftRpcServerFactory.addRaftRequestProcessors
> 为RpcServer配置业务处理.

```
public static void addRaftRequestProcessors(final RpcServer rpcServer, final Executor raftExecutor,
                                                final Executor cliExecutor) {
	// raft core processors
	final AppendEntriesRequestProcessor appendEntriesRequestProcessor = new AppendEntriesRequestProcessor(raftExecutor);
	rpcServer.registerConnectionClosedEventListener(appendEntriesRequestProcessor);
	rpcServer.registerProcessor(appendEntriesRequestProcessor);
	rpcServer.registerProcessor(new GetFileRequestProcessor(raftExecutor));
	rpcServer.registerProcessor(new InstallSnapshotRequestProcessor(raftExecutor));
	rpcServer.registerProcessor(new RequestVoteRequestProcessor(raftExecutor));
	rpcServer.registerProcessor(new PingRequestProcessor());
	rpcServer.registerProcessor(new TimeoutNowRequestProcessor(raftExecutor));
	rpcServer.registerProcessor(new ReadIndexRequestProcessor(raftExecutor));
	
	// raft cli service
	rpcServer.registerProcessor(new AddPeerRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new RemovePeerRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new ResetPeerRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new ChangePeersRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new GetLeaderRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new SnapshotRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new TransferLeaderRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new GetPeersRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new AddLearnersRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new RemoveLearnersRequestProcessor(cliExecutor));
	rpcServer.registerProcessor(new ResetLearnersRequestProcessor(cliExecutor));
} 
```
### (8). RaftRpcFactory接口
```
public interface RaftRpcFactory {
  void registerProtobufSerializer(final String className, final Object... args);
  
  RpcClient createRpcClient(final ConfigHelper<RpcClient> helper);
  
  RpcServer createRpcServer(final Endpoint endpoint, final ConfigHelper<RpcServer> helper);
  
  RpcResponseFactory getRpcResponseFactory();
}
```
### (9). RaftRpcServerFactory类图

!["RaftRpcServerFactory"](/assets/jraft/imgs/RaftRpcServerFactory-ClassDiagram.jpg)

### (10). 总结
> RaftRpcServerFactory的主要职责是创建:RpcServer,并为RpcServer配置业务处理(看到了吧!典型的工厂模式).  