---
layout: post
title: 'Apache Hadoop伪集群'
date: 2018-03-24
author: 李新
tags: Hadoop
---

### (1). 配置免密钥
```
# 生成密钥
ssh-keygen -t rsa -P ""

# 将公钥添加到授权的KEY中
cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys

```

!["Mac 启用远程登录"](/assets/hadoop/imgs/mac-ssh-remote-login-setting.jpg)


```
# 测试是否免密钥
lixin-macbook:.ssh lixin$ ssh lixin-macbook.local
   Last login: Tue Nov 17 15:56:33 2018 from 172.17.13.175
```

### (2). Apache Hadoop下载地址

> http://archive.apache.org/dist/hadoop/   
> http://archive.apache.org/dist/hadoop/core/hadoop-2.7.5/  

### (3). 以Hadoop2.7.5为例

> 我下载后解压路径如下:  
> <font color='red'>/Users/lixin/Developer/hadoop</font>  

### (4). core-site.xml
```
<configuration>
    <!-- 指定集群的文件系统类型:分布式文件系统 -->
    <property>
        <name>fs.default.name</name>
        <value>hdfs://lixin-macbook.local:9000</value>
    </property>

    <!-- 指定临时文件存储目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/Users/lixin/Developer/hadoop/hadoopDatas/tmpDatas</value>
    </property>

    <!-- 缓冲区大小,实际工作中根据服务器性能动态调整 -->
    <property>
        <name>io.file.buffer.size</name>
        <value>4096</value>
    </property>

   <!-- 开启HDFS的垃圾桶机制,删除掉的数据可以从垃圾桶中回收,单位分钟 -->
    <property>
        <name>fs.trash.interval</name>
        <value>10080</value>
    </property>
</configuration>
```

### (5). hdfs-site.xml

```
<configuration>
    <!-- 辅助namenode的secondary的访问地址和端口 -->
    <property>
       <name>dfs.namenode.secondary.http-address</name>
       <value>lixin-macbook.local:50090</value>
    </property>

    <!-- 指定namenode的访问地址和端口  -->
    <property>
       <name>dfs.namenode.http-address</name>
       <value>lixin-macbook.local:50070</value>
    </property>
    
    <!-- namenode存储元数据的路径 -->
    <property>
       <name>dfs.namenode.name.dir</name>
       <value>file:///Users/lixin/Developer/hadoop/hadoopDatas/nameNodeDatas,file:///Users/lixin/Developer/hadoop/hadoopDatas/nameNodeDatas2</value>
    </property>

    <!-- dataNode数据的存储位置,多个目录用逗号分隔 -->
    <property>
       <name>dfs.datanode.data.dir</name>
       <value>file:///Users/lixin/Developer/hadoop/hadoopDatas/dataNodeDatas,file:///Users/lixin/Developer/hadoop/hadoopDatas/dataNodeDatas2</value>
    </property>

    <!-- namenode日志文件存放目录 -->
    <property>
       <name>dfs.namenode.edits.dir</name>
       <value>file:///Users/lixin/Developer/hadoop/hadoopDatas/nn/edits</value>
    </property>

    <!-- namenode -->
    <property>
       <name>dfs.namenode.checkpoint.dir</name>
       <value>file:///Users/lixin/Developer/hadoop/hadoopDatas/snn/name</value>
    </property>

    <!--  -->
    <property>
       <name>dfs.namenode.checkpoint.edits.dir</name>
       <value>file:///Users/lixin/Developer/hadoop/hadoopDatas/dfs/snn/edits</value>
    </property>

    <!-- 每个文件存储的副本数量,一个文件在HDFS中存储有3个副本 -->
    <property>
       <name>dfs.replication</name>
       <value>1</value>
    </property>

    <!-- 设置HDFS的文件权限,暂时关闭文件权限 -->
    <property>
       <name>dfs.permissions.enabled</name>
       <value>false</value>
    </property>
    
    <!-- 对一个大的文件进行切片,设置切片时的大小:128M -->
    <property>
       <name>dfs.blocksize</name>
       <value>134217728</value>
    </property>
</configuration>
```

### (6). hadoop-env.sh(配置JDK)
>  配置JAVA_HOME    

```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
```

### (7). mapred-site.xml

