---
layout: post
title: '了解Gossip协议(一)' 
date: 2022-11-07
author: 李新
tags:  Gossip
---


### (1). Gossip是什么
Gossip 协议(Gossip Protocol)又称Epidemic协议(Epidemic Protocol),是基于流行病传播方式的节点或者进程之间信息交换的协议,在分布式系统中被广泛使用,
Gossip Protocol最初是由施乐公司帕洛阿尔托研究中心(Palo Alto Research Center)的研究员艾伦·德默斯(Alan Demers)于1987年创造的.  
从Gossip单词就可以看到,其中文意思是八卦、流言等意思,我们可以想象下绯闻的传播(或者流行病的传播),Gossip协议的工作原理就类似于这个.Gossip协议利用一种随机的方式将信息传播到整个网络中,并在一定时间内使得系统内的所有节点数据一致.   
Gossip其实是一种去中心化思路的分布式协议,解决状态在集群中的传播和状态一致性的保证两个问题,使用Gossip协议的有:Redis Cluster、Consul、Apache Cassandra等. 
### (2). Gossip优势
+ 可扩展性（Scalable）
  - gossip协议是可扩展的,一般需要O(logN)轮就可以将信息传播到所有的节点,其中N代表节点的个数.每个节点仅发送固定数量的消息,并且与网络中节点数目无法.在数据传送的时候,节点并不会等待消息的ack,所以消息传送失败也没有关系,因为可以通过其他节点将消息传递给之前传送失败的节点.系统可以轻松扩展到数百万个进程.
+ 容错（Fault-tolerance）
  - 网络中任何节点的重启或者宕机都不会影响Gossip协议的运行.
+ 健壮性（Robust）
  - Gossip协议是去中心化的协议,所以集群中的所有节点都是对等的,没有特殊的节点,所以任何节点出现问题都不会阻止其他节点继续发送消息.任何节点都可以随时加入或离开,而不会影响系统的整体服务质量(QOS)
+ 最终一致性（Convergent consistency）
  - Gossip协议实现信息指数级的快速传播,因此在有新信息需要传播时,消息可以快速地发送到全局节点,在有限的时间内能够做到所有节点都拥有最新的数据. 

### (3). Gossip协议的类型
前面说了节点会将信息传播到整个网络中,那么节点在什么情况下发起信息交换?这就涉及到Gossip协议的类型.目前主要有两种方法: 
+ Anti-Entropy(反熵): 每个节点周期性地随机选择其他节点，然后通过互相交换自己的所有数据来消除两者之间的差异.Anti-Entropy 这种方法非常可靠,但是每次节点两两交换自己的所有数据会带来非常大的通信负担,以此不会频繁使用.Anti-Entropy 使用“simple epidemics”的方式，所以其包含两种状态：susceptible 和 infective，这种模型也称为 SI model。处于 infective 状态的节点代表其有数据更新，并且会将这个数据分享给其他节点；处于 susceptible 状态的节点代表其并没有收到来自其他节点的更新. 
+ Rumor-Mongerin(谣言传播): 当一个节点有了新的信息后,这个节点变成活跃状态,并周期性地联系其他节点向其发送新信息.直到所有的节点都知道该新信息.因为节点之间只是交换新信息,所有大大减少了通信的负担.Rumor-Mongering 使用“complex epidemics”方法,相比Anti-Entropy多了一种状态:removed，这种模型也称为 SIR model.处于removed状态的节点说明其已经接收到来自其他节点的更新,但是其并不会将这个更新分享给其他节点.因为Rumor消息会在某个时间标记为removed,然后不会发送给其他节点,所以Rumor-Mongering类型的Gossip协议有极小概率使得更新不会达到所有节点.一般来说:为了在通信代价和可靠性之间取得折中,需要将这两种方法结合使用. 

### (4). Gossip协议的通讯方式
不管是Anti-Entropy还是Rumor-Mongering都涉及到节点间的数据交互方式,节点间的交互方式主要有三种：Push、Pull 以及 Push&Pull.
+ Push: 发起信息交换的节点 A 随机选择联系节点 B,并向其发送自己的信息,节点 B 在收到信息后更新比自己新的数据,一般拥有新信息的节点才会作为发起节点.   
+ Pull: 发起信息交换的节点 A 随机选择联系节点 B,并从对方获取信息.一般无新信息的节点才会作为发起节点.   
+ Push&Pull: 发起信息交换的节点 A 向选择的节点 B 发送信息,同时从对方获取数据，用于更新自己的本地数据.   

### (5). Gossip源码索引目录
+ ["Gossip源码剖析之GossipManager(二)"](/2022/11/07/Gossip-GossipManager.html)   
+ ["Gossip源码剖析之RandomActiveGossipThread(三)"](/2022/11/07/Gossip-RandomActiveGossipThread.html)  