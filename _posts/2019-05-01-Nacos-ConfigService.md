---
layout: post
title: 'Nacos源码之:ConfigService(七)'
date: 2019-05-01
author: 李新
tags: Nacos
---

### (1). 概述
在这一小节,主要剖析:ConfigService,它是由:NacosFactory构建出来的.  

### (2). NacosFactory
```
public class NacosFactory {

    public static ConfigService createConfigService(Properties properties) throws NacosException {
		// ***********************************************************
		// 委托给了:ConfigFactory
		// ***********************************************************
        return ConfigFactory.createConfigService(properties);
    }
}
```
### (3). ConfigFactory
```
public class ConfigFactory {

    public static ConfigService createConfigService(Properties properties) throws NacosException {
        try {
			// ************************************************************
			// 通过反射,加载并且new
			// ************************************************************
            Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
            Constructor constructor = driverImplClass.getConstructor(Properties.class);
            ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
            return vendorImpl;
        } catch (Throwable e) {
            throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
        }
    }
}
```
### (4). ConfigService
```
public class NacosConfigService implements ConfigService {
	private static final long POST_TIMEOUT = 3000L;
	private static final String EMPTY = "";
	
	/**
	 * http agent
	 */
	private HttpAgent agent;
	/**
	 * longpolling
	 */
	private ClientWorker worker;
	private String namespace;
	private String encode;
	
	// ******************************************************************************
	// 责任链管理者
	// ******************************************************************************
	private ConfigFilterChainManager configFilterChainManager = new ConfigFilterChainManager();

	// ********************************************************
	// 1. 构建器初始化
	//    设置encode
	//    获得
	// ********************************************************
	public NacosConfigService(Properties properties) throws NacosException {
		// properties = {secretKey=, namespace=45a6112b-a866-4e92-a8d6-9e440bcdebbe, fileExtension=properties, username=, enableRemoteSyncConfig=false, configLongPollTimeout=, configRetryTime=, encode=, serverAddr=127.0.0.1:8848, maxRetry=, group=erp:test-provider, clusterName=, password=, accessKey=, endpoint=}
		String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
        if (StringUtils.isBlank(encodeTmp)) {
            encode = Constants.ENCODE;
        } else {
            encode = encodeTmp.trim();
        }
		// 获得namespace
        initNamespace(properties);
		
		// ********************************************************
		// 2. 最终请求Nacos会通过MetricsHttpAgent去请求的
		// ********************************************************
        agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
        agent.start();
		
		// 3. 创建:ClientWorker(会委派给:MetricsHttpAgent)
        worker = new ClientWorker(agent, configFilterChainManager, properties);
    }
	
	
	// ***********************************************
	// 4. 获取配置信息
	// ***********************************************
	private String getConfigInner(
	     // 45a6112b-a866-4e92-a8d6-9e440bcdebbe
	     String tenant, 
		 // test-provider
		 String dataId, 
		 // erp:test-provider
		 String group, 
		 // 3000
		 long timeoutMs) throws NacosException {
		
		// 如果group为null,则,group=DEFAULT_GROUP
		group = null2defaultGroup(group);
		// 检查参数
		ParamUtils.checkKeyParam(dataId, group);
		
		// 返回结果
		ConfigResponse cr = new ConfigResponse();
		cr.setDataId(dataId);
		cr.setTenant(tenant);
		cr.setGroup(group);

		// 优先使用本地配置
		String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
		if (content != null) {
			LOGGER.warn("[{}] [get-config] get failover ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
				dataId, group, tenant, ContentUtils.truncateContent(content));
			cr.setContent(content);
			configFilterChainManager.doFilter(null, cr);
			content = cr.getContent();
			return content;
		}

		try {
			// ******************************************************************
			// 5. 委托给:ClientWorker
			// ******************************************************************
			String[] ct = worker.getServerConfig(dataId, group, tenant, timeoutMs);
			// 并获得数组的第0个元素,赋值给:ConfigResponse
			cr.setContent(ct[0]);
			
			// 调用责任链,对:ConfigResponse进行处理
			configFilterChainManager.doFilter(null, cr);
			
			// 返回被责任链处理过之后的内容
			content = cr.getContent();
			return content;
		} catch (NacosException ioe) {
			if (NacosException.NO_RIGHT == ioe.getErrCode()) {
				throw ioe;
			}
			LOGGER.warn("[{}] [get-config] get from server error, dataId={}, group={}, tenant={}, msg={}",
				agent.getName(), dataId, group, tenant, ioe.toString());
		}
		
		// 当抛出的异常不是NacosException时,获取本地的快照.
		LOGGER.warn("[{}] [get-config] get snapshot ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
			dataId, group, tenant, ContentUtils.truncateContent(content));
		content = LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
		cr.setContent(content);
		configFilterChainManager.doFilter(null, cr);
		content = cr.getContent();
		return content;
	} // end getConfigInner
}	
```
### (5). ClientWorker
```
public class ClientWorker {
	
	public String[] getServerConfig(String dataId, String group, String tenant, long readTimeout)
        throws NacosException {
		// 创建返回结果	
        String[] ct = new String[2];
		// 又做了一次group检查
        if (StringUtils.isBlank(group)) {
            group = Constants.DEFAULT_GROUP;
        }
		

        HttpResult result = null;
        try {
			// 构建请求参数
            List<String> params = null;
            if (StringUtils.isBlank(tenant)) {
                params = new ArrayList<String>(Arrays.asList("dataId", dataId, "group", group));
            } else {
                params = new ArrayList<String>(Arrays.asList("dataId", dataId, "group", group, "tenant", tenant));
            }
			
			// **************************************************************
			// 委托给:ServerHttpAgent,最终还是会委托给:HttpSimpleClient,这是Nacos自己基于HttpURLConnection写的一个HTTP工具
			// **************************************************************
            result = agent.httpGet(Constants.CONFIG_CONTROLLER_PATH, null, params, agent.getEncode(), readTimeout);
        } catch (IOException e) {  
			// 异常处理
            String message = String.format(
                "[%s] [sub-server] get server config exception, dataId=%s, group=%s, tenant=%s", agent.getName(),
                dataId, group, tenant);
            LOGGER.error(message, e);
            throw new NacosException(NacosException.SERVER_ERROR, e);
        }

        switch (result.code) {
            case HttpURLConnection.HTTP_OK:  // 成功的情况下,把结果放在内存快照里.
                LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, result.content);
				// ct[0] 为返回的结果内容
                ct[0] = result.content;
                // ct[1] 为Config-Type(properties)
				if (result.headers.containsKey(CONFIG_TYPE)) {
                    ct[1] = result.headers.get(CONFIG_TYPE).get(0);
                } else {
                    ct[1] = ConfigType.TEXT.getType();
                }
                return ct;
            case HttpURLConnection.HTTP_NOT_FOUND:
                LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, null);
                return ct;
            case HttpURLConnection.HTTP_CONFLICT: {
                LOGGER.error(
                    "[{}] [sub-server-error] get server config being modified concurrently, dataId={}, group={}, "
                        + "tenant={}", agent.getName(), dataId, group, tenant);
                throw new NacosException(NacosException.CONFLICT,
                    "data being modified, dataId=" + dataId + ",group=" + group + ",tenant=" + tenant);
            }
            case HttpURLConnection.HTTP_FORBIDDEN: {
                LOGGER.error("[{}] [sub-server-error] no right, dataId={}, group={}, tenant={}", agent.getName(), dataId,
                    group, tenant);
                throw new NacosException(result.code, result.content);
            }
            default: {
                LOGGER.error("[{}] [sub-server-error]  dataId={}, group={}, tenant={}, code={}", agent.getName(), dataId,
                    group, tenant, result.code);
                throw new NacosException(result.code,
                    "http error, code=" + result.code + ",dataId=" + dataId + ",group=" + group + ",tenant=" + tenant);
            }
        }
    }
}
```
### (6). 启动时发起的HTTP请求
```
# 1. get
http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=test-provider.properties&group=erp%3Atest-provider&tenant=45a6112b-a866-4e92-a8d6-9e440bcdebbe
	useLocalCache=true
	config-type=properties

# 2. post
http://127.0.0.1:8848/nacos/v1/cs/configs/listener
```
### (7). 总结
好像没什么要总结的,都比较简单.  