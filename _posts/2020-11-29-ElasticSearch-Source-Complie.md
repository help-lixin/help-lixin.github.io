---
layout: post
title: 'ElasticSearch 7.1 源码编译并运行(二)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 下载ES源码
```
lixin-macbook:GitRepository lixin$ pwd
/Users/lixin/GitRepository
lixin-macbook:GitRepository lixin$ git clone -b v7.1.0 https://github.com/elastic/elasticsearch.git elasticsearch-v7.1.0
```

### (2). 编译前准备

> 1. 安装Docker(ES编译时,需要构建docker镜像)   
> 2. ES 7开始已经自带OpenJDK 12   

### (3). 编译并打包
```
# 我的工作目录
lixin-macbook:elasticsearch-v7.1.0 lixin$ pwd
/Users/lixin/GitRepository/elasticsearch-v7.1.0
# 打包
lixin-macbook:elasticsearch-v7.1.0 lixin$ ./gradlew  assemble
```
### (4). 打包结果
```
# 我的工作目录
lixin-macbook:elasticsearch-v7.1.0 lixin$ pwd
/Users/lixin/GitRepository/elasticsearch-v7.1.0

# 打包后结果在这里(distribution/archives/)
lixin-macbook:elasticsearch-v7.1.0 lixin$ tree  distribution/archives/

├── darwin-tar
│   ├── build
│   │   └── distributions
│   │       └── elasticsearch-7.1.0-SNAPSHOT-darwin-x86_64.tar.gz
├── linux-tar
│   ├── build
│   │   └── distributions
│   │       └── elasticsearch-7.1.0-SNAPSHOT-linux-x86_64.tar.gz
├── no-jdk-darwin-tar
│   ├── build
│   │   └── distributions
│   │       └── elasticsearch-7.1.0-SNAPSHOT-no-jdk-darwin-x86_64.tar.gz
│   └── build.gradle
├── no-jdk-linux-tar
│   ├── build
│   │   └── distributions
│   │       └── elasticsearch-7.1.0-SNAPSHOT-no-jdk-linux-x86_64.tar.gz
│   └── build.gradle
├── no-jdk-windows-zip
│   ├── build
│   │   ├── distributions
│   │   │   ├── elasticsearch-7.1.0-SNAPSHOT-no-jdk-windows-x86_64.zip├── oss-no-jdk-windows-zip
│   ├── build
│   │   ├── distributions
│   │   │   ├── elasticsearch-oss-7.1.0-SNAPSHOT-no-jdk-windows-x86_64.zip
├── oss-windows-zip
│   ├── build
│   │   ├── distributions
│   │   │   ├── elasticsearch-oss-7.1.0-SNAPSHOT-windows-x86_64.zip
└── windows-zip
    ├── build
    │   ├── distributions
    │   │   ├── elasticsearch-7.1.0-SNAPSHOT-windows-x86_64.zip


```
### (5). 解压并启动
```
lixin-macbook:bin lixin$ pwd
/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/bin

# 启动ES
lixin-macbook:bin lixin$ ./elasticsearch -d
```

