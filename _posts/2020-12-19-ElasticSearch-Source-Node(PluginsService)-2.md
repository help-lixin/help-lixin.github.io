---
layout: post
title: 'ElasticSearch 源码Node(PluginsService)-2(四)'
date: 2020-12-19
author: 李新
tags: ElasticSearch源码
---

### (1). 概述
> 前面,对:NodeEnvironment进行了分析,这一节主要分析:PluginsService. 

### (2). Node
```
protected Node(
            final Environment environment, 
            Collection<Class<? extends Plugin>> classpathPlugins, 
            boolean forbidPrivateIndexSettings) {
    // tmpSettings = {
    //    "cluster.name" : "elasticsearch",
    //     "http.cors.allow-origin" : "*",
    //     "http.cors.enabled" : "true",
    //     "node.name" : "lixin-macbook.local",
    //     "path.home" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0",
    //     "path.logs" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/logs",   
    //     "client.type" : "node"
    // }

    // TODO ... ...

    // *********************************************************************************
    // 创建插件服务
    // *********************************************************************************
    this.pluginsService = 
            new PluginsService(
                tmpSettings,
                // 配置文件所在目录 
                // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/config
                environment.configFile(), 

                // 模块所在目录
                // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules
                environment.modulesFile(),

                // 插件所在目录 
                // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/plugins
                environment.pluginsFile(), 

                // 空的集合.
                classpathPlugins);      
    // 读取所有插件的:additionalSettings/getFeature中的配置信息
    // 更新到:       settings
    // settings = {
    //    "client.type" : "node",
    //    "cluster.name" : "elasticsearch",
    //    "http.cors.allow-origin" : true,
    //    "http.cors.enabled" : true,
    //    "node.attr.ml.machine_memory" : "8589934592",
    //    "node.attr.ml.max_open_jobs" : "20",
    //    "node.attr.xpack.installed" : true,
    //    "node.name" : "lixin-macbook.local",
    //    "path.home" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0",
    //    "path.logs" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/logs",
    //    "transport.features.x-pack" : true,
    //    "http.type" : "security4",
    //    "http.type.default" : "netty4",
    //    "transport.type" : "security4" ,
    //    "transport.type.default" : "netty4"
    // }                          
    this.settings = pluginsService.updatedSettings();

    // TODO ... ...
}
```
### (3). PluginsService

