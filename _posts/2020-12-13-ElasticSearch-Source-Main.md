---
layout: post
title: 'ElasticSearch 源码入口Elasticsearch(一)'
date: 2020-12-13
author: 李新
tags: ElasticSearch源码
---

### (1). 查找Main入口
> elasticsearch-7.1.0/bin/elasticsearch

```
lixin-macbook:~ lixin$ jps
2821 Elasticsearch


lixin-macbook:~ lixin$ ps -ef|grep 2821
501  2821   588   0  1:49下午 ttys000    0:37.29 /Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/bin/java -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -Des.networkaddress.cache.ttl=60 -Des.networkaddress.cache.negative.ttl=10 -XX:+AlwaysPreTouch -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -XX:-OmitStackTraceInFastThrow -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Djava.io.tmpdir=/var/folders/l2/v7kxnww15mjb9sps4yb25sqh0000gn/T/elasticsearch-3886936329023304367 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=data -XX:ErrorFile=logs/hs_err_pid%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintGCApplicationStoppedTime -Xloggc:logs/gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=32 -XX:GCLogFileSize=64m -Dio.netty.allocator.type=unpooled -Des.path.home=/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0 -Des.path.conf=/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/config -Des.distribution.flavor=default -Des.distribution.type=tar -Des.bundled_jdk=true -cp /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/lib/* org.elasticsearch.bootstrap.Elasticsearch
```
### (2). 编译脚本(增加远程断点)
> elasticsearch-7.1.0/bin/elasticsearch

```
ES_JAVA_OPTS="${JVM_OPTIONS//\$\{ES_TMPDIR\}/$ES_TMPDIR} $ES_JAVA_OPTS -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,address=23456,server=y,suspend=y"
```

### (3). Elasticsearch类结构图
!["Elasticsearch 类结构图"](/assets/elasticsearch/imgs/Elasticsearch-class.jpg)

### (4). Elasticsearch即为:Main入口
```
public static void main(final String[] args) throws Exception {
    // *******************************************************************
    // 5. 重写DNS缓存
    // *******************************************************************
    overrideDnsCachePolicyProperties();
   
    System.setSecurityManager(new SecurityManager() {
        @Override
        public void checkPermission(Permission perm) {
            // grant all permissions so that we can later set the security manager to the one that we want
        }

    });
    LogConfigurator.registerErrorListener();

    // *******************************************************************
    // 6. 创建:Elasticsearch
    // *******************************************************************
    final Elasticsearch elasticsearch = new Elasticsearch();


    // *******************************************************************
    // 7. main
    // Terminal.DEFAULT实际就是包裹了:System.console()
    // *******************************************************************
    int status = main(args, elasticsearch, Terminal.DEFAULT);

    if (status != ExitCodes.OK) {
        exit(status);
    }
} //end main
```
### (5). Elasticsearch.overrideDnsCachePolicyProperties
> networkaddress.cache.ttl             : 解析正确的域名,保存60秒      
> networkaddress.cache.negative.ttl    : 解析错误的域名,每隔10秒发起一次解析        

```
private static void overrideDnsCachePolicyProperties() {
    for (final String property : new String[] {"networkaddress.cache.ttl", "networkaddress.cache.negative.ttl" }) {
        final String overrideProperty = "es." + property;

        // 从系统参数(命令行参数)中读取配置:
        // es.networkaddress.cache.ttl = 60
        // es.networkaddress.cache.negative.ttl = 10
        final String overrideValue = System.getProperty(overrideProperty);
        if (overrideValue != null) {
            try {
                // networkaddress.cache.ttl
                // networkaddress.cache.negative.ttl
                // 覆盖JDK自带的DNS缓存
                Security.setProperty(property, Integer.toString(Integer.valueOf(overrideValue)));
            } catch (final NumberFormatException e) {
                throw new IllegalArgumentException(
                        "failed to parse [" + overrideProperty + "] with value [" + overrideValue + "]", e);
            }
        }
    }
}// end overrideDnsCachePolicyProperties
```
### (6). new Elasticsearch
> 1. 调用父类,设置启动系统前的回调函数(Runnable).   
> 2. 配置命令行解析. 


