---
layout: post
title: 'ElasticSearch 源码Bootstrap(二)'
date: 2020-12-13
author: 李新
tags: ElasticSearch源码
---

### (1). 概述
> 在上一节,剖析了:ES的main入口为:Elasticsearch.它最终会把请求委托给:Bootstrap.init方法,在这里主要剖析:Bootstrap.   

### (2). Bootstrap.init 
```

private static volatile Bootstrap INSTANCE;
private volatile Node node;
private final CountDownLatch keepAliveLatch = new CountDownLatch(1);
private final Thread keepAliveThread;
private final Spawner spawner = new Spawner();

Bootstrap() {
    // 创建keeplive线程
    keepAliveThread = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                // CountDownLatch阻塞
                keepAliveLatch.await();
            } catch (InterruptedException e) {
                // bail out
            }
        }
    }, "elasticsearch[keepAlive/" + Version.CURRENT + "]");
    // daemon线程
    keepAliveThread.setDaemon(false);

    // keep this thread alive (non daemon thread) until we shutdown
    // 当进程关闭时:CountDownLatch唤醒.
    Runtime.getRuntime().addShutdownHook(new Thread() {
        @Override
        public void run() {
            keepAliveLatch.countDown();
        }
    });
} // end Bootstrap


static void init(
            final boolean foreground,
            final Path pidFile,
            final boolean quiet,
            final Environment initialEnv) throws BootstrapException, NodeValidationException, UserException {
    
    BootstrapInfo.init();
    // 创建keeplive线程
    INSTANCE = new Bootstrap();

    // 加载:elasticsearch.keystore
    final SecureSettings keystore = loadSecureSettings(initialEnv);

    // 对配置文件进行解码
    final Environment environment = createEnvironment(pidFile, keystore, initialEnv.settings(), initialEnv.configFile());

    // environment.settings() = "lixin.macbook.local"
    LogConfigurator.setNodeName(Node.NODE_NAME_SETTING.get(environment.settings()));
    try {
        LogConfigurator.configure(environment);
    } catch (IOException e) {
        throw new BootstrapException(e);
    }

    if (environment.pidFile() != null) { // false
        try {
            PidFile.create(environment.pidFile(), true);
        } catch (IOException e) {
            throw new BootstrapException(e);
        }
    }

    // closeStandardStreams = false
    final boolean closeStandardStreams = (foreground == false) || quiet;
    try {
        if (closeStandardStreams) { // false
            final Logger rootLogger = LogManager.getRootLogger();
            final Appender maybeConsoleAppender = Loggers.findAppender(rootLogger, ConsoleAppender.class);
            if (maybeConsoleAppender != null) {
                Loggers.removeAppender(rootLogger, maybeConsoleAppender);
            }
            closeSystOut();
        }

        // 检查lucene版本与ES版本
        // fail if somebody replaced the lucene jars
        checkLucene();

        // 给线程配置异常处理
        Thread.setDefaultUncaughtExceptionHandler(new ElasticsearchUncaughtExceptionHandler());

        // ************************************************************************
        // 3. Bootstrap.setup
        // ************************************************************************
        INSTANCE.setup(true, environment);

        try {
            // any secure settings must be read during node construction
            IOUtils.close(keystore);
        } catch (IOException e) {
            throw new BootstrapException(e);
        }

        // ************************************************************************
        // 5.Bootstrap.start
        // ************************************************************************
        INSTANCE.start();

        if (closeStandardStreams) { // false
            closeSysError();
        }
    } catch (NodeValidationException | RuntimeException e) {
        // ... ...
        throw e;
    }
} // end init
```
### (3). Bootstrap.setup
```
private void setup(
        // true
        boolean addShutdownHook, 
        Environment environment) throws BootstrapException {
    
    
    Settings settings = environment.settings();

    try {
        // ******************************************************
        // 4. Spawner.spawnNativeControllers
        // 读取所有的modules,如果有native,则创建:Process
        // ******************************************************
        spawner.spawnNativeControllers(environment);
    } catch (IOException e) {
        throw new BootstrapException(e);
    }

    initializeNatives(
            environment.tmpFile(),
            BootstrapSettings.MEMORY_LOCK_SETTING.get(settings),
            BootstrapSettings.SYSTEM_CALL_FILTER_SETTING.get(settings),
            BootstrapSettings.CTRLHANDLER_SETTING.get(settings));

    // initialize probes before the security manager is installed
    initializeProbes();

    if (addShutdownHook) {
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                try {
                    IOUtils.close(node, spawner);
                    LoggerContext context = (LoggerContext) LogManager.getContext(false);
                    Configurator.shutdown(context);
                } catch (IOException ex) {
                    throw new ElasticsearchException("failed to stop node", ex);
                }
            }
        });
    }

    try {
        // look for jar hell
        final Logger logger = LogManager.getLogger(JarHell.class);
        JarHell.checkJarHell(logger::debug);
    } catch (IOException | URISyntaxException e) {
        throw new BootstrapException(e);
    }

    // Log ifconfig output before SecurityManager is installed
    IfConfig.logIfNecessary();

    // install SM after natives, shutdown hooks, etc.
    try {
        Security.configure(environment, BootstrapSettings.SECURITY_FILTER_BAD_DEFAULTS_SETTING.get(settings));
    } catch (IOException | NoSuchAlgorithmException e) {
        throw new BootstrapException(e);
    }

    
    // *******************************************************
    // Node是核心,后续会专门用几章节再讲
    // *******************************************************
    node = new Node(environment) {
        @Override
        protected void validateNodeBeforeAcceptingRequests(
            final BootstrapContext context,
            final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {
            BootstrapChecks.check(context, boundTransportAddress, checks);
        }
    };
}// end setup
```
### (4). Spawner.spawnNativeControllers
> 读取:modules目录,如果有native则创建:Process对象,加载native对象. 