```
<configuration>

    <!-- 开启MapReduce小任务模式 -->
    <property>
        <name>mapreduce.job.ubertask.enable</name>
        <value>true</value>
    </property>

    <!-- 历史任务的主机和端口 -->
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>lixin-macbook.local:10020</value>
    </property>

    <!-- 网页来访问历史任务的主机和端口 -->
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>lixin-macbook.local:19888</value>
    </property>
</configuration>
```
### (8). yarn-site.xml
```
<configuration>

    <!-- 配置Yarn主节点的位置 --> 
    <property>
       <name>yarn.resourcemanager.hostname</name>
       <value>lixin-macbook.local</value>
    </property>

    <!-- --> 
    <property>
       <name>yarn.nodemanager.aux-services</name>
       <value>mapreduce_shuffle</value>
    </property>

    <!-- 开启日志聚合功能 -->
    <property>
       <name>yarn.log-aggregation-enable</name>
       <value>true</value>
    </property>

    <!-- 设置聚合日志在HDFS保存的时间,单位为:秒 -->
    <property>
       <name>yarn.log-aggregation.retain-seconds</name>
       <value>604800</value>
    </property>
    
    <!-- 设置Yarn集群的内存分配方案 -->
    <property>
       <name>yarn.nodemanager.resource.memory-mb</name>
       <value>20480</value>
    </property>
    <property>
       <name>yarn.scheduler.minimum-allocation-mb</name>
       <value>2048</value>
    </property>
    <property>
       <name>yarn.nodemanager.vmem-pmem-ratio</name>
       <value>2.1</value>
    </property>
</configuration>
```
### (9). mapred-env.sh

>  配置JAVA_HOME    

```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home

```

### (10). slaves
> 配置Slave机器(**如果是真实集群,配置多个机器名称**)   

```
localhost
```

### (11). 创建Hadoop数据目录

```
mkdir -p /Users/lixin/Developer/hadoop/hadoopDatas/tmpDatas
mkdir -p /Users/lixin/Developer/hadoop/hadoopDatas/nameNodeDatas
mkdir -p /Users/lixin/Developer/hadoop/hadoopDatas/nameNodeDatas2
mkdir -p /Users/lixin/Developer/hadoop/hadoopDatas/dataNodeDatas
mkdir -p /Users/lixin/Developer/hadoop/hadoopDatas/dataNodeDatas2
mkdir -p /Users/lixin/Developer/hadoop/hadoopDatas/nn/edits
mkdir -p /Users/lixin/Developer/hadoop/hadoopDatas/snn/name
mkdir -p /Users/lixin/Developer/hadoop/hadoopDatas/dfs/snn/edits
```

> 以树型方式查看数据目录

```
lixin-macbook:hadoop lixin$ pwd
   /Users/lixin/Developer/hadoop
lixin-macbook:hadoop lixin$ tree hadoopDatas/
hadoopDatas/
├── dataNodeDatas
├── dataNodeDatas2
├── dfs
│   └── snn
│       └── edits
├── nameNodeDatas
├── nameNodeDatas2
├── nn
│   └── edits
├── snn
│   └── name
└── tmpDatas
```

### (12). 配置Hadoop环境变量

```
# 修改环境变量
vi ~/.bash_profile     
   export HADOOP_HOME=/Users/lixin/Developer/hadoop    
   export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin   

# 环境变量生效
source ~/.bash_profile    
```
### (13). 启动集群
> 启动时,如果要输入密码,请把密钥权限修改为:600(chmod 600 ~/.ssh/id_rsa*)  

```
# 工作目录 
lixin-macbook:hadoop lixin$ pwd
   /Users/lixin/Developer/hadoop

# 格式化namenode
lixin-macbook:hadoop lixin$ ./bin/hdfs namenode -format

# 启动HDFS(NameNode/DataNode/SecondaryNameNode)
lixin-macbook:hadoop lixin$ ./sbin/start-dfs.sh

# 启动Yarn(ResourceManager/NodeManager)
./sbin/start-yarn.sh 

# 启动jobhistory(JobHistoryServer)
./sbin/mr-jobhistory-daemon.sh start historyserver

```
### (14). 查看HDFS
> http://lixin-macbook.local:50070/explorer.html#/     

!["查看HDFS界面"](/assets/hadoop/imgs/hadoop-hdfs-manager-ui.png )

> 查看HDFS文件目录   

!["查看HDFS文件目录"](/assets/hadoop/imgs/hadoop-hdfs-browse-directory.jpg)


### (15). 查看Yarn集群
>  http://lixin-macbook.local:8088/cluster 

!["查看Yarn界面"](/assets/hadoop/imgs/hadoop-yarn-manager-ui.jpg)

### (16). 查看历史任务
> http://lixin-macbook.local:19888/jobhistory   
!["查看历史任务"](/assets/hadoop/imgs/hadoop-jobhistory-manager-ui.png)