```
// ==================================Elasticsearch==================================
private final OptionSpecBuilder versionOption;
private final OptionSpecBuilder daemonizeOption;
private final OptionSpec<Path> pidfileOption;
private final OptionSpecBuilder quietOption;

// visible for testing
Elasticsearch() {
    // ************************************************************
    // 5.1 调用父类EnvironmentAwareCommand/Command
    // 调用父类,传递描述和一个空的Runnable
    // ************************************************************
    super("starts elasticsearch", () -> {});

    // 配置命令行解
    // -v : 打印版本信息并退出
    versionOption = parser.acceptsAll(Arrays.asList("V", "version"),
        "Prints elasticsearch version information and exits");
        
    // -d  : 在后台运行
    daemonizeOption = parser.acceptsAll(Arrays.asList("d", "daemonize"),
        "Starts Elasticsearch in the background")
        .availableUnless(versionOption);

    // -p : 创建pid文件,配置value解析
    //    
    pidfileOption = parser.acceptsAll(Arrays.asList("p", "pidfile"),
        "Creates a pid file in the specified path on start")
        .availableUnless(versionOption)
        .withRequiredArg()
        .withValuesConvertedBy(new PathConverter());

    // -q :  关闭输出/错误/控制台
    quietOption = parser.acceptsAll(Arrays.asList("q", "quiet"),
        "Turns off standard output/error streams logging in console")
        .availableUnless(versionOption)
        .availableUnless(daemonizeOption);
}// end Elasticsearch构造器
// ==================================Elasticsearch==================================

// ==================================EnvironmentAwareCommand=============================
public EnvironmentAwareCommand(final String description, final Runnable beforeMain) {
    // description = "starts elasticsearch";
    // beforeMain = new Runnable(){}

    // ************************************************************
    // 5.2 调用父类:Command
    // ************************************************************
    super(description, beforeMain);
    this.settingOption = parser.accepts("E", "Configure a setting").withRequiredArg().ofType(KeyValuePair.class);
} // end EnvironmentAwareCommand 构造器
// ==================================EnvironmentAwareCommand=========================


// ==================================Command==================================
protected final String description;
private final Runnable beforeMain;

public Command(final String description, final Runnable beforeMain) {
    // description = "starts elasticsearch";
    // beforeMain = new Runnable(){}


    // ************************************************************
    // 5.3 为description和beforeMain
    // ************************************************************
    this.description = description;
    this.beforeMain = beforeMain;
}// end Command构造器
// ==================================Command==================================
```

### (7). Elasticsearch.main
```
static int main(
    // 参数
    final String[] args, 
    // ES对象
    final Elasticsearch elasticsearch, 
    // 终端(在解析参数失败时,用于输出内容)
    final Terminal terminal) throws Exception {
    // ************************************************************
    // 8.Command.main
    // ************************************************************
    return elasticsearch.main(args, terminal);
} // end main
```
### (8). Command.main
> 1. 创建进程关闭时的回调函数(线程)    
> 2. 在启动进程之前先:调用回调函数(Runnable).   
> 3. 对参数(args)进行解析,并委托调用子类:EnvironmentAwareCommand.execute方法.    

