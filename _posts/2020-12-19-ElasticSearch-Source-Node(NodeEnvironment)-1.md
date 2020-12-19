---
layout: post
title: 'ElasticSearch 源码Node(NodeEnvironment)-1(四)'
date: 2020-12-19
author: 李新
tags: ElasticSearch源码
---

### (1). 概述
> 在上一节,剖析到:Bootstrap.init方法,在内部,它首先会:new Node,然后,再调用:Node.start();
> 这一节,以及后面几节,都会剖析Node.

### (2). Bootstrap创建Node
```
private void setup(
        boolean addShutdownHook, 
        Environment environment) throws BootstrapException {
    // 创建Node        
    node = new Node(environment) {
            @Override
            protected void validateNodeBeforeAcceptingRequests(
                final BootstrapContext context,
                final BoundTransportAddress boundTransportAddress, List<BootstrapCheck> checks) throws NodeValidationException {
                BootstrapChecks.check(context, boundTransportAddress, checks);
            }
    };// end node
}
```
### (3). Node构造器
```
public Node(Environment environment) {
    this(environment, Collections.emptyList(), true);
}

protected Node(
            final Environment environment, 
            Collection<Class<? extends Plugin>> classpathPlugins, 
            boolean forbidPrivateIndexSettings) {
    // environment = {
    //    "cluster.name" : "elasticsearch",
    //     "http.cors.allow-origin" : "*",
    //     "http.cors.enabled" : "true",
    //     "node.name" : "lixin-macbook.local",
    //     "path.home" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0",
    //     "path.logs" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/logs"   
    // }

    logger = LogManager.getLogger(Node.class);
    // 定义要关闭的资源
    final List<Closeable> resourcesToClose = new ArrayList<>(); 
    boolean success = false;

    try {
        // 基于:environment生新构建出配置
        Settings tmpSettings = 
                 Settings.builder()
                 // 基于:environment
                 .put(environment.settings())
                 // key = "client.type" , value = "node"
                 .put(Client.CLIENT_TYPE_SETTING_S.getKey(), CLIENT_TYPE)
                 .build();

        // 创建 NodeEnviornment
        // 创建索引目录:/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0
        // 写入节点元数据信息
        // 打印日磁盘信息
        // 打印堆信息
        nodeEnvironment = new NodeEnvironment(tmpSettings, environment);

    }catch (IOException ex) {
        throw new ElasticsearchException("failed to bind service", ex);
    } finally {
        if (!success) {
            IOUtils.closeWhileHandlingException(resourcesToClose);
        }// end if
    }// end try
}// end Node
```

### (4). NodeEnvironment
> 固名思意:是节点的环境变量.

