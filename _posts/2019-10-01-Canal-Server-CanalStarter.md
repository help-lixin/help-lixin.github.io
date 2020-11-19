---
layout: post
title: 'Canal Server源码之二(CanalStarter)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). CanalStarter
> CanalLauncher读取配置文件,并调用:CanalStarter.start方法

```
public class CanalStarter {

    private static final Logger logger          = LoggerFactory.getLogger(CanalStarter.class);

    // *******************************************************
    // Canal控制器入口
    private CanalController     controller      = null;
    // 抽象MQ生产者
    private CanalMQProducer     canalMQProducer = null;
    // 关闭线程
    private Thread              shutdownThread  = null;
    
    private CanalMQStarter      canalMQStarter  = null;
    // 全局:canal.properties
    private volatile Properties properties;
    // 运行标记
    private volatile boolean    running         = false;
    //     
    private CanalAdminWithNetty canalAdmin;

    public synchronized void start() throws Throwable {
        // canal.serverMode = tcp
        String serverMode = CanalController.getProperty(properties, CanalConstants.CANAL_SERVER_MODE);
        if (serverMode.equalsIgnoreCase("kafka")) { // false
            canalMQProducer = new CanalKafkaProducer();
        } else if (serverMode.equalsIgnoreCase("rocketmq")) { //false
            canalMQProducer = new CanalRocketMQProducer();
        }

        if (canalMQProducer != null) {// flase
            // disable netty
            System.setProperty(CanalConstants.CANAL_WITHOUT_NETTY, "true");
            // 设置为raw避免ByteString->Entry的二次解析
            System.setProperty("canal.instance.memory.rawEntry", "false");
        }

        logger.info("## start the canal server.");
        // **********************************************
        // 创建:CanalController
        controller = new CanalController(properties);
        // 开始启动
        controller.start();
        logger.info("## the canal server is running now ......");
        // 配置关闭线程
        shutdownThread = new Thread() {
            public void run() {
                try {
                    logger.info("## stop the canal server");
                    controller.stop();
                    CanalLauncher.runningLatch.countDown();
                } catch (Throwable e) {
                    logger.warn("##something goes wrong when stopping canal Server:", e);
                } finally {
                    logger.info("## canal server is down.");
                }
            }

        };
        // 添加关闭Hook
        Runtime.getRuntime().addShutdownHook(shutdownThread);

        if (canalMQProducer != null) { // false
            canalMQStarter = new CanalMQStarter(canalMQProducer);
            MQProperties mqProperties = buildMQProperties(properties);
            String destinations = CanalController.getProperty(properties, CanalConstants.CANAL_DESTINATIONS);
            canalMQStarter.start(mqProperties, destinations);
            controller.setCanalMQStarter(canalMQStarter);
        }

        // start canalAdmin
        // canal.admin.port = 11110
        String port = properties.getProperty(CanalConstants.CANAL_ADMIN_PORT);
        if (canalAdmin == null && StringUtils.isNotEmpty(port)) { // true
            //canal.admin.user = admin
            String user = properties.getProperty(CanalConstants.CANAL_ADMIN_USER);
            // canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441
            String passwd = properties.getProperty(CanalConstants.CANAL_ADMIN_PASSWD);

            
            CanalAdminController canalAdmin = new CanalAdminController(this);
            canalAdmin.setUser(user);
            canalAdmin.setPasswd(passwd);

            String ip = properties.getProperty(CanalConstants.CANAL_IP);

            // ****************************************************
            // 创建:CanalAdminWithNetty
            CanalAdminWithNetty canalAdminWithNetty = CanalAdminWithNetty.instance();
            canalAdminWithNetty.setCanalAdmin(canalAdmin);
            canalAdminWithNetty.setPort(Integer.valueOf(port));
            canalAdminWithNetty.setIp(ip);
            canalAdminWithNetty.start();
            this.canalAdmin = canalAdminWithNetty;
        }
        running = true;
    }// end start
}
```
### (2). 总结
> 1. 创建CanalController.start()方法
> 2. 创建CanalAdminWithNetty,端口为:11110


```
lixin-macbook:canal-server lixin$ lsof -nP -p 17747|grep LISTEN
java    17747 lixin   95u    IPv4 0x73965c583e0d9249      0t0                 TCP *:11112 (LISTEN)
java    17747 lixin  113u    IPv4 0x73965c5844d62609      0t0                 TCP *:11111 (LISTEN)
java    17747 lixin  117u    IPv4 0x73965c584cc4d249      0t0                 TCP *:11110 (LISTEN)
```