---
layout: post
title: 'ElasticSearch 源码编译(二)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 下载ES源码
```
lixin-macbook:GitRepository lixin$ pwd
/Users/lixin/GitRepository
lixin-macbook:GitRepository lixin$ git clone -b v5.6.16 https://github.com/elastic/elasticsearch.git elasticsearch-v5.6.16
```
### (2). 配置阿里maven
>  vi  ~/.gradle/init.gradle

```
allprojects{
    repositories {
        def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public/'
        all { ArtifactRepository repo ->
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                    remove repo
                }
            }
        }
        maven {
            url REPOSITORY_URL
        }
    }
}
```
### (3). 编译并打包
```
# 我的工作目录
lixin-macbook:elasticsearch-v5.6.16 lixin$ pwd
/Users/lixin/GitRepository/elasticsearch-v5.6.16
# 打包
lixin-macbook:elasticsearch-v5.6.16 lixin$ ./gradlew  assemble
```
### (4). 打包结果
```
# 我的工作目录
lixin-macbook:elasticsearch-v5.6.16 lixin$ pwd
/Users/lixin/GitRepository/elasticsearch-v5.6.16

# 打包后结果在这里(tar/build/distributions/)
lixin-macbook:elasticsearch-v5.6.16 lixin$ ll distribution/
total 72
drwxr-xr-x  14 lixin  staff    448 11 29 20:36 ./
drwxr-xr-x  41 lixin  staff   1312 11 29 22:06 ../
-rw-r--r--@  1 lixin  staff   6148 11 29 20:36 .DS_Store
drwxr-xr-x   6 lixin  staff    192 11 29 20:36 build/
-rw-r--r--   1 lixin  staff  21130  9 22 15:47 build.gradle
drwxr-xr-x   4 lixin  staff    128  9 22 22:41 bwc/
drwxr-xr-x   7 lixin  staff    224  9 22 22:31 deb/
-rw-r--r--   1 lixin  staff    713  9 22 22:41 distribution.iml
drwxr-xr-x   8 lixin  staff    256  9 22 22:31 integ-test-zip/
drwxr-xr-x   7 lixin  staff    224  9 22 22:31 rpm/
drwxr-xr-x   3 lixin  staff     96  9 22 15:47 src/
drwxr-xr-x   8 lixin  staff    256  9 22 22:31 tar/
drwxr-xr-x   5 lixin  staff    160  9 22 22:41 tools/
drwxr-xr-x   8 lixin  staff    256  9 22 22:31 zip/
```
### (5). 解压并启动
```
lixin-macbook:bin lixin$ pwd
/Users/lixin/Developer/elastic-search/elasticsearch-5.6.16/bin

# 启动ES
lixin-macbook:bin lixin$ ./elasticsearch
```

