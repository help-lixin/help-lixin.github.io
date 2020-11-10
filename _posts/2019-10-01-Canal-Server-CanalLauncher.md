---
layout: post
title: 'Canal Server源码一(CanalLauncher)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). Canal Server开启远程断点
> 修改startup.sh添加远程断点

```
 JAVA_OPTS="-server -Xms2048m -Xmx3072m -Xmn1024m -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,address=2345,server=y,suspend=y  -XX:SurvivorRatio=2 -XX:PermSize=96m -XX:MaxPermSize=256m -Xss256k -XX:-UseAdaptiveSizePolicy -XX:MaxTenuringThreshold=15 -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:+HeapDumpOnOutOfMemoryError"
```
### (2). 启动
> lixin-macbook:canal-server lixin$ ./bin/startup.sh
### (3). Eclipse开启远程断点
!["Canal Debug"](/assets/canal/imgs/canal-debug.jpg)

### (4). CanalLauncher.main

```
package com.alibaba.otter.canal.deployer;
import java.io.FileInputStream;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import org.apache.commons.lang.BooleanUtils;
import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.alibaba.otter.canal.common.utils.AddressUtils;
import com.alibaba.otter.canal.common.utils.NamedThreadFactory;
import com.alibaba.otter.canal.instance.manager.plain.PlainCanal;
import com.alibaba.otter.canal.instance.manager.plain.PlainCanalConfigClient;

public class CanalLauncher {
    private static final String             CLASSPATH_URL_PREFIX = "classpath:";
    private static final Logger             logger               = LoggerFactory.getLogger(CanalLauncher.class);
    public static final CountDownLatch      runningLatch         = new CountDownLatch(1);
    private static ScheduledExecutorService executor             = Executors.newScheduledThreadPool(1,new NamedThreadFactory("canal-server-scan"));

    public static void main(String[] args) {
        try {
            logger.info("## set default uncaught exception handler");
            setGlobalUncaughtExceptionHandler();
            logger.info("## load canal configurations");

            // *******************************************************
            // startup.sh中显示指定了-Dcanal.conf
            // CANAL_OPTS="-Dcanal.conf=$canal_conf"
            // conf = /Users/lixin/Developer/canal-server/bin/../conf/canal.properties
            String conf = System.getProperty("canal.conf", "classpath:canal.properties");
            Properties properties = new Properties();
            if (conf.startsWith(CLASSPATH_URL_PREFIX)) { // false
                conf = StringUtils.substringAfter(conf, CLASSPATH_URL_PREFIX);
                properties.load(CanalLauncher.class.getClassLoader().getResourceAsStream(conf));
            } else {
                // 加载文件
                properties.load(new FileInputStream(conf));
            }

            // ************************************************************
            // 创建:CanalStarter,设置其:canal.properties
            final CanalStarter canalStater = new CanalStarter(properties);


            // 通过远程Manager加载配置文件
            // 判断是否有开启admin:canal.admin.manager
            // managerAddress = null
            String managerAddress = CanalController.getProperty(properties, CanalConstants.CANAL_ADMIN_MANAGER);
            if (StringUtils.isNotEmpty(managerAddress)) { // false
                String user = CanalController.getProperty(properties, CanalConstants.CANAL_ADMIN_USER);
                String passwd = CanalController.getProperty(properties, CanalConstants.CANAL_ADMIN_PASSWD);
                String adminPort = CanalController.getProperty(properties, CanalConstants.CANAL_ADMIN_PORT, "11110");
                boolean autoRegister = BooleanUtils.toBoolean(CanalController.getProperty(properties,
                    CanalConstants.CANAL_ADMIN_AUTO_REGISTER));
                String autoCluster = CanalController.getProperty(properties, CanalConstants.CANAL_ADMIN_AUTO_CLUSTER);
                String registerIp = CanalController.getProperty(properties, CanalConstants.CANAL_REGISTER_IP);
                if (StringUtils.isEmpty(registerIp)) {
                    registerIp = AddressUtils.getHostIp();
                }
                final PlainCanalConfigClient configClient = new PlainCanalConfigClient(managerAddress,
                    user,
                    passwd,
                    registerIp,
                    Integer.parseInt(adminPort),
                    autoRegister,
                    autoCluster);
                PlainCanal canalConfig = configClient.findServer(null);
                if (canalConfig == null) {
                    throw new IllegalArgumentException("managerAddress:" + managerAddress
                                                       + " can't not found config for [" + registerIp + ":" + adminPort
                                                       + "]");
                }
                Properties managerProperties = canalConfig.getProperties();
                // merge local
                managerProperties.putAll(properties);
                int scanIntervalInSecond = Integer.valueOf(CanalController.getProperty(managerProperties,
                    CanalConstants.CANAL_AUTO_SCAN_INTERVAL,
                    "5"));
                executor.scheduleWithFixedDelay(new Runnable() {

                    private PlainCanal lastCanalConfig;

                    public void run() {
                        try {
                            if (lastCanalConfig == null) {
                                lastCanalConfig = configClient.findServer(null);
                            } else {
                                PlainCanal newCanalConfig = configClient.findServer(lastCanalConfig.getMd5());
                                if (newCanalConfig != null) {
                                    // 远程配置canal.properties修改重新加载整个应用
                                    canalStater.stop();
                                    Properties managerProperties = newCanalConfig.getProperties();
                                    // merge local
                                    managerProperties.putAll(properties);
                                    canalStater.setProperties(managerProperties);
                                    canalStater.start();

                                    lastCanalConfig = newCanalConfig;
                                }
                            }

                        } catch (Throwable e) {
                            logger.error("scan failed", e);
                        }
                    }

                }, 0, scanIntervalInSecond, TimeUnit.SECONDS);
                canalStater.setProperties(managerProperties);
            } else {
                // 设置:canal.properties
                canalStater.setProperties(properties);
            }
            // 调用start方法
            canalStater.start();
            runningLatch.await();
            executor.shutdownNow();
        } catch (Throwable e) {
            logger.error("## Something goes wrong when starting up the canal Server:", e);
        }
    }

    private static void setGlobalUncaughtExceptionHandler() {
        Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {

            @Override
            public void uncaughtException(Thread t, Throwable e) {
                logger.error("UnCaughtException", e);
            }
        });
    }

}
```
### (5). 全局canal.properties文件
```
#################################################
######### 		common argument		#############
#################################################
# tcp bind ip
canal.ip =
# register ip to zookeeper
canal.register.ip =
canal.port = 11111
canal.metrics.pull.port = 11112
# canal instance user/passwd
# canal.user = canal
# canal.passwd = E3619321C1A937C46A0D8BD1DAC39F93B27D4458

# canal admin config
#canal.admin.manager = 127.0.0.1:8089
canal.admin.port = 11110
canal.admin.user = admin
canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441

canal.zkServers =
# flush data to zk
canal.zookeeper.flush.period = 1000
canal.withoutNetty = false
# tcp, kafka, RocketMQ
canal.serverMode = tcp
# flush meta cursor/parse position to file
canal.file.data.dir = ${canal.conf.dir}
canal.file.flush.period = 1000
## memory store RingBuffer size, should be Math.pow(2,n)
canal.instance.memory.buffer.size = 16384
## memory store RingBuffer used memory unit size , default 1kb
canal.instance.memory.buffer.memunit = 1024 
## meory store gets mode used MEMSIZE or ITEMSIZE
canal.instance.memory.batch.mode = MEMSIZE
canal.instance.memory.rawEntry = true

## detecing config
canal.instance.detecting.enable = false
#canal.instance.detecting.sql = insert into retl.xdual values(1,now()) on duplicate key update x=now()
canal.instance.detecting.sql = select 1
canal.instance.detecting.interval.time = 3
canal.instance.detecting.retry.threshold = 3
canal.instance.detecting.heartbeatHaEnable = false

# support maximum transaction size, more than the size of the transaction will be cut into multiple transactions delivery
canal.instance.transaction.size =  1024
# mysql fallback connected to new master should fallback times
canal.instance.fallbackIntervalInSeconds = 60

# network config
canal.instance.network.receiveBufferSize = 16384
canal.instance.network.sendBufferSize = 16384
canal.instance.network.soTimeout = 30

# binlog filter config
canal.instance.filter.druid.ddl = true
canal.instance.filter.query.dcl = false
canal.instance.filter.query.dml = false
canal.instance.filter.query.ddl = false
canal.instance.filter.table.error = false
canal.instance.filter.rows = false
canal.instance.filter.transaction.entry = false

# binlog format/image check
canal.instance.binlog.format = ROW,STATEMENT,MIXED 
canal.instance.binlog.image = FULL,MINIMAL,NOBLOB

# binlog ddl isolation
canal.instance.get.ddl.isolation = false

# parallel parser config
canal.instance.parser.parallel = true
## concurrent thread number, default 60% available processors, suggest not to exceed Runtime.getRuntime().availableProcessors()
#canal.instance.parser.parallelThreadSize = 16
## disruptor ringbuffer size, must be power of 2
canal.instance.parser.parallelBufferSize = 256

# table meta tsdb info
canal.instance.tsdb.enable = true
canal.instance.tsdb.dir = ${canal.file.data.dir:../conf}/${canal.instance.destination:}
canal.instance.tsdb.url = jdbc:h2:${canal.instance.tsdb.dir}/h2;CACHE_SIZE=1000;MODE=MYSQL;
canal.instance.tsdb.dbUsername = canal
canal.instance.tsdb.dbPassword = canal
# dump snapshot interval, default 24 hour
canal.instance.tsdb.snapshot.interval = 24
# purge snapshot expire , default 360 hour(15 days)
canal.instance.tsdb.snapshot.expire = 360

# aliyun ak/sk , support rds/mq
canal.aliyun.accessKey =
canal.aliyun.secretKey =

#################################################
######### 		destinations		#############
#################################################
canal.destinations = example
# conf root dir
canal.conf.dir = ../conf
# auto scan instance dir add/remove and start/stop instance
canal.auto.scan = true
canal.auto.scan.interval = 5

canal.instance.tsdb.spring.xml = classpath:spring/tsdb/h2-tsdb.xml
#canal.instance.tsdb.spring.xml = classpath:spring/tsdb/mysql-tsdb.xml

canal.instance.global.mode = spring
canal.instance.global.lazy = false
canal.instance.global.manager.address = ${canal.admin.manager}
#canal.instance.global.spring.xml = classpath:spring/memory-instance.xml
canal.instance.global.spring.xml = classpath:spring/file-instance.xml
#canal.instance.global.spring.xml = classpath:spring/default-instance.xml

##################################################
######### 		     MQ 		     #############
##################################################
canal.mq.servers = 127.0.0.1:6667
canal.mq.retries = 0
canal.mq.batchSize = 16384
canal.mq.maxRequestSize = 1048576
canal.mq.lingerMs = 100
canal.mq.bufferMemory = 33554432
canal.mq.canalBatchSize = 50
canal.mq.canalGetTimeout = 100
canal.mq.flatMessage = true
canal.mq.compressionType = none
canal.mq.acks = all
#canal.mq.properties. =
canal.mq.producerGroup = test
# Set this value to "cloud", if you want open message trace feature in aliyun.
canal.mq.accessChannel = local
# aliyun mq namespace
#canal.mq.namespace =

##################################################
#########     Kafka Kerberos Info    #############
##################################################
canal.mq.kafka.kerberos.enable = false
canal.mq.kafka.kerberos.krb5FilePath = "../conf/kerberos/krb5.conf"
canal.mq.kafka.kerberos.jaasFilePath = "../conf/kerberos/jaas.conf"
```
### (6). 总结
> 1. CanalLauncher启动时会加载配置文件(CANAL_OPTS="-Dcanal.conf=/Users/lixin/Developer/canal-server/bin/../conf/canal.properties")canal.properties.
> 2. 创建:CanalStarter
> 3. 调用:CanalStarter.start方法