```
public final int main(String[] args, Terminal terminal) throws Exception {

    if (addShutdownHook()) { // true
        //  创建关闭Hook线程
        shutdownHookThread = new Thread(() -> {
            try {
                this.close();
            } catch (final IOException e) {
                try (
                    StringWriter sw = new StringWriter();
                    PrintWriter pw = new PrintWriter(sw)) {
                    e.printStackTrace(pw);
                    terminal.println(sw.toString());
                } catch (final IOException impossible) {
                    // StringWriter#close declares a checked IOException from the Closeable interface but the Javadocs for StringWriter
                    // say that an exception here is impossible
                    throw new AssertionError(impossible);
                }
            }
        });

        // 添加系统关闭时的回调函数
        Runtime.getRuntime().addShutdownHook(shutdownHookThread);
    }

    //beforeMain在:Elasticsearch时创建,实际是一个空对象
    // beforeMain = new Runnable(){}
    beforeMain.run();

    try {
        // **********************************************************  
        // 8.1 Command.mainWithoutErrorHandling
        // **********************************************************  
        mainWithoutErrorHandling(args, terminal);
    } catch (OptionException e) {
        printHelp(terminal);
        terminal.println(Terminal.Verbosity.SILENT, "ERROR: " + e.getMessage());
        return ExitCodes.USAGE;
    } catch (UserException e) {
        if (e.exitCode == ExitCodes.USAGE) {
            printHelp(terminal);
        }
        terminal.println(Terminal.Verbosity.SILENT, "ERROR: " + e.getMessage());
        return e.exitCode;
    }
    return ExitCodes.OK;
} // end main



void mainWithoutErrorHandling(
        String[] args, 
        Terminal terminal) throws Exception {

    // 对参数进行解析        
    final OptionSet options = parser.parse(args);

    // 如果是help
    if (options.has(helpOption)) {
        printHelp(terminal);
        return;
    }

    
    if (options.has(silentOption)) { // false
        terminal.setVerbosity(Terminal.Verbosity.SILENT);
    } else if (options.has(verboseOption)) { // false
        terminal.setVerbosity(Terminal.Verbosity.VERBOSE);
    } else { // true
        terminal.setVerbosity(Terminal.Verbosity.NORMAL);
    }


    // **********************************************************
    // 9. EnvironmentAwareCommand.execute
    // **********************************************************
    execute(terminal, options);
} //end mainWithoutErrorHandling

protected boolean addShutdownHook() {
    return true;
} // end addShutdownHook
```
### (9). EnvironmentAwareCommand.execute
```
protected void execute(Terminal terminal, OptionSet options) throws Exception {
    final Map<String, String> settings = new HashMap<>();
    // 
    for (final KeyValuePair kvp : settingOption.values(options)) { // false
        if (kvp.value.isEmpty()) {
            throw new UserException(ExitCodes.USAGE, "setting [" + kvp.key + "] must not be empty");
        }
        if (settings.containsKey(kvp.key)) {
            final String message = String.format(
                    Locale.ROOT,
                    "setting [%s] already set, saw [%s] and [%s]",
                    kvp.key,
                    settings.get(kvp.key),
                    kvp.value);
            throw new UserException(ExitCodes.USAGE, message);
        }
        settings.put(kvp.key, kvp.value);
    }

    // 获取系统启动时的变量,添加到settings(Map)中
    putSystemPropertyIfSettingIsMissing(settings, "path.data", "es.path.data");
    putSystemPropertyIfSettingIsMissing(settings, "path.home", "es.path.home");
    putSystemPropertyIfSettingIsMissing(settings, "path.logs", "es.path.logs");
    // settings = { "path.home" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0" }

    // *****************************************************************
    // 10.创建:org.elasticsearch.env.Environment
    // 11. 委托给子类:Elasticsearch.execute去执行
    // *****************************************************************
    execute(terminal, options, createEnv(settings));
} //end execute


private static void putSystemPropertyIfSettingIsMissing(
    final Map<String, String> settings, 
    final String setting, 
    final String key) {
    
    // 读取系统启动时的环境变量:
    // es.path.data = null
    // es.path.home = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0"
    // es.path.logs = null
    final String value = System.getProperty(key);
    if (value != null) {
        // setting在map中已经存在,则抛出异常    
        if (settings.containsKey(setting)) {
            final String message =
                    String.format(
                            Locale.ROOT,
                            "duplicate setting [%s] found via command-line [%s] and system property [%s]",
                            setting,
                            settings.get(setting),
                            value);
            throw new IllegalArgumentException(message);
        } else {
            // 添加到Map中
            // setting key包含:
            // path.data
            // path.home
            // path.logs
            settings.put(setting, value);
        }
    }
}// end putSystemPropertyIfSettingIsMissing

```
### (10). EnvironmentAwareCommand.createEnv
```
protected Environment createEnv(
        final Map<String, String> settings) throws UserException {

    // esPathConf = /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/config        
    final String esPathConf = System.getProperty("es.path.conf");
    
    if (esPathConf == null) {
        throw new UserException(ExitCodes.CONFIG, "the system property [es.path.conf] must be set");
    }

    // ****************************************************
    // 解析:elasticsearch.yaml文件.并将结果保存到:Environment里.
    // ****************************************************
    return InternalSettingsPreparer.prepareEnvironment(Settings.EMPTY, settings,
            getConfigPath(esPathConf),
            // HOSTNAME is set by elasticsearch-env and elasticsearch-env.bat so it is always available
            () -> System.getenv("HOSTNAME"));
}// end createEnv
```
### (11). Elasticsearch.execute
```
protected void execute(
        Terminal terminal, 
        OptionSet options, 
        Environment env) throws UserException {

    //  没有参数       
    if (options.nonOptionArguments().isEmpty() == false) { // false
        throw new UserException(ExitCodes.USAGE, "Positional arguments not allowed, found " + options.nonOptionArguments());
    }

    // -v 
    if (options.has(versionOption)) { // false
        final String versionOutput = String.format(
            Locale.ROOT,
            "Version: %s, Build: %s/%s/%s/%s, JVM: %s",
            Build.CURRENT.getQualifiedVersion(),
            Build.CURRENT.flavor().displayName(),
            Build.CURRENT.type().displayName(),
            Build.CURRENT.shortHash(),
            Build.CURRENT.date(),
            JvmInfo.jvmInfo().version()
        );
        terminal.println(versionOutput);
        return;
    }

    // false
    final boolean daemonize = options.has(daemonizeOption);
    // null
    final Path pidFile = pidfileOption.value(options);
    // false
    final boolean quiet = options.has(quietOption);

    try {
        env.validateTmpFile();
    } catch (IOException e) {
        throw new UserException(ExitCodes.CONFIG, e.getMessage());
    }

    try {
        // ******************************************************************
        // 12. Elasticsearch.init
        // ******************************************************************
        init(daemonize, pidFile, quiet, env);
    } catch (NodeValidationException e) {
        throw new UserException(ExitCodes.CONFIG, e.getMessage());
    }
}// end execute
```
### (12). Elasticsearch.init
```
void init(
        final boolean daemonize, 
        final Path pidFile, 
        final boolean quiet, 
        Environment initialEnv)
        throws NodeValidationException, UserException {

    try {
        // **********************************************************
        // 最终委托给了:Bootstrap.init方法
        // **********************************************************
        Bootstrap.init(!daemonize, pidFile, quiet, initialEnv);
    } catch (BootstrapException | RuntimeException e) {
        // format exceptions to the console in a special way
        // to avoid 2MB stacktraces from guice, etc.
        throw new StartupException(e);
    }
} // end init
```
### (13). 总结
> 1. 获得es.path.home=/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0    
> 2. 基于这个目录(es.path.home),创建:Environment.   
> 3. Environment内部数据结构包含:Settings/dataFiles/repoFiles/configFile/pluginsFile/modulesFile/sharedDataFile.   
> 4. 最终委托给:org.elasticsearch.bootstrap.Bootstrap.init方法.    