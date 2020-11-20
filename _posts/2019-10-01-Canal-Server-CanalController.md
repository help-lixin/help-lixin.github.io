---
layout: post
title: 'Canal Server源码之三(CanalController)'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). CanalController构造器
> 1. CanalStarter.start方法会创建:CanalController

```
public class CanalController {
    // Canal实例生成器
    private CanalInstanceGenerator    instanceGenerator;
    // 共享的全局配置
    private InstanceConfig      globalInstanceConfig;
    // 所有的实例配置信息
    private Map<String, InstanceConfig>    instanceConfigs;
    private boolean           autoScan = true;
    private InstanceAction    defaultAction;
    // 实例配置监听器
    private Map<InstanceMode, InstanceConfigMonitor> instanceConfigMonitors;


    public CanalController(final Properties properties){
        // Manager
        managerClients = MigrateMap.makeComputingMap(new Function<String, PlainCanalConfigClient>() {

            public PlainCanalConfigClient apply(String managerAddress) {
                return getManagerClient(managerAddress);
            }
        });

        // *************************************************
        // 初始化全局参数设置(以及:)
        globalInstanceConfig = initGlobalConfig(properties);
        // 创建Map
        instanceConfigs = new MapMaker().makeMap();
        // 初始化instance config
        initInstanceConfig(properties);

        // init socketChannel
        // canal.socketChannel=null
        String socketChannel = getProperty(properties, CanalConstants.CANAL_SOCKETCHANNEL);
        if (StringUtils.isNotEmpty(socketChannel)) { // false
            System.setProperty(CanalConstants.CANAL_SOCKETCHANNEL, socketChannel);
        }

        // 兼容1.1.0版本的ak/sk参数名
        // accesskey = null
        String accesskey = getProperty(properties, "canal.instance.rds.accesskey");
        // secretkey = null
        String secretkey = getProperty(properties, "canal.instance.rds.secretkey");
        if (StringUtils.isNotEmpty(accesskey)) { // false
            System.setProperty(CanalConstants.CANAL_ALIYUN_ACCESSKEY, accesskey);
        }
        if (StringUtils.isNotEmpty(secretkey)) { // false
            System.setProperty(CanalConstants.CANAL_ALIYUN_SECRETKEY, secretkey);
        }

        // 准备canal server
        // canal.ip=""
        ip = getProperty(properties, CanalConstants.CANAL_IP);
        // canal.register.ip
        registerIp = getProperty(properties, CanalConstants.CANAL_REGISTER_IP);
        // canal.port
        port = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_PORT, "11111"));
        // 11110
        adminPort = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_ADMIN_PORT, "11110"));
        
        // ***********************************************
        // 创建:CanalServerWithEmbedded实例
        embededCanalServer = CanalServerWithEmbedded.instance();
        // 设置自定义的instanceGenerator
        embededCanalServer.setCanalInstanceGenerator(instanceGenerator);
        // canal.metrics.pull.port=11112
        int metricsPort = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_METRICS_PULL_PORT, "11112"));
        embededCanalServer.setMetricsPort(metricsPort);

        // canal admin配置信息
        // canal.admin.user=admin
        this.adminUser = getProperty(properties, CanalConstants.CANAL_ADMIN_USER);
        // canal.admin.passwd=4ACFE3202A5FF5CF467898FC58AAB1D615029441
        this.adminPasswd = getProperty(properties, CanalConstants.CANAL_ADMIN_PASSWD);
        embededCanalServer.setUser(getProperty(properties, CanalConstants.CANAL_USER));
        embededCanalServer.setPasswd(getProperty(properties, CanalConstants.CANAL_PASSWD));

        // canal.withoutNetty = false
        String canalWithoutNetty = getProperty(properties, CanalConstants.CANAL_WITHOUT_NETTY);
        
        if (canalWithoutNetty == null || "false".equals(canalWithoutNetty)) {
            // ****************************************************
            // 创建:CanalServerWithNetty
            canalServer = CanalServerWithNetty.instance();
            // ip=""
            canalServer.setIp(ip);
            // port=11111
            canalServer.setPort(port);
        }

        // 处理下ip为空，默认使用hostIp暴露到zk中
        if (StringUtils.isEmpty(ip) && StringUtils.isEmpty(registerIp)) { //true
            //ip = 192.168.0.144
            ip = registerIp = AddressUtils.getHostIp();
        }

        if (StringUtils.isEmpty(ip)) { // false
            ip = AddressUtils.getHostIp();
        }

        if (StringUtils.isEmpty(registerIp)) { //false
            registerIp = ip; // 兼容以前配置
        }

        // canal.zkServers=null
        final String zkServers = getProperty(properties, CanalConstants.CANAL_ZKSERVERS);
        if (StringUtils.isNotEmpty(zkServers)) { // false
            zkclientx = ZkClientx.getZkClient(zkServers);
            // 初始化系统目录
            zkclientx.createPersistent(ZookeeperPathUtils.DESTINATION_ROOT_NODE, true);
            zkclientx.createPersistent(ZookeeperPathUtils.CANAL_CLUSTER_ROOT_NODE, true);
        }

        // 192.168.0.144:11111
        final ServerRunningData serverData = new ServerRunningData(registerIp + ":" + port);
        
        // ****************************************************
        // 配置监控信息
        ServerRunningMonitors.setServerData(serverData);
        // 配置ServerRunningMonitor
        ServerRunningMonitors.setRunningMonitors(MigrateMap.makeComputingMap(new Function<String, ServerRunningMonitor>() {

            public ServerRunningMonitor apply(final String destination) {
                ServerRunningMonitor runningMonitor = new ServerRunningMonitor(serverData);
                runningMonitor.setDestination(destination);
                runningMonitor.setListener(new ServerRunningListener() {

                    public void processActiveEnter() {
                        try {
                            // ********************************
                            // 触发创建:CanalInstanceWithSpring
                            MDC.put(CanalConstants.MDC_DESTINATION, String.valueOf(destination));
                            embededCanalServer.start(destination);
                            if (canalMQStarter != null) {
                                canalMQStarter.startDestination(destination);
                            }
                        } finally {
                            MDC.remove(CanalConstants.MDC_DESTINATION);
                        }
                    }

                    public void processActiveExit() {
                        try {
                            MDC.put(CanalConstants.MDC_DESTINATION, String.valueOf(destination));
                            if (canalMQStarter != null) {
                                canalMQStarter.stopDestination(destination);
                            }
                            embededCanalServer.stop(destination);
                        } finally {
                            MDC.remove(CanalConstants.MDC_DESTINATION);
                        }
                    }

                    public void processStart() {
                        try {
                            if (zkclientx != null) {
                                final String path = ZookeeperPathUtils.getDestinationClusterNode(destination,
                                    registerIp + ":" + port);
                                initCid(path);
                                zkclientx.subscribeStateChanges(new IZkStateListener() {

                                    public void handleStateChanged(KeeperState state) throws Exception {

                                    }

                                    public void handleNewSession() throws Exception {
                                        initCid(path);
                                    }

                                    @Override
                                    public void handleSessionEstablishmentError(Throwable error) throws Exception {
                                        logger.error("failed to connect to zookeeper", error);
                                    }
                                });
                            }
                        } finally {
                            MDC.remove(CanalConstants.MDC_DESTINATION);
                        }
                    }

                    public void processStop() {
                        try {
                            MDC.put(CanalConstants.MDC_DESTINATION, String.valueOf(destination));
                            if (zkclientx != null) {
                                final String path = ZookeeperPathUtils.getDestinationClusterNode(destination,
                                    registerIp + ":" + port);
                                releaseCid(path);
                            }
                        } finally {
                            MDC.remove(CanalConstants.MDC_DESTINATION);
                        }
                    }

                });
                if (zkclientx != null) {
                    runningMonitor.setZkClient(zkclientx);
                }
                // 触发创建一下cid节点
                runningMonitor.init();
                return runningMonitor;
            }
        }));

        // 初始化monitor机制
        // canal.auto.scan=true
        autoScan = BooleanUtils.toBoolean(getProperty(properties, CanalConstants.CANAL_AUTO_SCAN));
        if (autoScan) { //true
            defaultAction = new InstanceAction() {

                public void start(String destination) {
                    InstanceConfig config = instanceConfigs.get(destination);
                    if (config == null) {
                        // 重新读取一下instance config
                        config = parseInstanceConfig(properties, destination);
                        instanceConfigs.put(destination, config);
                    }

                    if (!embededCanalServer.isStart(destination)) {
                        // HA机制启动
                        ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);
                        if (!config.getLazy() && !runningMonitor.isStart()) {
                            runningMonitor.start();
                        }
                    }

                    logger.info("auto notify start {} successful.", destination);
                }

                public void stop(String destination) {
                    // 此处的stop，代表强制退出，非HA机制，所以需要退出HA的monitor和配置信息
                    InstanceConfig config = instanceConfigs.remove(destination);
                    if (config != null) {
                        embededCanalServer.stop(destination);
                        ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);
                        if (runningMonitor.isStart()) {
                            runningMonitor.stop();
                        }
                    }

                    logger.info("auto notify stop {} successful.", destination);
                }

                public void reload(String destination) {
                    // 目前任何配置变化，直接重启，简单处理
                    stop(destination);
                    start(destination);

                    logger.info("auto notify reload {} successful.", destination);
                }

                @Override
                public void release(String destination) {
                    // 此处的release，代表强制释放，主要针对HA机制释放运行，让给其他机器抢占
                    InstanceConfig config = instanceConfigs.get(destination);
                    if (config != null) {
                        ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);
                        if (runningMonitor.isStart()) {
                            boolean release = runningMonitor.release();
                            if (!release) {
                                // 如果是单机模式,则直接清除配置
                                instanceConfigs.remove(destination);
                                // 停掉服务
                                runningMonitor.stop();
                                if (instanceConfigMonitors.containsKey(InstanceConfig.InstanceMode.MANAGER)) {
                                    ManagerInstanceConfigMonitor monitor = (ManagerInstanceConfigMonitor) instanceConfigMonitors.get(InstanceConfig.InstanceMode.MANAGER);
                                    Map<String, InstanceAction> instanceActions = monitor.getActions();
                                    if (instanceActions.containsKey(destination)) {
                                        // 清除内存中的autoScan cache
                                        monitor.release(destination);
                                    }
                                }
                            }
                        }
                    }

                    logger.info("auto notify release {} successful.", destination);
                }
            };

            instanceConfigMonitors = MigrateMap.makeComputingMap(new Function<InstanceMode, InstanceConfigMonitor>() {

                public InstanceConfigMonitor apply(InstanceMode mode) {
                    int scanInterval = Integer.valueOf(getProperty(properties,
                        CanalConstants.CANAL_AUTO_SCAN_INTERVAL,
                        "5"));

                    if (mode.isSpring()) {
                        SpringInstanceConfigMonitor monitor = new SpringInstanceConfigMonitor();
                        monitor.setScanIntervalInSecond(scanInterval);
                        monitor.setDefaultAction(defaultAction);
                        // 设置conf目录，默认是user.dir + conf目录组成
                        String rootDir = getProperty(properties, CanalConstants.CANAL_CONF_DIR);
                        if (StringUtils.isEmpty(rootDir)) {
                            rootDir = "../conf";
                        }

                        if (StringUtils.equals("otter-canal", System.getProperty("appName"))) {
                            monitor.setRootConf(rootDir);
                        } else {
                            // eclipse debug模式
                            monitor.setRootConf("src/main/resources/");
                        }
                        return monitor;
                    } else if (mode.isManager()) {
                        ManagerInstanceConfigMonitor monitor = new ManagerInstanceConfigMonitor();
                        monitor.setScanIntervalInSecond(scanInterval);
                        monitor.setDefaultAction(defaultAction);
                        String managerAddress = getProperty(properties, CanalConstants.CANAL_ADMIN_MANAGER);
                        monitor.setConfigClient(getManagerClient(managerAddress));
                        return monitor;
                    } else {
                        throw new UnsupportedOperationException("unknow mode :" + mode + " for monitor");
                    }
                }
            });
        }
    }// end CanalController构造器


    private InstanceConfig initGlobalConfig(Properties properties) {
        // canal.admin.manager
        // adminManagerAddress = null
        String adminManagerAddress = getProperty(properties, CanalConstants.CANAL_ADMIN_MANAGER);
        
        // 创建实例配置信息
        InstanceConfig globalConfig = new InstanceConfig();

        // canal.instance.global.mode
        // modeStr = spring
        String modeStr = getProperty(properties, CanalConstants.getInstanceModeKey(CanalConstants.GLOBAL_NAME));
        if (StringUtils.isNotEmpty(adminManagerAddress)) { //false
            // 如果指定了manager地址,则强制适用manager
            globalConfig.setMode(InstanceMode.MANAGER);
        } else if (StringUtils.isNotEmpty(modeStr)) {// true
            // 设置mode为spring
            globalConfig.setMode(InstanceMode.valueOf(StringUtils.upperCase(modeStr)));
        }

        // canal.instance.global.lazy=false
        String lazyStr = getProperty(properties, CanalConstants.getInstancLazyKey(CanalConstants.GLOBAL_NAME));
        if (StringUtils.isNotEmpty(lazyStr)) { //true
            globalConfig.setLazy(Boolean.valueOf(lazyStr));
        }

        // canal.instance.global.manager.address = ${canal.admin.manager}
        String managerAddress = getProperty(properties,
            CanalConstants.getInstanceManagerAddressKey(CanalConstants.GLOBAL_NAME));
        if (StringUtils.isNotEmpty(managerAddress)) { //true
            if (StringUtils.equals(managerAddress, "${canal.admin.manager}")) {
                // managerAddress = null
                managerAddress = adminManagerAddress;
            }

            globalConfig.setManagerAddress(managerAddress);
        }

        
        // canal.instance.global.spring.xml = classpath:spring/file-instance.xml
        String springXml = getProperty(properties, CanalConstants.getInstancSpringXmlKey(CanalConstants.GLOBAL_NAME));
        if (StringUtils.isNotEmpty(springXml)) {
            // springXml = classpath:spring/file-instance.xml
            globalConfig.setSpringXml(springXml);
        }

        // **********************************************
        // 创建实例的生成器
        instanceGenerator = new CanalInstanceGenerator() {

            public CanalInstance generate(String destination) {
                InstanceConfig config = instanceConfigs.get(destination);
                if (config == null) {
                    throw new CanalServerException("can't find destination:" + destination);
                }

                if (config.getMode().isManager()) {
                    PlainCanalInstanceGenerator instanceGenerator = new PlainCanalInstanceGenerator(properties);
                    instanceGenerator.setCanalConfigClient(managerClients.get(config.getManagerAddress()));
                    instanceGenerator.setSpringXml(config.getSpringXml());
                    return instanceGenerator.generate(destination);
                } else if (config.getMode().isSpring()) {
                    SpringCanalInstanceGenerator instanceGenerator = new SpringCanalInstanceGenerator();
                    instanceGenerator.setSpringXml(config.getSpringXml());
                    return instanceGenerator.generate(destination);
                } else {
                    throw new UnsupportedOperationException("unknow mode :" + config.getMode());
                }

            }

        };
        return globalConfig;
    }//end initGlobalConfig


    private void initInstanceConfig(Properties properties) {
        // canal.destinations=example
        String destinationStr = getProperty(properties, CanalConstants.CANAL_DESTINATIONS);
        // 按逗号分隔
        // destinations = ["example"]
        String[] destinations = StringUtils.split(destinationStr, CanalConstants.CANAL_DESTINATION_SPLIT);

        for (String destination : destinations) { 
            InstanceConfig config = parseInstanceConfig(properties, destination);
            // key:example  value:InstanceConfig[globalConfig=InstanceConfig[globalConfig=<null>,mode=SPRING,lazy=false,managerAddress=<null>,springXml=classpath:spring/file-instance.xml],mode=<null>,lazy=<null>,managerAddress=<null>,springXml=<null>]

            InstanceConfig oldConfig = instanceConfigs.put(destination, config);

            if (oldConfig != null) { //false
                //  相应的key已经存在,给出提示信息
                logger.warn("destination:{} old config:{} has replace by new config:{}", destination, oldConfig, config);
            }
        }
    }// end initInstanceConfig


    private InstanceConfig parseInstanceConfig(Properties properties, String destination) {

        // 在canal.properties可配置部分实例(example)信息
        // properties = canal.properties
        // destination = example

        // canal.admin.manager=null
        String adminManagerAddress = getProperty(properties, CanalConstants.CANAL_ADMIN_MANAGER);

        // 实例配置信息,包含着全局共享的配置信息
        InstanceConfig config = new InstanceConfig(globalInstanceConfig);
        // canal.instance.example.mode=null
        String modeStr = getProperty(properties, CanalConstants.getInstanceModeKey(destination));
        if (StringUtils.isNotEmpty(adminManagerAddress)) {// false
            // 如果指定了manager地址,则强制适用manager
            config.setMode(InstanceMode.MANAGER);
        } else if (StringUtils.isNotEmpty(modeStr)) {//false
            config.setMode(InstanceMode.valueOf(StringUtils.upperCase(modeStr)));
        }

        // canal.instance.example.lazy=null
        String lazyStr = getProperty(properties, CanalConstants.getInstancLazyKey(destination));
        if (!StringUtils.isEmpty(lazyStr)) {
            config.setLazy(Boolean.valueOf(lazyStr));
        }

        if (config.getMode().isManager()) { //false
            String managerAddress = getProperty(properties, CanalConstants.getInstanceManagerAddressKey(destination));
            if (StringUtils.isNotEmpty(managerAddress)) {
                if (StringUtils.equals(managerAddress, "${canal.admin.manager}")) {
                    managerAddress = adminManagerAddress;
                }
                config.setManagerAddress(managerAddress);
            }
        } else if (config.getMode().isSpring()) { // (因为全局共享配置信息配置为Spring)true
            // canal.instance.example.spring.xml=null
            String springXml = getProperty(properties, CanalConstants.getInstancSpringXmlKey(destination));
            if (StringUtils.isNotEmpty(springXml)) {
                config.setSpringXml(springXml);
            }
        }
        return config;
    }// end parseInstanceConfig
}
```