```
void spawnNativeControllers(final Environment environment) throws IOException {
    if (!spawned.compareAndSet(false, true)) {
        throw new IllegalStateException("native controllers already spawned");
    }
    if (!Files.exists(environment.modulesFile())) {
        throw new IllegalStateException("modules directory [" + environment.modulesFile() + "] not found");
    }
    
    // environment.modulesFile() = /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules
    List<Path> paths = PluginsService.findPluginDirs(environment.modulesFile());
    // paths = /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules下所有的目录

    for (final Path modules : paths) {
        // 读取每一个模块:
        // 比如模块: percolator
        // 和模块下的配置文件(plugin-descriptor.properties)
        // 并转换成:PluginInfo
        final PluginInfo info = PluginInfo.readFromProperties(modules);

        // spawnPath = /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator/platform/darwin-x86_64/bin/controller
        final Path spawnPath = Platforms.nativeControllerPath(modules);

        if (!Files.isRegularFile(spawnPath)) { // true
            continue;
        }

        if (!info.hasNativeController()) {
            final String message = String.format(
                Locale.ROOT,
                "module [%s] does not have permission to fork native controller",
                modules.getFileName());
            throw new IllegalArgumentException(message);
        }

        // 通过:ProcessBuilder,加载了一个本地进程(so文件)
        // ProcessBuilder pb = new ProcessBuilder("/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/x-pack-ml/platform/darwin-x86_64/bin/controller");

        // spawnPath = /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/x-pack-ml/platform/darwin-x86_64/bin/controller

        // 创建:通过ProcessBuilder构建出:Process
        final Process process = spawnNativeController(spawnPath, environment.tmpFile());
        processes.add(process);
    }
}
```
### (5). Bootstrap.start
```
private void start() throws NodeValidationException {
    node.start();
    keepAliveThread.start();
} // end start
```
### (6). 总结
> Bootstrap会加载一些Native进程,然后,初始化:org.elasticsearch.node.Node对象.
> org.elasticsearch.node.Node留到后面几节分析,里面有大量的逻辑. 
