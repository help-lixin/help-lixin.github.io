---
layout: post
title: 'Redis 主从复制原理'
date: 2021-07-05
author: 李新
tags:  Redis 
---
### (1). 主从复制流程
```
1) slave执行slaveof <masterIP> <masterPort>,并保master节点信息.          
2) slave建立与master的socket连接,并周期性的ping,master节点返回:pong.          
3) slave发送指令: psync <runId> [offset].             
4) master接受到psync指令建立"缓冲区"(repl_backlog_size),并触发bgsave生成RDB文件,通过sokcet发送给slave.           
5) slave接受RDB,清空数据,执行RDB文件恢复过程.          
6) 发送命令告诉master,RDB恢复已经完成.           
7) master发送"缓冲区"信息给slave.           
8) slave接收信息,执行bgrewriteaof,恢复数据.              
```
### (2). psyn命令与积压缓冲区
```
psync [runId]  [offset]

runId:    每个redis节点启动都会生成唯一的runId,每次redis重启后,runId也会发生变化.
offset:   主节点和从节点都各自维护自己的主从复制偏移量offset,当主节点有写入命令时,offset=offset+命令的字节长度.从节点在收到主节点发送的命令后,也会增加自己的offset,并把自己的offset发送给主节点.这样,主节点同时保存自己的offset,从节点的offset,通过对比offset来判断主从节点数据是否一致


积压缓存区参数详解:
repl_backlog_size: 保存在主节点上的一个固定长度的先进先出队列,默认大小为1MB
1) master发送数据给slave过程中,master还会进行一些写操作,这时候的数据存储"缓冲区".从节点同步主节点数据完成后,主节点将缓冲区的数据继续发送给从节点,用于部分复制. 
2）主节点(master)响应写命令时,不但会把命名发送给从节点,还会写入复制积压缓冲区,用于复制命令丢失的数据补救.
3) 注意了: 这个缓冲区是FIFO的,如果缓冲区满了,最先进入的元素就会丢失.   
```
### (3). psyn全量复制
```
1）从节点发送psync ? -1命令,因为第一次发送,不知道主节点的runId,所以为:？,因为是第一次复制,所以offset = -1.   
2）主节点发现从节点是第一次复制,变返回FULLRESYNC {runId} {offset},runId是主节点的runId,offset是主节点目前的offset.  
3）从节点接收主节点信息后,保存.  
4）主节点在发送FULLRESYNC后,启动bgsave命令,生成RDB文件.  
5) 主节点发送RDB文件给从节点,同时,主节点会建立:缓冲区,在从节点全量复制的这段时间,所有的命令,都会在缓冲区.   
6）从节点清理自己的数据库数据.
7）从节点加载RDB文件,将数据保存的自己的数据库中.
8）从节点告之主节点,数据复制完成,开启部份复制功能.
```
### (4). psyn部份复制
```
1）部分复制主要是Redis针对全量复制的过高开销做出的一种优化措施,使用psync {runId}{offset}命令实现.当从节点(slave)正在复制主节点(master)时,如果出现网络闪断或者命令丢失等异常情况时,从节点会向主节点要求补发丢失的命令数据,如果主节点的复制积压缓冲区内存将这部分数据则直接发送给从节点,这样就可以保持主从节点复制的一致性.补发的这部分数据一般远远小于全量数据.  
2）主从连接中断期间主节点依然响应命令,但因复制连接中断命令无法发送给从节点,不过主节点内部存在的复制积压缓冲区,依然可以保存最近一段时间的写命令数据,默认最大缓存1MB.当从节点网络恢复后,从节点会再次连上主节点.  
3）当主从连接恢复后,由于从节点之前保存了自身已复制的偏移量和主节点的运行ID.因此会把它们当做psync参数发送个主节点,要求进行部分复制操作.
4）主节点接到psync命令后首先核对参数runId是否与自身一致,如果一致,说明之前复制的是当前主节点,之后根据参数offset在自身复制积压缓冲区查找,如果偏移量之后的数据存在缓冲区中,则对从节点发送+COUTINUE响应,表示可以进行部分复制.因为缓冲区大小固定,若发生缓存溢出,则要进行全量复制. 
5）主节点根据偏移量把复制积压缓冲区里的数据发送给从节点,保证主从复制进入正常状态.
```

### (5). psyn同步图解
!["psyn同步图解"](/assets/redis/imgs/redis-master-slave-sync.webp)

### (6). 注意事项
```
1) 积压缓冲区是FIFO,如果offset不在积压缓冲区内,是会造成全量复制的,所以,积压缓冲区的大小要配置适当.   
2) 在生产上,要尽量错开高峰期做主从同步(全量)的操作.    
```