```
public NodeEnvironment(
        Settings settings, 
        Environment environment) throws IOException {

    // node.local_storage = true
    // 判断节点是否开启本地存储(本地存储默认是开启的)
    if (!DiscoveryNode.nodeRequiresLocalStorage(settings)) { // false
        nodePaths = null;
        sharedDataPath = null;
        locks = null;
        nodeLockId = -1;
        nodeMetaData = new NodeMetaData(generateNodeId(settings));
        return;
    }

    boolean success = false;
    NodeLock nodeLock = null;

    try {
        // 从环境变量中获得:sharedDataFile
        // sharedDataFile  = null;
        sharedDataPath = environment.sharedDataFile();
        IOException lastException = null;

        // node.max_local_storage_nodes = 1
        // maxLocalStorageNodes
        int maxLocalStorageNodes = MAX_LOCAL_STORAGE_NODES_SETTING.get(settings);

        final AtomicReference<IOException> onCreateDirectoriesException = new AtomicReference<>();


        for (int possibleLockId = 0; 
                 possibleLockId < maxLocalStorageNodes; 
                 possibleLockId++) {
            try {
                // ****************************************************
                // 5. NodeEnvironment.NodeLock
                // ****************************************************
                // possibleLockId = 0
                nodeLock = new NodeLock(
                    possibleLockId, 
                    logger, 
                    environment,
                    dir -> {
                        try {
                            Files.createDirectories(dir);
                        } catch (IOException e) {
                            onCreateDirectoriesException.set(e);
                            throw e;
                        }
                        return true;
                    });

                break;
            } catch (LockObtainFailedException e) {
                // ignore any LockObtainFailedException
            } catch (IOException e) {
                if (onCreateDirectoriesException.get() != null) {
                    throw onCreateDirectoriesException.get();
                }
                lastException = e;
            }
        } // end for

        if (nodeLock == null) { // false
            final String message = String.format(
                Locale.ROOT,
                "failed to obtain node locks, tried [%s] with lock id%s;" +
                    " maybe these locations are not writable or multiple nodes were started without increasing [%s] (was [%d])?",
                Arrays.toString(environment.dataFiles()),
                maxLocalStorageNodes == 1 ? " [0]" : "s [0--" + (maxLocalStorageNodes - 1) + "]",
                MAX_LOCAL_STORAGE_NODES_SETTING.getKey(),
                maxLocalStorageNodes);
            throw new IllegalStateException(message, lastException);
        }// end if

        // 赋值
        // locks = [ "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/node.lock" ]
        this.locks = nodeLock.locks;

        // nodePaths = [ 
        //   { 
        //      "path":"/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0"  ,
        //      "indicesPath" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices",
        //      "fileStore" : "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0"
        //  }
        // ]
        this.nodePaths = nodeLock.nodePaths;

        // nodeLockId = 0
        this.nodeLockId = nodeLock.nodeId;
        
        
        // **********************************************************************
        // 7. 加载或创建节点的元数据(NodeEnvironment.loadOrCreateNodeMetaData)
        // 将元数据写入到:/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/_state
        // **********************************************************************
        this.nodeMetaData = loadOrCreateNodeMetaData(settings, logger, nodePaths);

        if (logger.isDebugEnabled()) {
            logger.debug("using node location [{}], local_lock_id [{}]", nodePaths, nodeLockId);
        }

        // 打印日志,显示:加载磁盘/已使用空间
        // [lixin-macbook.local] using [1] data paths, mounts [[/ (/dev/disk1s5)]], net usable_space [242.6gb], net total_space [465.7gb], types [apfs]
        maybeLogPathDetails();

        // 打印headp信息
        // [lixin-macbook.local] heap size [990.7mb], compressed ordinary object pointers [true]
        maybeLogHeapDetails();

        
        applySegmentInfosTrace(settings);

        // 判断以下目录是否可读写:
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices/DEL7k88cRM-w_7agKny13A/_state
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices/DEL7k88cRM-w_7agKny13A
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices/DEL7k88cRM-w_7agKny13A/0/index
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices/DEL7k88cRM-w_7agKny13A/0/translog
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices/DEL7k88cRM-w_7agKny13A/0/_state
        // /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices/DEL7k88cRM-w_7agKny13A/0

        assertCanWrite();

        // 判断是否为:master或者datanode节点
        if (DiscoveryNode.isMasterNode(settings) || DiscoveryNode.isDataNode(settings)) {  // true
            // 尝试创建临时文件并移动,看是否支持.
            ensureAtomicMoveSupported(nodePaths);
        }

        // 如果不是data节点
        if (DiscoveryNode.isDataNode(settings) == false) { // false
            if (DiscoveryNode.isMasterNode(settings) == false) {
                ensureNoIndexMetaData(nodePaths);
            }
            ensureNoShardData(nodePaths);
        }
        // 返回成功
        success = true;
    } finally {
        if (success == false) { // false
            close();
        }
    }
} // end NodeEnvironment 构造器
```
### (5). NodeEnvironment.NodeLock
> 1. 创建:data/nodes/0目录.   
> 2. 创建:data/nodes/0/node.lock锁文件.   
> 3. 调用NodeEnvironment$NodePath创建:indices目录.    

