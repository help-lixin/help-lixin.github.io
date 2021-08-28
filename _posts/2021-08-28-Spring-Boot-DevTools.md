---
layout: post
title: 'SpringBoot DevTools 源码学习' 
date: 2021-08-28
author: 李新
tags:  SpringBootDevTools
---

### (1). 添加依赖
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-devtools</artifactId>
	<optional>true</optional>
	<scope>runtime</scope>
</dependency>
```

### (2). 查看Spring的入口(spring.factories)
> spring-boot-devtools-2.1.0.RELEASE.jar/META-INF/spring.factories 

```
# Application Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.devtools.restart.RestartScopeInitializer

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.devtools.restart.RestartApplicationListener,\
org.springframework.boot.devtools.logger.DevToolsLogFactory.Listener

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.devtools.autoconfigure.DevToolsDataSourceAutoConfiguration,\
org.springframework.boot.devtools.autoconfigure.LocalDevToolsAutoConfiguration,\
org.springframework.boot.devtools.autoconfigure.RemoteDevToolsAutoConfiguration

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.devtools.env.DevToolsHomePropertiesPostProcessor,\
org.springframework.boot.devtools.env.DevToolsPropertyDefaultsPostProcessor
```

### (3). LocalDevToolsAutoConfiguration$RestartConfiguration
> 

```
@Configuration
@ConditionalOnInitializedRestarter
@EnableConfigurationProperties(DevToolsProperties.class)
public class LocalDevToolsAutoConfiguration {
    
	// ****************************************************************
	// 热部署配置(RestartConfiguration)
	// ****************************************************************
	@Configuration
  	@ConditionalOnProperty(prefix = "spring.devtools.restart", name = "enabled", matchIfMissing = true)
  	static class RestartConfiguration
  			implements ApplicationListener<ClassPathChangedEvent> {
  
  		private final DevToolsProperties properties;
  
        // 1. 注入配置文件
  		RestartConfiguration(DevToolsProperties properties) {
  			this.properties = properties;
  		}
  
  		@Override
  		public void onApplicationEvent(ClassPathChangedEvent event) {
  			if (event.isRestartRequired()) {
  				Restarter.getInstance().restart(
  						new FileWatchingFailureHandler(fileSystemWatcherFactory()));
  			}
  		}
  
         // 2. 配置文件监听
  		@Bean
  		@ConditionalOnMissingBean
  		public ClassPathFileSystemWatcher classPathFileSystemWatcher() {
			// urls = [file:/Users/lixin/Workspace/spring-web-demo/target/classes/]
  			URL[] urls = Restarter.getInstance().getInitialUrls();
			
			// *****************************************************************
			// 从现在情况来看:ClassPathFileSystemWatcher应该是重点了,因为:它聚合着监听和重启策略
			// *****************************************************************
  			ClassPathFileSystemWatcher watcher = new ClassPathFileSystemWatcher(
			        // 3. 创建文件系统监听工厂
  					fileSystemWatcherFactory(), 
					// 4. 创建重启策略
					classPathRestartStrategy(), 
					urls);
  			watcher.setStopWatcherOnRestart(true);
  			return watcher;
  		}
  
        // 4.1 创建重启策略
  		@Bean
  		@ConditionalOnMissingBean
  		public ClassPathRestartStrategy classPathRestartStrategy() {
			// 设置哪些文件排除不在重启的范围内
  			return new PatternClassPathRestartStrategy(this.properties.getRestart().getAllExclude());
  		}
  
  		@Bean
  		public HateoasObjenesisCacheDisabler hateoasObjenesisCacheDisabler() {
  			return new HateoasObjenesisCacheDisabler();
  		}
  
        // 3.1 创建文件系统监听工厂
  		@Bean
  		public FileSystemWatcherFactory fileSystemWatcherFactory() {
  			return this::newFileSystemWatcher;
  		}
  
  		@Bean
  		@ConditionalOnProperty(prefix = "spring.devtools.restart", name = "log-condition-evaluation-delta", matchIfMissing = true)
  		public ConditionEvaluationDeltaLoggingListener conditionEvaluationDeltaLoggingListener() {
  			return new ConditionEvaluationDeltaLoggingListener();
  		}
  
        // 3.2 创建文件系统监听工厂
  		private FileSystemWatcher newFileSystemWatcher() {
			// 获得重启部份的属性配置
  			Restart restartProperties = this.properties.getRestart();
			
			// 文件系统监听(间隔/频率)
  			FileSystemWatcher watcher = new FileSystemWatcher(true,
  					restartProperties.getPollInterval(),
  					restartProperties.getQuietPeriod());
  			
			// 触发文件
			String triggerFile = restartProperties.getTriggerFile();
  			if (StringUtils.hasLength(triggerFile)) {
  				watcher.setTriggerFilter(new TriggerFileFilter(triggerFile));
  			}
			
			// 监听的目录
  			List<File> additionalPaths = restartProperties.getAdditionalPaths();
  			for (File path : additionalPaths) {
  				watcher.addSourceFolder(path.getAbsoluteFile());
  			}
  			return watcher;
  		}
  
  	}// end 
}
```
### (4). ClassPathFileSystemWatcher
```
public class ClassPathFileSystemWatcher
		implements InitializingBean, DisposableBean, ApplicationContextAware {

	private final FileSystemWatcher fileSystemWatcher;

	private ClassPathRestartStrategy restartStrategy;

	private ApplicationContext applicationContext;

	private boolean stopWatcherOnRestart;

	// 1. 监听工厂/重启策略/监听URL
	public ClassPathFileSystemWatcher(FileSystemWatcherFactory fileSystemWatcherFactory,
			ClassPathRestartStrategy restartStrategy, URL[] urls) {
		Assert.notNull(fileSystemWatcherFactory,
				"FileSystemWatcherFactory must not be null");
		Assert.notNull(urls, "Urls must not be null");
		this.fileSystemWatcher = fileSystemWatcherFactory.getFileSystemWatcher();
		this.restartStrategy = restartStrategy;
		this.fileSystemWatcher.addSourceFolders(new ClassPathFolders(urls));
	}

	/**
	 * Set if the {@link FileSystemWatcher} should be stopped when a full restart occurs.
	 * @param stopWatcherOnRestart if the watcher should be stopped when a restart occurs
	 */
	public void setStopWatcherOnRestart(boolean stopWatcherOnRestart) {
		this.stopWatcherOnRestart = stopWatcherOnRestart;
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext)
			throws BeansException {
		this.applicationContext = applicationContext;
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		if (this.restartStrategy != null) {
			FileSystemWatcher watcherToStop = null;
			if (this.stopWatcherOnRestart) {
				watcherToStop = this.fileSystemWatcher;
			}
			
			// ****************************************************************
			// 为FileSystemWatcher类,配置监听器
			// ****************************************************************
			this.fileSystemWatcher.addListener(new ClassPathFileChangeListener(
					this.applicationContext, this.restartStrategy, watcherToStop));
		}
		
		// *******************************************************************
		// 为FileSystemWatcher类,启动监听
		// *******************************************************************
		this.fileSystemWatcher.start();
	}

	// 停止监听
	@Override
	public void destroy() throws Exception {
		this.fileSystemWatcher.stop();
	}
}
```
### (5). 总结
通过FileSystemWatcher类监听目录变化(监听的目录,仅为当前开发的工程目录,所以,class文件相对比较少),当目录发生变化时,触发ClassLoader的重新加载.  