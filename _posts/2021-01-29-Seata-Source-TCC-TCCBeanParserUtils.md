---
layout: post
title: 'Seata  TCC全局事务之TCCBeanParserUtils(二)'
date: 2021-01-29
author: 李新
tags: Seata-TCC源码
---

### (1). 查看下DefaultRemotingParser的类图

!["DefaultRemotingParser类图"](/assets/seata/imgs/seata-DefaultRemotingParser-Class-Diagram.jpg)

### (2). TCCBeanParserUtils.isTccAutoProxy
```
public static boolean isTccAutoProxy(Object bean, String beanName, ApplicationContext applicationContext) {
	// 判断bean是否为远程的bean
	// 2. TCCBeanParserUtils.parserRemotingServiceInfo
	boolean isRemotingBean = parserRemotingServiceInfo(bean, beanName);
	
	// 在上一步,如果bean有找到相应的:RemotingParser对象,那么把bean解析成:RemotingDesc
	// 并设置在缓存里.
	//get RemotingBean description
	RemotingDesc remotingDesc = DefaultRemotingParser.get().getRemotingBeanDesc(beanName);
	
	//is remoting bean
	if (isRemotingBean) {  
		// 当是远程的Bean时,判断是否为:@LocalTCC
		if (remotingDesc != null && remotingDesc.getProtocol() == Protocols.IN_JVM) {
			// 针对@LocalTCC的处理
			//LocalTCC
			return isTccProxyTargetBean(remotingDesc);
		} else {
			// sofa:reference / dubbo:reference, factory bean
			return false;
		}
	} else {
		if (remotingDesc == null) {
			//check FactoryBean
			if (isRemotingFactoryBean(bean, beanName, applicationContext)) {
				remotingDesc = DefaultRemotingParser.get().getRemotingBeanDesc(beanName);
				return isTccProxyTargetBean(remotingDesc);
			} else {
				return false;
			}
		} else {
			return isTccProxyTargetBean(remotingDesc);
		}
	}
}
```
### (3). TCCBeanParserUtils.parserRemotingServiceInfo
```
protected static boolean parserRemotingServiceInfo(Object bean, String beanName) {

	// 通过SPI加载(RemotingParser)的实现.
	// 判断是否为远程对象
	// DubboRemotingParser的判断是通过动态代理之后,对象的名称是否为(ReferenceBean/ServiceBean)
	// LocalTCCRemotingParser的判断是接口上是否有标注指定的注解@LocalTCC
	// SofaRpcRemotingParser的判断是通过动态代理之后,对象的名称是否为(ReferenceFactoryBean/ServiceFactoryBean)
	// HSFRemotingParser的判断是通过动态代理之后,对象的名称是否为(HSFApiConsumerBean/HSFApiProviderBean/HSFSpringConsumerBean/HSFSpringProviderBean)
	// **********************************************************************
	// 4. 调用:DefaultRemotingParser.isRemoting,获得对应的:RemotingParser
	// **********************************************************************
	RemotingParser remotingParser = DefaultRemotingParser.get().isRemoting(bean, beanName);
	
	
	// 如果判断是:远程对象
	if (remotingParser != null) {  // 代表解析器已经找到,也就代表是需要进行代理的对象
		// ****************************************************************
		// 6. 调用:DefaultRemotingParser.parserRemotingServiceInfo
		//    把bean(对象)进行解析(获取相应的注解信息等),解析成业务模型:RemotingDesc
		// ****************************************************************
		return DefaultRemotingParser.get().parserRemotingServiceInfo(bean, beanName, remotingParser) != null;
	}
	return false;
}// end 

```
### (4). DefaultRemotingParser.isRemoting
```
public RemotingParser isRemoting(Object bean, String beanName) {
	// allRemotingParsers是通过SPI加载的(RemotingParser)
	// allRemotingParsers = [
	// io.seata.rm.tcc.remoting.parser.DubboRemotingParser@4567e53d, 
	// io.seata.rm.tcc.remoting.parser.LocalTCCRemotingParser@7351a16e, 
	// io.seata.rm.tcc.remoting.parser.SofaRpcRemotingParser@5bb7643d, 
	// io.seata.rm.tcc.remoting.parser.HSFRemotingParser@3ac04654
	// ]
	for (RemotingParser remotingParser : allRemotingParsers) {
		// *********************************************************************
		// 5. 我这里以:io.seata.rm.tcc.remoting.parser.DubboRemotingParser.isRemoting为例
		// *********************************************************************
		if (remotingParser.isRemoting(bean, beanName)) {
			return remotingParser;
		}
	}
	return null;
}
```
### (5). DubboRemotingParser.isRemoting
```
// AbstractedRemotingParser
public abstract class AbstractedRemotingParser implements RemotingParser {
    @Override
    public boolean isRemoting(Object bean, String beanName) throws FrameworkException {
		// 判断是否为:Service/Refrence
        return isReference(bean, beanName) || isService(bean, beanName);
    }
} // end AbstractedRemotingParser


// DubboRemotingParser 
public class DubboRemotingParser extends AbstractedRemotingParser {

	// 判断bean的名称是否为:
	// com.alibaba.dubbo.config.spring.ReferenceBean
	// org.apache.dubbo.config.spring.ReferenceBean
    @Override
    public boolean isReference(Object bean, String beanName) throws FrameworkException {
        Class<?> c = bean.getClass();
        return "com.alibaba.dubbo.config.spring.ReferenceBean".equals(c.getName())
            || "org.apache.dubbo.config.spring.ReferenceBean".equals(c.getName());
    }

	// 判断bean的名称是否为:
	// com.alibaba.dubbo.config.spring.ServiceBean
	// org.apache.dubbo.config.spring.ServiceBean
    @Override
    public boolean isService(Object bean, String beanName) throws FrameworkException {
        Class<?> c = bean.getClass();
        return "com.alibaba.dubbo.config.spring.ServiceBean".equals(c.getName())
            || "org.apache.dubbo.config.spring.ServiceBean".equals(c.getName());
    }
	
	// *******************************************************************
	// 5.1 把Bean解析成业务模型(RemotingDesc)对象
	// *******************************************************************
	public RemotingDesc getServiceDesc(Object bean, String beanName) throws FrameworkException {
		// 再次检查一下是不是符合dubbo规范的对象
		if (!this.isRemoting(bean, beanName)) {
			return null;
		}
		
		try {
			// 构建业务模型对象:RemotingDesc
			RemotingDesc serviceBeanDesc = new RemotingDesc();
			// 以下是通过反射获得相应的内容,设置到业务模型(RemotingDesc)里
			Class<?> interfaceClass = (Class<?>)ReflectionUtil.invokeMethod(bean, "getInterfaceClass");
			String interfaceClassName = (String)ReflectionUtil.getFieldValue(bean, "interfaceName");
			String version = (String)ReflectionUtil.invokeMethod(bean, "getVersion");
			String group = (String)ReflectionUtil.invokeMethod(bean, "getGroup");
			serviceBeanDesc.setInterfaceClass(interfaceClass);
			serviceBeanDesc.setInterfaceClassName(interfaceClassName);
			serviceBeanDesc.setUniqueId(version);
			serviceBeanDesc.setGroup(group);
			serviceBeanDesc.setProtocol(Protocols.DUBBO);
			if (isService(bean, beanName)) {
				Object targetBean = ReflectionUtil.getFieldValue(bean, "ref");
				serviceBeanDesc.setTargetBean(targetBean);
			}
			return serviceBeanDesc;
		} catch (Throwable t) {
			throw new FrameworkException(t);
		}
	} // end getServiceDesc
	
} // end DubboRemotingParser 
```
### (6). DefaultRemotingParser.parserRemotingServiceInfo
```
// domain
protected static Map<String, RemotingDesc> remotingServiceMap = new ConcurrentHashMap<>();

// methods
public RemotingDesc parserRemotingServiceInfo(Object bean, String beanName, RemotingParser remotingParser) {
	
	// ************************************************************
	// 5.1 委托给:RemotingParser.getServiceDesc解析成业务模型:RemotingDesc
	//   我这里以:DubboRemotingParser为例
	// ************************************************************
	RemotingDesc remotingBeanDesc = remotingParser.getServiceDesc(bean, beanName);
	
	if (remotingBeanDesc == null) {
		return null;
	}

	// 设置到缓存中,后面还需要用到
	remotingServiceMap.put(beanName, remotingBeanDesc);

	//获得接口上的信息.
	Class<?> interfaceClass = remotingBeanDesc.getInterfaceClass();
	Method[] methods = interfaceClass.getMethods();
	if (remotingParser.isService(bean, beanName)) {
		try {
			//service bean, registry resource
			Object targetBean = remotingBeanDesc.getTargetBean();
			for (Method m : methods) {
				// 获得接口上的所有方法,查看是否有标注的注解:@TwoPhaseBusinessAction
				TwoPhaseBusinessAction twoPhaseBusinessAction = m.getAnnotation(TwoPhaseBusinessAction.class);
				if (twoPhaseBusinessAction != null) {
					// **********************************************************************
					// 每一个注解:@TwoPhaseBusinessAction,代表一个:TCCResource
					// **********************************************************************
					TCCResource tccResource = new TCCResource();
					tccResource.setActionName(twoPhaseBusinessAction.name());
					tccResource.setTargetBean(targetBean);
					
					// prepared阶段
					tccResource.setPrepareMethod(m);
					// commit阶段
					tccResource.setCommitMethodName(twoPhaseBusinessAction.commitMethod());
					// commit(限定了入参类型必须是:BusinessActionContext)
					tccResource.setCommitMethod(ReflectionUtil
						.getMethod(interfaceClass, twoPhaseBusinessAction.commitMethod(),
							new Class[] {BusinessActionContext.class}));
					
					// rollback阶段
					tccResource.setRollbackMethodName(twoPhaseBusinessAction.rollbackMethod());
					// rollback(限定了入参类型必须是:BusinessActionContext)
					tccResource.setRollbackMethod(ReflectionUtil
						.getMethod(interfaceClass, twoPhaseBusinessAction.rollbackMethod(),
							new Class[] {BusinessActionContext.class}));
					
					// TCCResource.getBranchType() == BranchType.TCC
					//registry tcc resource
					// *****************************************************************
					// 7. 向TC注册资源
					// *****************************************************************
					DefaultResourceManager.get().registerResource(tccResource);
				}
			}
		} catch (Throwable t) {
			throw new FrameworkException(t, "parser remoting service error");
		}
	}
	
	if (remotingParser.isReference(bean, beanName)) {
		//reference bean, TCC proxy
		remotingBeanDesc.setReference(true);
	}
	return remotingBeanDesc;
}// end 
```
### (7). DefaultResourceManager.registerResource
> DefaultResourceManager我前面是有剖析的,在这里我不详解