```
public final class NodeEnvironment  implements Closeable {
    // 内部对象
    public static class NodeLock implements Releasable {
        private final int nodeId;
        private final Lock[] locks;
        private final NodePath[] nodePaths;

        public NodeLock(
                        final int nodeId, 
                        final Logger logger,
                        final Environment environment,
                        // 回调函数
                        final CheckedFunction<Path, Boolean, IOException> pathFunction
                        ) throws IOException {
            // nodeId = 0
            this.nodeId = nodeId;
            // environment.dataFiles().length = 1 
            nodePaths = new NodePath[environment.dataFiles().length];
            // 创建锁.
            locks = new Lock[nodePaths.length];
            try {
                
                final Path[] dataPaths = environment.dataFiles();
                // dataPaths=["/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data"];

                for (int dirIndex = 0; dirIndex < dataPaths.length; dirIndex++) {
                    // dataDir = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data"
                    Path dataDir = dataPaths[dirIndex];

                    // 创建目录:nodes/${dirIndex}
                    // dir = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0"
                    Path dir = resolveNodePath(dataDir, nodeId);

                    // 调用回调函数:创建目录
                    if (pathFunction.apply(dir) == false) { // 创建失败,则跳过这次循环.
                        continue;
                    }

                    // *********************************************************************************
                    // 调用Lucene打开(创建)索引:    
                    // *********************************************************************************
                    // dir = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0"
                    try (Directory luceneDir = FSDirectory.open(dir, NativeFSLockFactory.INSTANCE)) {

                        logger.trace("obtaining node lock on {} ...", dir.toAbsolutePath());
                        
                        // *********************************************************************************
                        // NODE_LOCK_FILENAME = "node.lock"
                        // 创建加锁文件: "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/node.lock"
                        // *********************************************************************************
                        locks[dirIndex] = luceneDir.obtainLock(NODE_LOCK_FILENAME);

                        // *********************************************************************************
                        // 6. 创建:NodeEnvironment$NodePath
                        // dir = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0"
                        // *********************************************************************************
                        nodePaths[dirIndex] = new NodePath(dir);
                    } catch (IOException e) {
                        logger.trace(() -> new ParameterizedMessage(
                            "failed to obtain node lock on {}", dir.toAbsolutePath()), e);
                        // release all the ones that were obtained up until now
                        throw (e instanceof LockObtainFailedException ? e
                            : new IOException("failed to obtain lock on " + dir.toAbsolutePath(), e));
                    }
                }
            } catch (IOException e) {
                close();
                throw e;
            }
        }// end NodeLock构造器
    }
} // end NodeEnvironment
```
### (6). NodeEnvironment$NodePath
>  1. 创建:indices目录.

```
public final class NodeEnvironment  implements Closeable {
    // 内部类
    public static class NodePath {
        /* ${data.paths}/nodes/{node.id} */
        public final Path path;
        /* ${data.paths}/nodes/{node.id}/indices */
        public final Path indicesPath;
        /** Cached FileStore from path */
        public final FileStore fileStore;

        public final int majorDeviceNumber;
        public final int minorDeviceNumber;

        public NodePath(Path path) throws IOException {
            // path = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0"
            this.path = path;

            // indicesPath = "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices"
            this.indicesPath = path.resolve(INDICES_FOLDER);

            this.fileStore = Environment.getFileStore(path);
            if (fileStore.supportsFileAttributeView("lucene")) {
                this.majorDeviceNumber = (int) fileStore.getAttribute("lucene:major_device_number");
                this.minorDeviceNumber = (int) fileStore.getAttribute("lucene:minor_device_number");
            } else {
                this.majorDeviceNumber = -1;
                this.minorDeviceNumber = -1;
            }

        }// end NodePath 构造器
    } // end NodePath
} //end NodeEnvironment
```
### (7). NodeEnvironment.loadOrCreateNodeMetaData
```
private static NodeMetaData loadOrCreateNodeMetaData(
        Settings settings, Logger logger,
        NodePath... nodePaths) throws IOException {

    // paths = [ "/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0" ]         
    final Path[] paths = Arrays.stream(nodePaths).map(np -> np.path).toArray(Path[]::new);

    // 加载元数据信息.
    

    // 1. 读取:/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/_state目录下:node-*.st文件
    // 我的目录对应文件如下: /Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/_state/node-13.st
    // 2. 获得:node-13.st中的数字:13
    // 3. 读取: node-13.st的内容.
    NodeMetaData metaData = NodeMetaData.FORMAT.loadLatestState(logger, NamedXContentRegistry.EMPTY, paths);

    // 第一次则会创建
    if (metaData == null) {  // false
        metaData = new NodeMetaData(generateNodeId(settings));
    }
    // we write again to make sure all paths have the latest state file
    NodeMetaData.FORMAT.writeAndCleanup(metaData, paths);
    return metaData;
}
```
### (8). 总结
> 1. 创建数据目录(/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/).   
> 2. 创建数据目录相应的锁文件(/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/node.lock).    
> 2. 创建indices目录(/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/indices).   
> 3. 创建元数据目录(/Users/lixin/Developer/elastic-search/elasticsearch-7.1.0/data/nodes/0/_state).  
> 4. 打印磁盘信息/堆信息.  
> 5. 尝试文件的删除/移动(确保能做索引合并).  