```
public class PluginsService {

    private final Settings settings;
    private final Path configPath;

    private final List<Tuple<PluginInfo, Plugin>> plugins;
    private final PluginsAndModules info;

    public PluginsService(
        Settings settings,
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/config
        Path configPath,
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules
        Path modulesDirectory,
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/plugins
        Path pluginsDirectory,
        // []空的集合
        Collection<Class<? extends Plugin>> classpathPlugins
    ) {

        this.settings = settings;
        this.configPath = configPath;

        List<Tuple<PluginInfo, Plugin>> pluginsLoaded = new ArrayList<>();
        List<PluginInfo> pluginsList = new ArrayList<>();
        final List<String> pluginsNames = new ArrayList<>();

        // classpathPlugins = []
        for (Class<? extends Plugin> pluginClass : classpathPlugins) { 
            Plugin plugin = loadPlugin(pluginClass, settings, configPath);
            PluginInfo pluginInfo = new PluginInfo(pluginClass.getName(), "classpath plugin", "NA", Version.CURRENT, "1.8",
                                                   pluginClass.getName(), Collections.emptyList(), false);
            if (logger.isTraceEnabled()) {
                logger.trace("plugin loaded from classpath [{}]", pluginInfo);
            }
            pluginsLoaded.add(new Tuple<>(pluginInfo, plugin));
            pluginsList.add(pluginInfo);
            pluginsNames.add(pluginInfo.getName());
        } // end for 


        // ********************************************************
        // 加载:modules目录下所有的:molue
        // 这个时候,实际并没有对插件初始化.
        // ********************************************************
        Set<Bundle> seenBundles = new LinkedHashSet<>();
        List<PluginInfo> modulesList = new ArrayList<>();
        // load modules
        if (modulesDirectory != null) { // false
            try {
                // ********************************************************
                // 4.加载模块(PluginsService.getModuleBundles)
                // ********************************************************
                Set<Bundle> modules = getModuleBundles(modulesDirectory);
                for (Bundle bundle : modules) {
                    // modulesList 存储:pluginInfo对象
                    modulesList.add(bundle.plugin);
                }

                // seenBundles存储:所有的:Bundle
                seenBundles.addAll(modules);
            } catch (IOException ex) {
                throw new IllegalStateException("Unable to initialize modules", ex);
            }
        } // end load modules

        // *******************************************************
        // 加载插件目录下的所有文件转换成:Bundle
        // *******************************************************
        // pluginDirectory = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/plugins"
        if (pluginsDirectory != null) {
            try {
                // 测试下目录是否可读
                if (isAccessibleDirectory(pluginsDirectory, logger)) {
                    // 检查目录下:.removing-这样的文件
                    checkForFailedPluginRemovals(pluginsDirectory);
                    // 加载插件与Modules加载方式一样,不往下分析,可以断点的一点就是:在这个时候:
                    // 插件(jar)里的对象并没有初始化.
                    Set<Bundle> plugins = getPluginBundles(pluginsDirectory);
                    for (final Bundle bundle : plugins) {
                        // pluginsList 存储: PluginInfo对象
                        pluginsList.add(bundle.plugin);
                        // pluginsNames 存储: 配置文件中指定的名称.
                        pluginsNames.add(bundle.plugin.getName());
                    }

                    // 所有的插件.
                    seenBundles.addAll(plugins);
                }
            } catch (IOException ex) {
                throw new IllegalStateException("Unable to initialize plugins", ex);
            }
        }// end load plugins


        // ***********************************************************************
        //    **************** PluginsService.loadBundle比较重要.***************
        // 1. 通过创建:ClassLoader加载jar
        // 2. 根据配置文件(plugin-descriptor.properties)中的配置
        //     [classname=org.elasticsearch.percolator.PercolatorPlugin]
        // 3. ClassLoader加载这个Class(PercolatorPlugin),PercolatorPlugin必须要是:Plugin的子类.
        //    loader.loadClass(className).asSubclass(Plugin.class);
        // 4. PercolatorPlugin必须是无参构造器,通过反射创建PercolatorPlugin对象的实例.
        // ***********************************************************************
        List<Tuple<PluginInfo, Plugin>> loaded = loadBundles(seenBundles);
        pluginsLoaded.addAll(loaded);

        // PluginsAndModules执有:module/plugin的所有:PluginInfo信息,估计是为了方便索引
        this.info = new PluginsAndModules(pluginsList, modulesList);

        // 初始化后的所有模块和插件,是一个二无组,转换成不可修改集合二元组.
        // 这是不是意味着:不支持热部署哈.
        this.plugins = Collections.unmodifiableList(pluginsLoaded);

        // Checking expected plugins
        List<String> mandatoryPlugins = MANDATORY_SETTING.get(settings);
        if (mandatoryPlugins.isEmpty() == false) { // false
            Set<String> missingPlugins = new HashSet<>();
            for (String mandatoryPlugin : mandatoryPlugins) {
                if (!pluginsNames.contains(mandatoryPlugin) && !missingPlugins.contains(mandatoryPlugin)) {
                    missingPlugins.add(mandatoryPlugin);
                }
            }
            if (!missingPlugins.isEmpty()) {
                final String message = String.format(
                        Locale.ROOT,
                        "missing mandatory plugins [%s], found plugins [%s]",
                        Strings.collectionToDelimitedString(missingPlugins, ", "),
                        Strings.collectionToDelimitedString(pluginsNames, ", "));
                throw new IllegalStateException(message);
            }
        } // end if

        // 打印日志( loaded module [xxxxx] )
        logPluginInfo(info.getModuleInfos(), "module", logger);
        logPluginInfo(info.getPluginInfos(), "plugin", logger);
    }// end PluginsService 构造器
}// end PluginsService
```
### (4). PluginsService.getModuleBundles
```
// 4.1. 加载modules
// getModuleBundles
// modulesDirectory = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules"
static Set<Bundle> getModuleBundles(Path modulesDirectory) throws IOException {
    // ***********************************************************
    // 4.2. 调用:findBundles
    // ***********************************************************
    return findBundles(modulesDirectory, "module");
}// end 

// 4.2. findBundles声明
private static Set<Bundle> findBundles(final Path directory, String type) throws IOException {
    // path = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules"
    // type = "module"


    final Set<Bundle> bundles = new HashSet<>();

    // ******************************************************
    // 4.3. findPluginDirs 
    // ******************************************************
    for (final Path plugin : findPluginDirs(directory)) {
        // plugin为modules下的目录

        // ****************************************************************
        // 5. PluginsService.readPluginBundle
        // 此处以percolator插件为例:   
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator
        // 读取插件所有的目录,并转换成:Bundle.
        // ****************************************************************
        final Bundle bundle = readPluginBundle(bundles, plugin, type);

        bundles.add(bundle);
    }
    return bundles;
}// end findBundles

// 4.3 findPluginDirs声明
// 加载目录:/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules下所有的目录
public static List<Path> findPluginDirs(final Path rootPath) throws IOException {

    // [
    //   "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator" ,
    //   "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/reindex" , 
    //    "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/analysis-common"
    // ]
    final List<Path> plugins = new ArrayList<>();

    // percolator = [ "percolator","reindex","analysis-common" ]
    final Set<String> seen = new HashSet<>();
    if (Files.exists(rootPath)) {
        try (DirectoryStream<Path> stream = Files.newDirectoryStream(rootPath)) {
            for (Path plugin : stream) {
                if (FileSystemUtils.isDesktopServicesStore(plugin) ||
                    plugin.getFileName().toString().startsWith(".removing-")) {
                    continue;
                }
                if (seen.add(plugin.getFileName().toString()) == false) {
                    throw new IllegalStateException("duplicate plugin: " + plugin);
                }
                plugins.add(plugin);
            }
        }
    }
    return plugins;
}// end findPluginDirs
```
### (5). PluginsService.readPluginBundle
> 读取插件信息