### (2). CanalController.start

> CanalStarter.start方法会调用:CanalController.start方法

```
public void start() throws Throwable {
    logger.info("## start the canal server[{}({}):{}]", ip, registerIp, port);
    // 创建整个canal的工作节点
    // path = /otter/canal/cluster/192.168.0.144:11111
    final String path = ZookeeperPathUtils.getCanalClusterNode(registerIp + ":" + port);

    initCid(path);
    if (zkclientx != null) { // false
        this.zkclientx.subscribeStateChanges(new IZkStateListener() {

            public void handleStateChanged(KeeperState state) throws Exception {

            }

            public void handleNewSession() throws Exception {
                initCid(path);
            }

            @Override
            public void handleSessionEstablishmentError(Throwable error) throws Exception {
                logger.error("failed to connect to zookeeper", error);
            }
        });
    }

    // ******************************************************
    // 优先启动embeded服务
    // com.alibaba.otter.canal.server.embedded.CanalServerWithEmbedded.start
    embededCanalServer.start();

    // 尝试启动一下非lazy状态的通道
    // 遍历所有的实例配置信息
    // key=example   value:InstanceConfig
    for (Map.Entry<String, InstanceConfig> entry : instanceConfigs.entrySet()) {
        // destination = example
        final String destination = entry.getKey();
        // config = InstanceConfig
        InstanceConfig config = entry.getValue();
        // 创建destination的工作节点
        if (!embededCanalServer.isStart(destination)) {  //!(false)
            // ServerRunningMonitors在CanalController构造器中有做注册
            // 会延迟触发:apply方法
            ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);

            if (!config.getLazy() && !runningMonitor.isStart()) { //true
                runningMonitor.start();
            }
        }

        if (autoScan) {
            instanceConfigMonitors.get(config.getMode()).register(destination, defaultAction);
        }
    }

    if (autoScan) {
        instanceConfigMonitors.get(globalInstanceConfig.getMode()).start();
        for (InstanceConfigMonitor monitor : instanceConfigMonitors.values()) {
            if (!monitor.isStart()) {
                monitor.start();
            }
        }
    }

    // 启动网络接口
    if (canalServer != null) {
        canalServer.start();
    }
}// end stop


private void initCid(String path) {
    // logger.info("## init the canalId = {}", cid);
    // 初始化系统目录
    if (zkclientx != null) { // false
        try {
            zkclientx.createEphemeral(path);
        } catch (ZkNoNodeException e) {
            // 如果父目录不存在，则创建
            String parentDir = path.substring(0, path.lastIndexOf('/'));
            zkclientx.createPersistent(parentDir, true);
            zkclientx.createEphemeral(path);
        } catch (ZkNodeExistsException e) {
            // ignore
            // 因为第一次启动时创建了cid,但在stop/start的时可能会关闭和新建,允许出现NodeExists问题s
        }

    }
}// end initCid
```