```
[2020-12-02T13:18:54,338][INFO ][o.e.e.NodeEnvironment    ] [lixin-macbook.local] using [1] data paths, mounts [[/ (/dev/disk1s5)]], net usable_space [252.3gb], net total_space [465.7gb], types [apfs]
[2020-12-02T13:18:54,343][INFO ][o.e.e.NodeEnvironment    ] [lixin-macbook.local] heap size [990.7mb], compressed ordinary object pointers [true]
[2020-12-02T13:18:54,346][INFO ][o.e.n.Node               ] [lixin-macbook.local] node name [lixin-macbook.local], node ID [XTcpVqUhQfymSuzxG5AivA], cluster name [elasticsearch]
[2020-12-02T13:18:54,347][INFO ][o.e.n.Node               ] [lixin-macbook.local] version[7.1.0-SNAPSHOT], pid[1043], build[default/tar/606a173/2020-12-01T13:40:42.939742Z], OS[Mac OS X/10.15.7/x86_64], JVM[Oracle Corporation/Java HotSpot(TM) 64-Bit Server VM/12.0.2/12.0.2+10]
[2020-12-02T13:18:54,348][INFO ][o.e.n.Node               ] [lixin-macbook.local] JVM home [/Library/Java/JavaVirtualMachines/jdk-12.0.2.jdk/Contents/Home]

# JVM参数指定
[2020-12-02T13:18:54,349][INFO ][o.e.n.Node               ] [lixin-macbook.local] JVM arguments [-Xms1g, -Xmx1g, -XX:+UseConcMarkSweepGC, -XX:CMSInitiatingOccupancyFraction=75, -XX:+UseCMSInitiatingOccupancyOnly, -Des.networkaddress.cache.ttl=60, -Des.networkaddress.cache.negative.ttl=10, -XX:+AlwaysPreTouch, -Xss1m, -Djava.awt.headless=true, -Dfile.encoding=UTF-8, -Djna.nosys=true, -XX:-OmitStackTraceInFastThrow, -Dio.netty.noUnsafe=true, -Dio.netty.noKeySetOptimization=true, -Dio.netty.recycler.maxCapacityPerThread=0, -Dlog4j.shutdownHookEnabled=false, -Dlog4j2.disable.jmx=true, -Djava.io.tmpdir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/elasticsearch-5028289922499440991, -XX:+HeapDumpOnOutOfMemoryError, -XX:HeapDumpPath=data, -XX:ErrorFile=logs/hs_err_pid%p.log, -Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m, -Djava.locale.providers=COMPAT, -Dio.netty.allocator.type=unpooled, -Des.path.home=/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0, -Des.path.conf=/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/config, -Des.distribution.flavor=default, -Des.distribution.type=tar, -Des.bundled_jdk=true]

[2020-12-02T13:18:54,355][WARN ][o.e.n.Node               ] [lixin-macbook.local] version [7.1.0-SNAPSHOT] is a pre-release version of Elasticsearch and is not suitable for production
[2020-12-02T13:18:57,033][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [aggs-matrix-stats]
[2020-12-02T13:18:57,034][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [analysis-common]
[2020-12-02T13:18:57,037][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [ingest-common]
[2020-12-02T13:18:57,037][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [ingest-geoip]
[2020-12-02T13:18:57,037][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [ingest-user-agent]
[2020-12-02T13:18:57,038][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [lang-expression]
[2020-12-02T13:18:57,039][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [lang-mustache]
[2020-12-02T13:18:57,040][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [lang-painless]
[2020-12-02T13:18:57,040][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [mapper-extras]
[2020-12-02T13:18:57,041][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [parent-join]
[2020-12-02T13:18:57,042][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [percolator]
[2020-12-02T13:18:57,042][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [rank-eval]
[2020-12-02T13:18:57,042][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [reindex]
[2020-12-02T13:18:57,043][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [repository-url]
[2020-12-02T13:18:57,044][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [transport-netty4]
[2020-12-02T13:18:57,044][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-ccr]
[2020-12-02T13:18:57,045][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-core]
[2020-12-02T13:18:57,045][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-deprecation]
[2020-12-02T13:18:57,046][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-graph]
[2020-12-02T13:18:57,047][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-ilm]
[2020-12-02T13:18:57,048][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-logstash]
[2020-12-02T13:18:57,048][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-ml]
[2020-12-02T13:18:57,049][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-monitoring]
[2020-12-02T13:18:57,049][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-rollup]
[2020-12-02T13:18:57,051][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-security]
[2020-12-02T13:18:57,051][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-sql]
[2020-12-02T13:18:57,052][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] loaded module [x-pack-watcher]
[2020-12-02T13:18:57,053][INFO ][o.e.p.PluginsService     ] [lixin-macbook.local] no plugins loaded
[2020-12-02T13:19:02,046][INFO ][o.e.x.s.a.s.FileRolesStore] [lixin-macbook.local] parsed [0] roles from file [/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/config/roles.yml]
[2020-12-02T13:19:02,925][INFO ][o.e.x.m.p.l.CppLogMessageHandler] [lixin-macbook.local] [controller/1068] [Main.cc@109] controller (64 bit): Version 7.1.0-SNAPSHOT (Build a8ee6de8087169) Copyright (c) 2019 Elasticsearch BV
[2020-12-02T13:19:03,489][DEBUG][o.e.a.ActionModule       ] [lixin-macbook.local] Using REST wrapper from plugin org.elasticsearch.xpack.security.Security
[2020-12-02T13:19:04,057][INFO ][o.e.d.DiscoveryModule    ] [lixin-macbook.local] using discovery type [zen] and seed hosts providers [settings]
[2020-12-02T13:19:05,042][INFO ][o.e.n.Node               ] [lixin-macbook.local] initialized
[2020-12-02T13:19:05,042][INFO ][o.e.n.Node               ] [lixin-macbook.local] starting ...
[2020-12-02T13:19:05,223][INFO ][o.e.t.TransportService   ] [lixin-macbook.local] publish_address {127.0.0.1:9300}, bound_addresses {[::1]:9300}, {127.0.0.1:9300}
[2020-12-02T13:19:05,233][WARN ][o.e.b.BootstrapChecks    ] [lixin-macbook.local] the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
[2020-12-02T13:19:05,253][INFO ][o.e.c.c.ClusterBootstrapService] [lixin-macbook.local] no discovery configuration found, will perform best-effort cluster bootstrapping after [3s] unless existing master is discovered
[2020-12-02T13:19:08,263][INFO ][o.e.c.c.Coordinator      ] [lixin-macbook.local] setting initial configuration to VotingConfiguration{XTcpVqUhQfymSuzxG5AivA}
[2020-12-02T13:19:08,524][INFO ][o.e.c.s.MasterService    ] [lixin-macbook.local] elected-as-master ([1] nodes joined)[{lixin-macbook.local}{XTcpVqUhQfymSuzxG5AivA}{B_f1eK3cQLy_KT0PIo305g}{127.0.0.1}{127.0.0.1:9300}{ml.machine_memory=8589934592, xpack.installed=true, ml.max_open_jobs=20} elect leader, _BECOME_MASTER_TASK_, _FINISH_ELECTION_], term: 1, version: 1, reason: master node changed {previous [], current [{lixin-macbook.local}{XTcpVqUhQfymSuzxG5AivA}{B_f1eK3cQLy_KT0PIo305g}{127.0.0.1}{127.0.0.1:9300}{ml.machine_memory=8589934592, xpack.installed=true, ml.max_open_jobs=20}]}
[2020-12-02T13:19:08,605][INFO ][o.e.c.c.CoordinationState] [lixin-macbook.local] cluster UUID set to [AOtIN7N8QVmE_aRPF3x5Mw]
# 监听了:9200/9300端口
[2020-12-02T13:19:08,673][INFO ][o.e.c.s.ClusterApplierService] [lixin-macbook.local] master node changed {previous [], current [{lixin-macbook.local}{XTcpVqUhQfymSuzxG5AivA}{B_f1eK3cQLy_KT0PIo305g}{127.0.0.1}{127.0.0.1:9300}{ml.machine_memory=8589934592, xpack.installed=true, ml.max_open_jobs=20}]}, term: 1, version: 1, reason: Publication{term=1, version=1}
[2020-12-02T13:19:08,729][INFO ][o.e.h.AbstractHttpServerTransport] [lixin-macbook.local] publish_address {127.0.0.1:9200}, bound_addresses {[::1]:9200}, {127.0.0.1:9200}
[2020-12-02T13:19:08,730][INFO ][o.e.n.Node               ] [lixin-macbook.local] started
[2020-12-02T13:19:08,752][WARN ][o.e.x.s.a.s.m.NativeRoleMappingStore] [lixin-macbook.local] Failed to clear cache for realms [[]]
[2020-12-02T13:19:08,831][INFO ][o.e.g.GatewayService     ] [lixin-macbook.local] recovered [0] indices into cluster_state
[2020-12-02T13:19:09,026][INFO ][o.e.c.m.MetaDataIndexTemplateService] [lixin-macbook.local] adding template [.watch-history-9] for index patterns [.watcher-history-9*]
[2020-12-02T13:19:09,105][INFO ][o.e.c.m.MetaDataIndexTemplateService] [lixin-macbook.local] adding template [.watches] for index patterns [.watches*]
[2020-12-02T13:19:09,180][INFO ][o.e.c.m.MetaDataIndexTemplateService] [lixin-macbook.local] adding template [.triggered_watches] for index patterns [.triggered_watches*]
[2020-12-02T13:19:09,256][INFO ][o.e.c.m.MetaDataIndexTemplateService] [lixin-macbook.local] adding template [.monitoring-logstash] for index patterns [.monitoring-logstash-7-*]
[2020-12-02T13:19:09,346][INFO ][o.e.c.m.MetaDataIndexTemplateService] [lixin-macbook.local] adding template [.monitoring-es] for index patterns [.monitoring-es-7-*]
[2020-12-02T13:19:09,429][INFO ][o.e.c.m.MetaDataIndexTemplateService] [lixin-macbook.local] adding template [.monitoring-beats] for index patterns [.monitoring-beats-7-*]
[2020-12-02T13:19:09,499][INFO ][o.e.c.m.MetaDataIndexTemplateService] [lixin-macbook.local] adding template [.monitoring-alerts-7] for index patterns [.monitoring-alerts-7]
[2020-12-02T13:19:09,575][INFO ][o.e.c.m.MetaDataIndexTemplateService] [lixin-macbook.local] adding template [.monitoring-kibana] for index patterns [.monitoring-kibana-7-*]
[2020-12-02T13:19:09,647][INFO ][o.e.x.i.a.TransportPutLifecycleAction] [lixin-macbook.local] adding index lifecycle policy [watch-history-ilm-policy]
[2020-12-02T13:19:09,844][INFO ][o.e.l.LicenseService     ] [lixin-macbook.local] license [4988d201-f37a-4ef0-aca3-7653b6790770] mode [basic] - valid
```

### (6). 总结