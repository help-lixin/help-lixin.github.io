---
layout: post
title: 'SOFAJRaft源码之ClientService初始化(十八)' 
date: 2022-12-28
author: 李新
tags:  SOFAJRaft
---

### (1). 概述
在准备剖析Replicator时,发现它会与网络通信有一些纠缠不清,因为与网络相关,想要剖析整个链路,不想再进行Mock了,所以,这一小篇主要剖析与网络相关的类:ClientService

### (2). ClientService UML图

!["ClientService UML图"](/assets/jraft/imgs/ClientService-ClassDiagram.jpg)

### (3). ClientService接口
> 为什么要看ClientService的接口,因为:接口代表着实现类的大体能力. 

```
public interface ClientService extends Lifecycle<RpcOptions> {

    boolean connect(final Endpoint endpoint);

    boolean checkConnection(final Endpoint endpoint, final boolean createIfAbsent);

    boolean disconnect(final Endpoint endpoint);

    boolean isConnected(final Endpoint endpoint);

    <T extends Message> Future<Message> invokeWithDone(final Endpoint endpoint, final Message request,
                                                       final RpcResponseClosure<T> done, final int timeoutMs);
}
```
### (4). RaftClientService接口
> RaftClientService的接口签名上能看出来,它是与RAFT协议相关的操作API封装. 

```
public interface RaftClientService extends ClientService {
    Future<Message> preVote(final Endpoint endpoint, final RpcRequests.RequestVoteRequest request,
                            final RpcResponseClosure<RpcRequests.RequestVoteResponse> done);

    Future<Message> requestVote(final Endpoint endpoint, final RpcRequests.RequestVoteRequest request,
                                final RpcResponseClosure<RpcRequests.RequestVoteResponse> done);

    Future<Message> appendEntries(final Endpoint endpoint, final RpcRequests.AppendEntriesRequest request,
                                  final int timeoutMs, final RpcResponseClosure<RpcRequests.AppendEntriesResponse> done);

    Future<Message> installSnapshot(final Endpoint endpoint, final RpcRequests.InstallSnapshotRequest request,
                                    final RpcResponseClosure<RpcRequests.InstallSnapshotResponse> done);

    Future<Message> getFile(final Endpoint endpoint, final RpcRequests.GetFileRequest request, final int timeoutMs,
                            final RpcResponseClosure<RpcRequests.GetFileResponse> done);

    Future<Message> timeoutNow(final Endpoint endpoint, final RpcRequests.TimeoutNowRequest request,
                               final int timeoutMs, final RpcResponseClosure<RpcRequests.TimeoutNowResponse> done);

    Future<Message> readIndex(final Endpoint endpoint, final RpcRequests.ReadIndexRequest request, final int timeoutMs,
                              final RpcResponseClosure<RpcRequests.ReadIndexResponse> done);
}
```
### (5). 总结
我的习惯是,如果接口能力很清晰了,就懒得去看实现了,实现类在心里大概有数了,最终无非不过是通过Netty通信,不想往下剖析的另一个原因是:因为,JRAFT用到了SOFABolt,怕呆会被带到:SOFABolt里去了. 