### (3). UML图解:CanalController创建过程
> 由于Mac一直没有找到合适的UML工具,暂时使用(UMLet),发现UMLet没有导出图片功能,所以,只能以片断的方式来画UML.     

!["CanalController构造器"](/assets/canal/imgs/canal-controller-initGlobalConfig.png)
!["CanalController构造器"](/assets/canal/imgs/canal-controller-netty-init-1.jpg)
!["CanalController构造器"](/assets/canal/imgs/canal-controller-netty-init-2.jpg)
!["CanalController构造器"](/assets/canal/imgs/canal-controller-running-monitors.jpg)

CanalController构造器初始化过程UML文件: 
["CanalController构造器初始化过程UML"](/assets/uml/canal/canal-controller-init.uxf)


### (4). UML图解:CanalController start过程


### (5). 总结
---

> 构造器总结   
> 1. 在构造器中创建:CanalInstanceGenerator  
> 2. 在构造器中创建:CanalServerWithEmbedded,并配置生成策略CanalInstanceGenerator  
> 3. 在构造器中创建:CanalServerWithNetty  
> 4. 在构造器中创建:ServerRunningData(ip+port)  
> 5. ServerRunningMonitors.setRunningMonitors注册运行监听器信息  

---
> start总结    
> 1. 调用:CanalServerWithEmbedded.start方法,初始化内部的:canalInstances为,CanalInstanceGenerator    
> 2. 遍历所有的instance,并调用:ServerRunningMonitors.getRunningMonitor(xxx),访方法会回调CanalController构造器,ServerRunningMonitors.setRunningMonitors(xxx)中的方法,会创建出一个ServerRunningMonitor,并配置监听器,并初始化    
> 3. 调用ServerRunningMonitor.start方法,会回调到**第2步**中为ServerRunningMonitor配置的监听器(processActiveEnter())    
> 4. ServerRunningMonitor.processActiveEnter方法内部会调用:CanalServerWithEmbedded.start(destination)    
> 5. <font color='red'>CanalServerWithEmbedded.start方法回调用构造器初始中的第1步,CanalInstanceGenerator.generate(destination)</font>   
> 6. <font color='red'>把destination压入环境变量(System.setProperty("canal.instance.destination", "example")),这样做的目的是因为:file-instance.xml中会加载:classpath:${canal.instance.destination:}/instance.properties</font>       
> 7. 创建:ClassPathXmlApplicationContext,加载:classpath:spring/file-instance.xml,并根据id(instance),获取实例:(CanalInstanceWithSpring)    
> 8. **清空环境变量**(System.setProperty("canal.instance.destination", ""))
> 9. 调用CanalServerWithNetty.start()方法.
> 10. <font color='red'>CanalServerWithNetty</font>为Netty的包装类,负责与Canal Client进行通信处理.
> 11. <font color='red'>CanalServerWithEmbedded</font>为Canal Server与MySQL进行通信管理(auth/dump/...).
> 12. 总结:在测试的时候,只要设置:System.setProperty("canal.instance.destination", ""),然后,创建ApplicationContext.