```
private static Bundle readPluginBundle(
            // []空集合
            final Set<Bundle> bundles, 
            // plugin = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator"
            final Path plugin, 
            // type = "module"
            String type) throws IOException {
        LogManager.getLogger(PluginsService.class).trace("--- adding [{}] [{}]", type, plugin.toAbsolutePath());
        
        final PluginInfo info;
        try {
            // *******************************************************
            // 6. 读取插件目录下的配置文件: PluginInfo.readFromProperties
            // *******************************************************
            info = PluginInfo.readFromProperties(plugin);
        } catch (final IOException e) {
            throw new IllegalStateException("Could not load plugin descriptor for " + type +
                                            " directory [" + plugin.getFileName() + "]", e);
        }

        // *****************************************************
        // 7. 根据:PluginInfo对象,构建成:PluginsService$Bundle
        // Bundle包含一个PluginInfo和一堆Jar包.
        // *****************************************************
        final Bundle bundle = new Bundle(info, plugin);
        // 添加到临时集合中.
        if (bundles.add(bundle) == false) {
            throw new IllegalStateException("duplicate " + type + ": " + info);
        }
        return bundle;
    }
```

### (6). PluginInfo.readFromProperties
> 读取插件目录下的配置文件,并转换成:PluginInfo对象.  

```
// 查看插件下的:plugin-descriptor.properties 
lixin-macbook:percolator lixin$ cat /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator/plugin-descriptor.properties 

version=7.1.0
name=percolator
classname=org.elasticsearch.percolator.PercolatorPlugin
java.version=1.8
elasticsearch.version=7.1.0
extended.plugins=
has.native.controller=false
```

> readFromProperties读取插件目录下的配置文件,并转换成:PluginInfo对象.

