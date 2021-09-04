---
layout: post
title: 'Liquibase源码之ChangeLogParserFactory(一)' 
date: 2021-09-03
author: 李新
tags:  Liquibase
---

### (1). 概述
从前面的学习,大概的了解到,Liquibase会读取XML,并转换成SQL文件,最终会在目标库运行脚本(保证一致性/元子性).   
那么,Liquibase是如何解析XML的呢?能否支持扩展呢?答案就在:ChangeLogParserFactory.      

### (2). Liquibase.getDatabaseChangeLog
```
public class Liquibase implements AutoCloseable {
	
	public void update(int changesToApply, Contexts contexts, LabelExpression labelExpression)
	            throws LiquibaseException {
        
		// ... ...
		// ************************************************************************
		// 解析(XML/JSON/YAML/SQLFile),转换成业务模型:DatabaseChangeLog
		// ************************************************************************
		getDatabaseChangeLog()
		// ... ...
	}
		
	public DatabaseChangeLog getDatabaseChangeLog() throws LiquibaseException {
		// databaseChangeLog 解析XML/JSON/YML之后的业务模型对象.
		// changeLogFile 待解析的数据文件.
		if (databaseChangeLog == null && changeLogFile != null) {
			// ************************************************************************
			// 2. ChangeLogParserFactory为典型的工厂模式+单例模式
			// ************************************************************************
			
			// ************************************************************************
			// 通过SPI加载:ChangeLogParser类的所有实现类,遍历所有的:ChangeLogParser.supports方法,是否支持解析
			// supports比较简单,判断changeLogFile的后缀即可.
			// ChangeLogParser的实现类有: YamlChangeLogParser/SqlChangeLogParser/XMLChangeLogSAXParser/...
			// ************************************************************************
			ChangeLogParser parser = ChangeLogParserFactory.getInstance().getParser(changeLogFile, resourceAccessor);
			
			// ************************************************************************
			// 委托给ChangeLogParser.parse方法进行解析(我这里是XML,所以,会委托给:XMLChangeLogSAXParser).
			// ************************************************************************
			databaseChangeLog = parser.parse(changeLogFile, changeLogParameters, resourceAccessor);
		}
		return databaseChangeLog;
	} //end getDatabaseChangeLog
}	
```
### (3). ChangeLogParserFactory.getInstance
```
public class ChangeLogParserFactory {

    private static ChangeLogParserFactory instance;

    private List<ChangeLogParser> parsers = new ArrayList<>();

    public static synchronized ChangeLogParserFactory getInstance() {
		if (instance == null) {
			// ******************************************************************
			// 1. 典型的单例,构建出:ChangeLogParserFactory
			// ******************************************************************
			instance = new ChangeLogParserFactory();
		}
		return instance;
	} // end getInstance
	

	private ChangeLogParserFactory() {
		try {
			// ******************************************************************
			// 2. Scope有点类似于Spring中的容器,这里典型的组合模式.
			// ******************************************************************
			List<ChangeLogParser> parser = Scope.getCurrentScope().getServiceLocator().findInstances(ChangeLogParser.class);
			register(parser);
		} catch (Exception e) {
			throw new UnexpectedLiquibaseException(e);
		}
	} // end 	ChangeLogParserFactory
}	
```
### (4). Scope 
```
public class Scope {
	
	// 定义有哪些可枚举的类
	public enum Attr {
		logService,
		ui,
		resourceAccessor,
		classLoader,
		database,
		quotingStrategy,
		changeLogHistoryService,
		lockService,
		executeMode,
		lineSeparator,
		serviceLocator,
		fileEncoding,
		databaseChangeLog,
		changeSet,
	} // end Attr
	
	private static ScopeManager scopeManager;
	
	// 典型的组合模式,在Java的ClassLoader里有这样的写法,在Spring国际化支持也有这样的写法
    // 目的是:Scop是有层级的,各司其职.
	private Scope parent;
	
	// 
	private SmartMap values = new SmartMap();
	private String scopeId;

	private LiquibaseListener listener;
	
	public static Scope getCurrentScope() {
		
		// 这样初始化:SingletonScopeManager是否会存在并发的问题,不过,Liquibase本身就不支持并发操作.
		if (scopeManager == null) {
			scopeManager = new SingletonScopeManager();
		}
		
		// 可以理解为,这里是Spring的父容器,通过父容器,加载一些常用的类实例.而子容器加载一些业务性质的类实例.
		if (scopeManager.getCurrentScope() == null) {
			Scope rootScope = new Scope();
			scopeManager.setCurrentScope(rootScope);

			// rootScope创建logService-->JavaLogService的映射关系
			rootScope.values.put(Attr.logService.name(), new JavaLogService());
			// rootScope创建resourceAccessor-->ClassLoaderResourceAccessor的映射关系
			rootScope.values.put(Attr.resourceAccessor.name(), new ClassLoaderResourceAccessor());
			// rootScope创建serviceLocator-->StandardServiceLocator的映射关系
			rootScope.values.put(Attr.serviceLocator.name(), new StandardServiceLocator());
			// rootScope创建ui-->ConsoleUIService的映射关系
			rootScope.values.put(Attr.ui.name(), new ConsoleUIService());
			
			// ***********************************************************************************
			// 1. 通过反射创建:LiquibaseConfiguration,并调用:init方法
			// ***********************************************************************************
			rootScope.getSingleton(LiquibaseConfiguration.class).init(rootScope);
			
			// 重写:LogService
			LogService overrideLogService = rootScope.getSingleton(LogServiceFactory.class).getDefaultLogService();
			if (overrideLogService == null) {
				throw new UnexpectedLiquibaseException("Cannot find default log service");
			}
			rootScope.values.put(Attr.logService.name(), overrideLogService);

			// 设置优先级最高的:ServiceLocator
			//check for higher-priority serviceLocator
			ServiceLocator serviceLocator = rootScope.getServiceLocator();
			for (ServiceLocator possibleLocator : serviceLocator.findInstances(ServiceLocator.class)) {
				if (possibleLocator.getPriority() > serviceLocator.getPriority()) {
					serviceLocator = possibleLocator;
				}
			}

			rootScope.values.put(Attr.serviceLocator.name(), serviceLocator);
		}
		return scopeManager.getCurrentScope();
	} // end getCurrentScope
	
	
	public <T extends SingletonObject> T getSingleton(Class<T> type) {
		// 优先从父Scope加载
		if (getParent() != null) {
			return getParent().getSingleton(type);
		}

		// 父Scope不存在的情况下,自己再进行加载
		String key = type.getName();
		T singleton = get(key, type);
		if (singleton == null) {
			try {
				try {
					// 尝试,通过有参(Scope),构建出Class<T> type,抛出异常的情况下,通过默认构造器,构建出Class<T> type.
					Constructor<T> constructor = type.getDeclaredConstructor(Scope.class);
					constructor.setAccessible(true);
					singleton = constructor.newInstance(this);
				} catch (NoSuchMethodException e) { //try without scope
					// 通过默认构造器,构建出:Class<T> type对象.
					Constructor<T> constructor = type.getDeclaredConstructor();
					constructor.setAccessible(true);
					singleton = constructor.newInstance();
				}
			} catch (Exception e) {
				throw new UnexpectedLiquibaseException(e);
			}
			values.put(key, singleton);
		}
		return singleton;
	} // end getSingleton
	
}
```
### (5). LiquibaseConfiguration.init
```
public class LiquibaseConfiguration implements SingletonObject {
	
	// init仅仅只是通过SPI加载了类的实现,并没有初始化,在3.6.x里,是调用相应的工厂时,才去加载.
	// 现在变成了运行期就去加载.
	public void init(Scope scope) {
		// 先清一下缓存
		configurationValueProviders.clear();
		// *******************************************************************
		// 1. 从scope里获得ServiceLocator,在这里按着现类为:StandardServiceLocator
		// *******************************************************************
		ServiceLocator serviceLocator = scope.getServiceLocator();
		
		// *******************************************************************
		// 2. 委托给:StandardServiceLocator.findInstances方法,去加载:AutoloadedConfigurations类的子类.
		// *******************************************************************
		final List<AutoloadedConfigurations> containers = serviceLocator.findInstances(AutoloadedConfigurations.class);
		
		for (AutoloadedConfigurations container : containers) {
			// 3. 通过日志记录.
			Scope.getCurrentScope().getLog(getClass()).fine("Found ConfigurationDefinitions in " + container.getClass().getName());
		}
		
		// 4. 委托给:StandardServiceLocator.findInstances方法,去加载:ConfigurationValueProvider类的子类.
		configurationValueProviders.addAll(serviceLocator.findInstances(ConfigurationValueProvider.class));
	} //end init
}
```
### (6). StandardServiceLocator.findInstances
```
public class StandardServiceLocator implements ServiceLocator {
	
	public <T> List<T> findInstances(Class<T> interfaceType) throws ServiceNotFoundException {
		// 所有实例的结果集
		List<T> allInstances = new ArrayList<>();
		
		// 1. 获得日志服务
		final Logger log = Scope.getCurrentScope().getLog(getClass());
		
		// *******************************************************************
		// 2. 这里,又把请求委托给了:java.util.ServiceLoader.load方法去加载Class<T>的所有实现类.
		//    注意:这里用的是java提供的SPI,所以,只需要找这个文件即可:META-INF/services/{interfaceType}
		//    相比,以前的3.6.x版本,提升了不少,感觉已经允许自己进行扩展了.
		// *******************************************************************
		final Iterator<T> services = ServiceLoader.load(interfaceType, Scope.getCurrentScope().getClassLoader(true)).iterator();
		while (services.hasNext()) {
			try {
				final T service = services.next();
				log.fine("Loaded "+interfaceType.getName()+" instance "+service.getClass().getName());
				allInstances.add(service);
			} catch (Throwable e) {
				log.info("Cannot load service: "+e.getMessage());
				log.fine(e.getMessage(), e);
			}
		} 
		return Collections.unmodifiableList(allInstances);
	}// end
	
}	
```

### (7). ChangeLogParser.


### (8). 总结
从上面的代码能剖析出:Liquibase对ChangeLogParser进行抽象,它的功能主要是对ChangeLog进行解析,ChangeLogParser实现类有:YamlChangeLogParser/SqlChangeLogParser/XMLChangeLogSAXParser.   
Liquibase在3.6.x版本时,SPI是自己写的,而在最新版本的时候(4.5.0),开始进行了优化,把SPI交给了jdk,所以,你想扩展的话,只要符合jdk SPI规范即可!        