```
// DefaultResourceManager
// 7.3 根据:BranchType获得:ResourceManager(TCCResourceManager)
public ResourceManager getResourceManager(BranchType branchType) {
	ResourceManager rm = resourceManagers.get(branchType);
	if (rm == null) {
		throw new FrameworkException("No ResourceManager for BranchType:" + branchType.name());
	}
	return rm;
} // end getResourceManager

// 7.1 调用:registerResource注册资源
public void registerResource(Resource resource) {
	// 7.2 根据branchType获取:ResourceManager(TCCResourceManager)
	// 7.4 调用:TCCResourceManager.registerResource
	getResourceManager(resource.getBranchType()).registerResource(resource);
} // end registerResource


// TCCResourceManager
public void registerResource(Resource resource) {
	TCCResource tccResource = (TCCResource)resource;
	// 先放在缓存里
	tccResourceCache.put(tccResource.getResourceId(), tccResource);
	// 7.5 调用父类:AbstractResourceManager.registerResource
	super.registerResource(tccResource);
} // end registerResource

// AbstractResourceManager
public void registerResource(Resource resource) {
	// 7.6 通过Netty通讯协议注册资源
	RmNettyRemotingClient.getInstance().registerResource(resource.getResourceGroupId(), resource.getResourceId());
} // end registerResource
```
### (8). TCCBeanParserUtils执行流程图
!["TCCBeanParserUtils执行流程图"](/assets/seata/imgs/seata-TCCBeanParserUtils-Sequence-Diagram.jpg)

### (9). 总结
> TCCBeanParserUtils.isTccAutoProxy的主要职责:    
> 1. 把Bean解析成业务模型:RemotingDesc.   
> 2. 向TC注册资源(告诉TC我是RM).   
> 3. 只有满足三个条件中一个(sofa:reference(@Refrence)/dubbo:reference(@Refrence)/@LocalTCC),才会为Bean配置织入逻辑(AOP).  
> 4. 所以,AOP的织入代码只在消费端(@LocalTCC除外)生效. 
> 5. 从现在的情况来分析:我猜测,TCC的逻辑应该是在消费者端,Seata会对消费者进行AOP的代码织入.   