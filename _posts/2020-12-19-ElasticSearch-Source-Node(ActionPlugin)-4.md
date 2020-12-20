---
layout: post
title: 'ElasticSearch 源码Node(ActionModule)(四)'
date: 2020-12-20
author: 李新
tags: ElasticSearch源码
---

### (1). 概述
>  这一节,主要剖析ActionModule.这个模块主要是用来接受请求的.

### (2). Node构造器
> 1. 创建: ActionModule.    
> 2. 调用: ActionModule.initRestHandlers会初始化所有的action(调用:RestController.registerHandler),并加载ActionPlugin中定义的action. 

```
ActionModule actionModule = 
        new ActionModule(
            false, 
            settings, 
            clusterModule.getIndexNameExpressionResolver(),
            settingsModule.getIndexScopedSettings(), settingsModule.getClusterSettings(), settingsModule.getSettingsFilter(),
            threadPool, 
            // ************************************************
            // 从插件中心,过滤出实现了:ActionPlugin接口的插件.
            // ************************************************
            pluginsService.filterPlugins(ActionPlugin.class), 
            client, 
            circuitBreakerService, 
            usageService
        ); 

// TODO ... ...

// ***************************************************************
// 
// ***************************************************************
actionModule.initRestHandlers(() -> clusterService.state().nodes());
```
### (3). ActionModule构造器
```
public ActionModule(
        boolean transportClient, 
        Settings settings, 
        IndexNameExpressionResolver indexNameExpressionResolver
        IndexScopedSettings indexScopedSettings, 
        ClusterSettings clusterSettings, 
        SettingsFilter settingsFilter,
        ThreadPool threadPool, 
        List<ActionPlugin> actionPlugins, 
        NodeClient nodeClient,
        CircuitBreakerService circuitBreakerService, 
        UsageService usageService) {
     
    this.transportClient = transportClient;
    this.settings = settings;
    this.indexNameExpressionResolver = indexNameExpressionResolver;
    this.indexScopedSettings = indexScopedSettings;
    this.clusterSettings = clusterSettings;
    this.settingsFilter = settingsFilter;
    this.actionPlugins = actionPlugins;

    actions = setupActions(actionPlugins);
    actionFilters = setupActionFilters(actionPlugins);
    autoCreateIndex = transportClient ? null : new AutoCreateIndex(settings, clusterSettings, indexNameExpressionResolver);

    destructiveOperations = new DestructiveOperations(settings, clusterSettings);

    Set<String> headers = Stream.concat(
        actionPlugins.stream().flatMap(p -> p.getRestHeaders().stream()),
        Stream.of(Task.X_OPAQUE_ID)
    ).collect(Collectors.toSet());
    UnaryOperator<RestHandler> restWrapper = null;

    for (ActionPlugin plugin : actionPlugins) {
        // 调用所有:
        UnaryOperator<RestHandler> newRestWrapper = plugin.getRestHandlerWrapper(threadPool.getThreadContext());
        if (newRestWrapper != null) {
            logger.debug("Using REST wrapper from plugin " + plugin.getClass().getName());
            if (restWrapper != null) {
                throw new IllegalArgumentException("Cannot have more than one plugin implementing a REST wrapper");
            }
            restWrapper = newRestWrapper;
        }
    }// end for

    mappingRequestValidators = new TransportPutMappingAction.RequestValidators(
        actionPlugins.stream().flatMap(p -> p.mappingRequestValidators().stream()).collect(Collectors.toList())
    ); 

    if (transportClient) { // flase
        restController = null;
    } else {
        // 创建RestController
        restController = new RestController(
                headers, 
                restWrapper, 
                nodeClient, 
                circuitBreakerService, 
                usageService);
    }// end else
}// end ActionModule
```
### (4). ActionModule.initRestHandlers
```
public void initRestHandlers(Supplier<DiscoveryNodes> nodesInCluster) {
    List<AbstractCatAction> catActions = new ArrayList<>();
    Consumer<RestHandler> registerHandler = a -> {
        if (a instanceof AbstractCatAction) {
            catActions.add((AbstractCatAction) a);
        }
    };

    // ************************************************************************************
    // RestAddVotingConfigExclusionAction(Settings,RestController)
    // 创建:RestAddVotingConfigExclusionAction,在其构造器内部,实际是调用:RestController.registerHandler
    // ************************************************************************************
    registerHandler.accept(new RestAddVotingConfigExclusionAction(settings, restController));
    registerHandler.accept(new RestClearVotingConfigExclusionsAction(settings, restController));
    registerHandler.accept(new RestMainAction(settings, restController));
    registerHandler.accept(new RestNodesInfoAction(settings, restController, settingsFilter));
    registerHandler.accept(new RestRemoteClusterInfoAction(settings, restController));
    registerHandler.accept(new RestNodesStatsAction(settings, restController));
    registerHandler.accept(new RestNodesUsageAction(settings, restController));
    registerHandler.accept(new RestNodesHotThreadsAction(settings, restController));
    registerHandler.accept(new RestClusterAllocationExplainAction(settings, restController));
    registerHandler.accept(new RestClusterStatsAction(settings, restController));
    registerHandler.accept(new RestClusterStateAction(settings, restController, settingsFilter));
    registerHandler.accept(new RestClusterHealthAction(settings, restController));
    registerHandler.accept(new RestClusterUpdateSettingsAction(settings, restController));
    registerHandler.accept(new RestClusterGetSettingsAction(settings, restController, clusterSettings, settingsFilter));
    registerHandler.accept(new RestClusterRerouteAction(settings, restController, settingsFilter));
    registerHandler.accept(new RestClusterSearchShardsAction(settings, restController));
    registerHandler.accept(new RestPendingClusterTasksAction(settings, restController));
    registerHandler.accept(new RestPutRepositoryAction(settings, restController));
    registerHandler.accept(new RestGetRepositoriesAction(settings, restController, settingsFilter));
    registerHandler.accept(new RestDeleteRepositoryAction(settings, restController));
    registerHandler.accept(new RestVerifyRepositoryAction(settings, restController));
    registerHandler.accept(new RestGetSnapshotsAction(settings, restController));
    registerHandler.accept(new RestCreateSnapshotAction(settings, restController));
    registerHandler.accept(new RestRestoreSnapshotAction(settings, restController));
    registerHandler.accept(new RestDeleteSnapshotAction(settings, restController));
    registerHandler.accept(new RestSnapshotsStatusAction(settings, restController));
    registerHandler.accept(new RestGetIndicesAction(settings, restController));
    registerHandler.accept(new RestIndicesStatsAction(settings, restController));
    registerHandler.accept(new RestIndicesSegmentsAction(settings, restController));
    registerHandler.accept(new RestIndicesShardStoresAction(settings, restController));
    registerHandler.accept(new RestGetAliasesAction(settings, restController));
    registerHandler.accept(new RestIndexDeleteAliasesAction(settings, restController));
    registerHandler.accept(new RestIndexPutAliasAction(settings, restController));
    registerHandler.accept(new RestIndicesAliasesAction(settings, restController));
    registerHandler.accept(new RestCreateIndexAction(settings, restController));
    registerHandler.accept(new RestResizeHandler.RestShrinkIndexAction(settings, restController));
    registerHandler.accept(new RestResizeHandler.RestSplitIndexAction(settings, restController));
    registerHandler.accept(new RestRolloverIndexAction(settings, restController));
    registerHandler.accept(new RestDeleteIndexAction(settings, restController));
    registerHandler.accept(new RestCloseIndexAction(settings, restController));
    registerHandler.accept(new RestOpenIndexAction(settings, restController));

    registerHandler.accept(new RestUpdateSettingsAction(settings, restController));
    registerHandler.accept(new RestGetSettingsAction(settings, restController));

    registerHandler.accept(new RestAnalyzeAction(settings, restController));
    registerHandler.accept(new RestGetIndexTemplateAction(settings, restController));
    registerHandler.accept(new RestPutIndexTemplateAction(settings, restController));
    registerHandler.accept(new RestDeleteIndexTemplateAction(settings, restController));

    registerHandler.accept(new RestPutMappingAction(settings, restController));
    registerHandler.accept(new RestGetMappingAction(settings, restController));
    registerHandler.accept(new RestGetFieldMappingAction(settings, restController));

    registerHandler.accept(new RestRefreshAction(settings, restController));
    registerHandler.accept(new RestFlushAction(settings, restController));
    registerHandler.accept(new RestSyncedFlushAction(settings, restController));
    registerHandler.accept(new RestForceMergeAction(settings, restController));
    registerHandler.accept(new RestUpgradeAction(settings, restController));
    registerHandler.accept(new RestUpgradeStatusAction(settings, restController));
    registerHandler.accept(new RestClearIndicesCacheAction(settings, restController));

    registerHandler.accept(new RestIndexAction(settings, restController));
    registerHandler.accept(new RestGetAction(settings, restController));
    registerHandler.accept(new RestGetSourceAction(settings, restController));
    registerHandler.accept(new RestMultiGetAction(settings, restController));
    registerHandler.accept(new RestDeleteAction(settings, restController));
    registerHandler.accept(new RestCountAction(settings, restController));
    registerHandler.accept(new RestTermVectorsAction(settings, restController));
    registerHandler.accept(new RestMultiTermVectorsAction(settings, restController));
    registerHandler.accept(new RestBulkAction(settings, restController));
    registerHandler.accept(new RestUpdateAction(settings, restController));

    registerHandler.accept(new RestSearchAction(settings, restController));
    registerHandler.accept(new RestSearchScrollAction(settings, restController));
    registerHandler.accept(new RestClearScrollAction(settings, restController));
    registerHandler.accept(new RestMultiSearchAction(settings, restController));

    registerHandler.accept(new RestValidateQueryAction(settings, restController));

    registerHandler.accept(new RestExplainAction(settings, restController));

    registerHandler.accept(new RestRecoveryAction(settings, restController));

    registerHandler.accept(new RestReloadSecureSettingsAction(settings, restController));

    // Scripts API
    registerHandler.accept(new RestGetStoredScriptAction(settings, restController));
    registerHandler.accept(new RestPutStoredScriptAction(settings, restController));
    registerHandler.accept(new RestDeleteStoredScriptAction(settings, restController));

    registerHandler.accept(new RestFieldCapabilitiesAction(settings, restController));

    // Tasks API
    registerHandler.accept(new RestListTasksAction(settings, restController, nodesInCluster));
    registerHandler.accept(new RestGetTaskAction(settings, restController));
    registerHandler.accept(new RestCancelTasksAction(settings, restController, nodesInCluster));

    // Ingest API
    registerHandler.accept(new RestPutPipelineAction(settings, restController));
    registerHandler.accept(new RestGetPipelineAction(settings, restController));
    registerHandler.accept(new RestDeletePipelineAction(settings, restController));
    registerHandler.accept(new RestSimulatePipelineAction(settings, restController));

    // CAT API
    registerHandler.accept(new RestAllocationAction(settings, restController));
    registerHandler.accept(new RestShardsAction(settings, restController));
    registerHandler.accept(new RestMasterAction(settings, restController));
    registerHandler.accept(new RestNodesAction(settings, restController));
    registerHandler.accept(new RestTasksAction(settings, restController, nodesInCluster));
    registerHandler.accept(new RestIndicesAction(settings, restController, indexNameExpressionResolver));
    registerHandler.accept(new RestSegmentsAction(settings, restController));
    // Fully qualified to prevent interference with rest.action.count.RestCountAction
    registerHandler.accept(new org.elasticsearch.rest.action.cat.RestCountAction(settings, restController));
    // Fully qualified to prevent interference with rest.action.indices.RestRecoveryAction
    registerHandler.accept(new org.elasticsearch.rest.action.cat.RestRecoveryAction(settings, restController));
    registerHandler.accept(new RestHealthAction(settings, restController));
    registerHandler.accept(new org.elasticsearch.rest.action.cat.RestPendingClusterTasksAction(settings, restController));
    registerHandler.accept(new RestAliasAction(settings, restController));
    registerHandler.accept(new RestThreadPoolAction(settings, restController));
    registerHandler.accept(new RestPluginsAction(settings, restController));
    registerHandler.accept(new RestFielddataAction(settings, restController));
    registerHandler.accept(new RestNodeAttrsAction(settings, restController));
    registerHandler.accept(new RestRepositoriesAction(settings, restController));
    registerHandler.accept(new RestSnapshotAction(settings, restController));
    registerHandler.accept(new RestTemplatesAction(settings, restController));

    /********************************************************************************
    // 加载:ActionPlugin接口的实现类.
    // 实际是调用:RestController.registerHandler
    /********************************************************************************
    for (ActionPlugin plugin : actionPlugins) {
        for (RestHandler handler : plugin.getRestHandlers(settings, restController, clusterSettings, indexScopedSettings,
                settingsFilter, indexNameExpressionResolver, nodesInCluster)) {
            registerHandler.accept(handler);
        }
    }

    registerHandler.accept(new RestCatAction(settings, restController, catActions));
}// end initRestHandlers
```
### (5). RestController.registerHandler
> 注册Handler(以RestAddVotingConfigExclusionAction为例)

```
public void registerHandler(
        // POST
        RestRequest.Method method, 
        // /_cluster/voting_config_exclusions/{node_name}
        String path, 
        // RestAddVotingConfigExclusionAction
        RestHandler handler) {

    if (handler instanceof BaseRestHandler) { // true
        usageService.addRestHandler((BaseRestHandler) handler);
    }


    // 保存或者更新
    handlers.insertOrUpdate(
            path, 
            new MethodHandlers(path, handler, method), 
            (mHandlers, newMHandler) -> {
              return mHandlers.addMethods(handler, method);
            }
    );
} // end registerHandler
```


### (6). 自定义ActionPlugin步骤
> 1. 创建(TestActionPlugin)插件,继承:Plugin并实现:ActionPlugin.    
> 2. 插件(TestActionPlugin)需要重写:getActions和getRestHandlers方法.     
> 3. 编写(TestAction)Action,可以继承于:BaseRestHandler.    
> 4. TestAction可以获取到:RestController,那么,就可以向RestController.registerHandler("POST","/hello",this).    
> 5. 创建配置(plugin-descriptor.properties)文件,文件内容不能多也不能少.   

### (7). 总结
> ActionModule用来定义接受请求.RestController固名思意:就是用来接受Rest请求的.   