```
public static PluginInfo readFromProperties(final Path path) throws IOException {
    // ES_PLUGIN_PROPERTIES = "plugin-descriptor.properties";
    // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator/plugin-descriptor.properties
    final Path descriptor = path.resolve(ES_PLUGIN_PROPERTIES);

    final Map<String, String> propsMap;
    {
        final Properties props = new Properties();
        // 读取配置文件
        try (InputStream stream = Files.newInputStream(descriptor)) {
            props.load(stream);
        }

        // 将配置文件内容转换到:propsMap里
        propsMap = props.stringPropertyNames().stream().collect(Collectors.toMap(Function.identity(), props::getProperty));
    }

    // 从配置文件中移除一项:name
    // name = "percolator"
    final String name = propsMap.remove("name");
    if (name == null || name.isEmpty()) {
        throw new IllegalArgumentException(
                "property [name] is missing in [" + descriptor + "]");
    }

    // description = "Percolator module adds capability to index queries and query these queries by specifying documents"
    final String description = propsMap.remove("description");
    if (description == null) {
        throw new IllegalArgumentException(
                "property [description] is missing for plugin [" + name + "]");
    }

    // version = "7.1.0"
    final String version = propsMap.remove("version");
    if (version == null) {
        throw new IllegalArgumentException(
                "property [version] is missing for plugin [" + name + "]");
    }

    // esVersionString = "7.1.0"
    final String esVersionString = propsMap.remove("elasticsearch.version");
    if (esVersionString == null) {
        throw new IllegalArgumentException(
                "property [elasticsearch.version] is missing for plugin [" + name + "]");
    }

    // 对字符串进行解析成:Vesion
    final Version esVersion = Version.fromString(esVersionString);
    // java.version = "1.8"
    final String javaVersionString = propsMap.remove("java.version");
    if (javaVersionString == null) {
        throw new IllegalArgumentException(
                "property [java.version] is missing for plugin [" + name + "]");
    }

    // 检查jdk版本
    JarHell.checkVersionFormat(javaVersionString);

    // classname = "org.elasticsearch.percolator.PercolatorPlugin"
    final String classname = propsMap.remove("classname");
    if (classname == null) {
        throw new IllegalArgumentException(
                "property [classname] is missing for plugin [" + name + "]");
    }

    // extended.plugins = ""
    // 扩展插件以逗号分隔.
    final String extendedString = propsMap.remove("extended.plugins");
    final List<String> extendedPlugins;
    if (extendedString == null) {
        extendedPlugins = Collections.emptyList();
    } else {
        extendedPlugins = Arrays.asList(Strings.delimitedListToStringArray(extendedString, ","));
    }

    // has.native.controller = fasle
    final String hasNativeControllerValue = propsMap.remove("has.native.controller");
    final boolean hasNativeController;
    if (hasNativeControllerValue == null) {
        hasNativeController = false;
    } else {
        switch (hasNativeControllerValue) {
            case "true":
                hasNativeController = true;
                break;
            case "false": // true
                hasNativeController = false;
                break;
            default:
                final String message = String.format(
                        Locale.ROOT,
                        "property [%s] must be [%s], [%s], or unspecified but was [%s]",
                        "has_native_controller",
                        "true",
                        "false",
                        hasNativeControllerValue);
                throw new IllegalArgumentException(message);
        }
    }

    // 针对6.0.0 ~ 6.3.0  删除:   requires.keystore
    if (esVersion.before(Version.V_6_3_0) && esVersion.onOrAfter(Version.V_6_0_0_beta2)) {
        propsMap.remove("requires.keystore");
    }

    // 如果propsMap还有内容(多的配置项),则抛出异常.
    // 看来配置项只能是这几项,否则,抛出异常.所以
    // 这也是为什么从配置文件变成Map,不是从Map中读取数据,而是一项一项的remove.
    if (propsMap.isEmpty() == false) {
        throw new IllegalArgumentException("Unknown properties in plugin descriptor: " + propsMap.keySet());
    }

    // 构建:PluginInfo
    return new PluginInfo(name, description, version, esVersion, javaVersionString,
                            classname, extendedPlugins, hasNativeController);
}
```
### (7). PluginsService$Bundle
> Bundle为一个组件,它包含:PluginInfo和所有的Jar包. 

```
static class Bundle {
    final PluginInfo plugin;
    final Set<URL> urls;

    Bundle(PluginInfo plugin, Path dir) throws IOException {
        // plugin = { "name" : "percolator" , "classname" : "org.elasticsearch.percolator.PercolatorPlugin" }
        // dir = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator"

        this.plugin = Objects.requireNonNull(plugin);


        Set<URL> urls = new LinkedHashSet<>();
        // gather urls for jar files
        // 过滤目录下:/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator
        // 所有jar文件
        try (DirectoryStream<Path> jarStream = Files.newDirectoryStream(dir, "*.jar")) {
            for (Path jar : jarStream) {
                // file:/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/modules/percolator/percolator-client-7.1.0.jar
                URL url = jar.toRealPath().toUri().toURL();
                // 添加到临时集合中
                if (urls.add(url) == false) {
                    throw new IllegalStateException("duplicate codebase: " + url);
                }
            }
        }
        this.urls = Objects.requireNonNull(urls);
    } //end 构造器
}
```
### (8). 总结
> PluginsService加载所有的modules和plugins. 