```
[2020-11-30T15:55:31,448][INFO ][o.e.n.Node               ] [] initializing ...
[2020-11-30T15:55:31,550][INFO ][o.e.e.NodeEnvironment    ] [nHwKUwc] using [1] data paths, mounts [[/System/Volumes/Data (/dev/disk1s1)]], net usable_space [273gb], net total_space [465.7gb], spins? [unknown], types [apfs]
[2020-11-30T15:55:31,550][INFO ][o.e.e.NodeEnvironment    ] [nHwKUwc] heap size [1.9gb], compressed ordinary object pointers [true]
[2020-11-30T15:55:31,553][INFO ][o.e.n.Node               ] node name [nHwKUwc] derived from node ID [nHwKUwcXQX2ESX6HuYcTxQ]; set [node.name] to override
[2020-11-30T15:55:31,553][INFO ][o.e.n.Node               ] version[5.6.17-SNAPSHOT], pid[1644], build[59bb0dc/2020-11-29T11:45:00.475Z], OS[Mac OS X/10.15.7/x86_64], JVM[Oracle Corporation/Java HotSpot(TM) 64-Bit Server VM/1.8.0_251/25.251-b08]
# 提示JVM参数,如果有本地调试时,需要知道指定哪些参数.可参考这里. 
[2020-11-30T15:55:31,553][INFO ][o.e.n.Node               ] JVM arguments [-Xms2g, -Xmx2g, -XX:+UseConcMarkSweepGC, -XX:CMSInitiatingOccupancyFraction=75, -XX:+UseCMSInitiatingOccupancyOnly, -XX:+AlwaysPreTouch, -Xss1m, -Djava.awt.headless=true, -Dfile.encoding=UTF-8, -Djna.nosys=true, -Djdk.io.permissionsUseCanonicalPath=true, -Dio.netty.noUnsafe=true, -Dio.netty.noKeySetOptimization=true, -Dio.netty.recycler.maxCapacityPerThread=0, -Dlog4j.shutdownHookEnabled=false, -Dlog4j2.disable.jmx=true, -Dlog4j.skipJansi=true, -XX:+HeapDumpOnOutOfMemoryError, -Des.path.home=/Users/lixin/Developer/elastic-search/elasticsearch-5.6.16]
[2020-11-30T15:55:31,554][WARN ][o.e.n.Node               ] version [5.6.17-SNAPSHOT] is a pre-release version of Elasticsearch and is not suitable for production
[2020-11-30T15:55:32,676][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [aggs-matrix-stats]
[2020-11-30T15:55:32,677][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [ingest-common]
[2020-11-30T15:55:32,677][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [lang-expression]
[2020-11-30T15:55:32,677][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [lang-groovy]
[2020-11-30T15:55:32,677][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [lang-mustache]
[2020-11-30T15:55:32,678][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [lang-painless]
[2020-11-30T15:55:32,678][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [parent-join]
[2020-11-30T15:55:32,678][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [percolator]
[2020-11-30T15:55:32,678][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [reindex]
[2020-11-30T15:55:32,679][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [transport-netty3]
[2020-11-30T15:55:32,679][INFO ][o.e.p.PluginsService     ] [nHwKUwc] loaded module [transport-netty4]
[2020-11-30T15:55:32,680][INFO ][o.e.p.PluginsService     ] [nHwKUwc] no plugins loaded
[2020-11-30T15:55:34,805][INFO ][o.e.d.DiscoveryModule    ] [nHwKUwc] using discovery type [zen]
[2020-11-30T15:55:35,730][INFO ][o.e.n.Node               ] initialized
[2020-11-30T15:55:35,730][INFO ][o.e.n.Node               ] [nHwKUwc] starting ...
# 监听:9200和9300端口
# 9200:为HTTP通信(Netty4HttpServerTransport)
# 9300:为二进制通信(TransportService)
[2020-11-30T15:55:35,992][INFO ][o.e.t.TransportService   ] [nHwKUwc] publish_address {127.0.0.1:9300}, bound_addresses {[::1]:9300}, {127.0.0.1:9300}
[2020-11-30T15:55:39,076][INFO ][o.e.c.s.ClusterService   ] [nHwKUwc] new_master {nHwKUwc}{nHwKUwcXQX2ESX6HuYcTxQ}{KOgzT9WoSzqlkY4MeQRj5Q}{127.0.0.1}{127.0.0.1:9300}, reason: zen-disco-elected-as-master ([0] nodes joined)
[2020-11-30T15:55:39,114][INFO ][o.e.g.GatewayService     ] [nHwKUwc] recovered [0] indices into cluster_state
[2020-11-30T15:55:39,146][INFO ][o.e.h.n.Netty4HttpServerTransport] [nHwKUwc] publish_address {127.0.0.1:9200}, bound_addresses {[::1]:9200}, {127.0.0.1:9200}
[2020-11-30T15:55:39,148][INFO ][o.e.n.Node               ] [nHwKUwc] started
```

### (6). 总结