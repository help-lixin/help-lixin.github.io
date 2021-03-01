---
layout: post
title: 'Liquibase Sharding-jdbc整合(三)'
date: 2021-02-27
author: 李新
tags: Liquibase 解决方案
---

### (1). 概述
> 先描述下问题,当使用Shardin-jdbc之后,从开发的视角,只有虚拟表了,那如何让Liquibase根据虚拟表信息,生成物理表呢?  
> 在这里我只聊下大概思路:  
> 1. 获得Sharding-jdbc代理的DataSource(ShardingDataSource).   
> 2. 在ShardingDataSource里有所有真实数据源的集合(dataSource.getDataSourceMap()).   
> 3. 在ShardingDataSource里有ShardingRuntimeContext,里面存在表规则信息(虚拟表与物理表的关系),提取这些表规则信息,转换成你的业务模型.    
> 4. 根据业务模型,调用Liquibase创建表.  

### (2). 提取ShardingDataSource信息,转换成业务模型(DataBaseInfo)
```
@Bean
@Conditional(ShardingJdbcDataSourceCondition.class)
@ConditionalOnClass(DataSource.class)
public DataBaseInfo dataBaseInfo(DataSource shardingDataSource) {
	if (null != shardingDataSource && shardingDataSource instanceof ShardingDataSource) {
		ShardingDataSource dataSource = (ShardingDataSource) shardingDataSource;
		// 所有的数据源集合.
		Map<String, DataSource> dataSourceMap = dataSource.getDataSourceMap();

		// 业务模型对象
		DataBaseInfo.Builder builder = DataBaseInfo.newBuilder();
		builder.dataSources(dataSourceMap);

		// 获得Sharding-jdbc配置信息.
		ShardingRuntimeContext shardingRuntimeContext = dataSource.getRuntimeContext();

		// 默认的数据源
		ShardingRuleConfiguration shardingRuleConfiguration = dataSource.getRuntimeContext().getRule().getRuleConfiguration();
		builder.defaultDataSourceName(shardingRuleConfiguration.getDefaultDataSourceName());

		// 数据源的类型(MySQL/Oracle...)
		String databaseType = shardingRuntimeContext.getDatabaseType().getName();
		builder.databaseType(databaseType);

		// 获得规则信息集
		ShardingRule shardingRule = shardingRuntimeContext.getRule();
		Collection<TableRule> tableRules = shardingRule.getTableRules();
		for (TableRule tableRule : tableRules) {
			String logicTable = tableRule.getLogicTable();
			List<DataNode> dataNodes = tableRule.getActualDataNodes();
			dataNodes.forEach(dataNode -> {
				String tableName = dataNode.getTableName();
				String dataSourceName = dataNode.getDataSourceName();
				TableInfo tableInfo = TableInfo.newBuilder().logicTable(logicTable)
						.dataSourceName(dataSourceName)
						.tableName(tableName).build();
				builder.addTableInfo(tableInfo);
				if (logger.isDebugEnabled()) {
					logger.debug("logicTable:[{}],tableNmae:[{}],dataSource:[{}]", logicTable,
							tableName, dataSourceName);
				}
			});
			return builder.build();
		}
	}
	return DataBaseInfo.newBuilder().build();
}
```
### (3). 将业务模型(DataBaseInfo),转换成Liquibase
```
@Bean
@ConditionalOnBean(DataBaseInfo.class)
public Map<Liquibase, Contexts> liquibases(
		DataBaseInfo dataBaseInfo,
		ObjectProvider<List<LiquibaseCustomizer>> customizersList,
		ObjectProvider<Map<String, Class<? extends Database>>> databasesMap,
		LiquibaseIntegrationProperties liquibaseIntegrationProperties,
		LiquibaseResourceLoader liquibaseResourceLoader) throws Exception {
	Map<String, Class<? extends Database>> databases = databasesMap.getIfAvailable();
	List<LiquibaseCustomizer> customizers = customizersList.getIfAvailable();
	Map<Liquibase, Contexts> liquibases = new HashMap<>();

	// changeLog位置
	String changeLogFile = liquibaseIntegrationProperties.getChangeLog();
	if (null == changeLogFile) {
		logger.error("liquibase.changeLogFile properties is require");
		throw new IllegalArgumentException("liquibase.changeLogFile properties is require");
	}
	// 数据库类型
	String databaseType = dataBaseInfo.getDatabaseType();
	// 数据源集合
	Map<String, DataSource> dataSourceMap = dataBaseInfo.getDataSourceMap();
	// 上下文信息
	String context = liquibaseIntegrationProperties.getContexts();
	// 默认的数据源名称
	String defaultDataSourceName = dataBaseInfo.getDefaultDataSourceName();

	// 创建资源访问授权器
	ResourceAccessor resourceAccessor = new SpringResourceAccessor(liquibaseResourceLoader.getResourceLoader());

	// 构建默认表的信息
	if (null != defaultDataSourceName) {
		StringBuilder contextBuilder = new StringBuilder(context);
		contextBuilder.append(",").append(TableType.PhysicalTable);

		// 获得数据源
		DataSource dataSource = dataSourceMap.get(defaultDataSourceName);
		// 根据数据源创建:Database
		Database database = buildDatabase(databases, databaseType, dataSource);

		// 构建:Liquibase
		Liquibase liquibase = new Liquibase(changeLogFile, resourceAccessor, database);
		// *******************************************************************
		// 构建上下文,这一步很重要:
		// 在Sharding-jdbc中默认的数据源,对上下文很重要,当上下文为:TableType.PhysicalTable时
		// changeLog中context对应得上的才会执行.
		// *******************************************************************
		Contexts contexts = new Contexts(contextBuilder.toString());
		// 允许业务对:liquibase进行自定义(实现:LiquibaseCustomizer即可)
		if (!customizers.isEmpty()) {
			customizers.forEach(customizer -> customizer.customize(liquibase));
		}
		liquibases.put(liquibase, contexts);
	}

	// 所有的逻辑表信息
	Iterator<TableInfo> iterator = dataBaseInfo.getTableInfos().iterator();
	while (iterator.hasNext()) {
		TableInfo tableInfo = iterator.next();
		String logicTable = tableInfo.getLogicTable();
		String tableName = tableInfo.getTableName();
		String dataSourceName = tableInfo.getDataSourceName();

		// 1. 首先增加用户自定义的上下文信息
		StringBuilder contextBuilder = new StringBuilder(context);
		// *********************************************************************
		// 2. TableInfo信息存在的情况下,代表着这是一张逻辑表(虚表)
		// *********************************************************************
		contextBuilder.append(",").append(TableType.LogicalTable);

		// *********************************************************************
		// 3. 如果有配置默认数据源,代表没有分片的表都路由到这个数据源上(所以,属于物理表)
		// *********************************************************************
		if (null != defaultDataSourceName && defaultDataSourceName.equalsIgnoreCase(dataSourceName)) {
			contextBuilder.append(",").append(TableType.PhysicalTable);
		}

		// 检查规则对应的数据源是否存在
		if (!dataSourceMap.containsKey(dataSourceName)) {
			String formatLine = String.format("Rule dataSourceName:[%s],logicTable:[%s],physicsTable:[%s],Not Found DataSource", defaultDataSourceName, logicTable, tableName);
			logger.error(formatLine);
			throw new IllegalArgumentException(formatLine);
		}

		// 获得数据源
		DataSource dataSource = dataSourceMap.get(dataSourceName);
		// 根据数据源创建:Database
		Database database = buildDatabase(databases, databaseType, dataSource);

		// 构建:Liquibase
		Liquibase liquibase = new Liquibase(changeLogFile, resourceAccessor, database);
		// *********************************************************************
		// 逻辑表与物理表的关系(这个是必须要存在的,开发在chagneLog时,表名称可以变量[逻辑表名称])
		// *********************************************************************
		liquibase.setChangeLogParameter(logicTable, tableName);
		// *********************************************************************
		// 物理表与数据源关系(这个是附助数据.)
		// *********************************************************************
		liquibase.setChangeLogParameter(tableName, dataSourceName);

		// 允许业务对:liquibase进行自定义(实现:LiquibaseCustomizer即可)
		if (!customizers.isEmpty()) {
			customizers.forEach(customizer -> customizer.customize(liquibase));
		}

		if (logger.isDebugEnabled()) {
			logger.debug("build Liquibase SUCCESS.[{}]", liquibase);
		}

		// 构建上下文
		Contexts contexts = new Contexts(contextBuilder.toString());
		liquibases.put(liquibase, contexts);
	}
	return liquibases;
}


private Database buildDatabase(Map<String, Class<? extends Database>> databases,
							   String databaseType,
							   DataSource dataSource) throws Exception {
	Database database = null;
	if (!databases.containsKey(databaseType)) {
		String formatLine = String.format("databases:[%s],No Match databaseType:[%s]", databases, databaseType);
		throw new IllegalArgumentException(formatLine);
	}
	// 不能共用对象,每一次都要构建出新的来
	database = databases.get(databaseType).newInstance();
	DatabaseConnection databaseConnection = buildDatabaseConnection(dataSource);
	database.setConnection(databaseConnection);
	return database;
}

private DatabaseConnection buildDatabaseConnection(DataSource dataSource) throws Exception {
	Connection connection = dataSource.getConnection();
	DatabaseConnection databaseConnection = new JdbcConnection(connection);
	return databaseConnection;
}
```
### (4). 调用Liquibase.update方法,进行建表
> 经过测试,发现:Liquibase不支持并发.  

```
public class LiquibaseLifecycle implements ApplicationContextAware, SmartLifecycle {
    private final Logger log = LoggerFactory.getLogger(LiquibaseLifecycle.class);

    private AtomicBoolean running = new AtomicBoolean(false);

    private Map<Liquibase, Contexts> liquibases;
    private ApplicationContext applicationContext;


    public LiquibaseLifecycle(Map<Liquibase, Contexts> liquibases) {
        this.liquibases = liquibases;
    }


    @Override
    public boolean isAutoStartup() {
        return true;
    }

    @Override
    public void stop(Runnable callback) {
        stop();
        callback.run();
    }

    @Override
    public int getPhase() {
        return 0;
    }

    @Override
    public void start() {
        if (running.compareAndSet(false, true)) {
            List<LiquibaseExecute> executes = new ArrayList<>(liquibases.size());
            CountDownLatch latch = new CountDownLatch(liquibases.size());

            // 把Liquibase集合转换成:Execute集合.
            Iterator<Map.Entry<Liquibase, Contexts>> iterator = liquibases.entrySet().iterator();
            while (iterator.hasNext()) {
                Map.Entry<Liquibase, Contexts> entry = iterator.next();
                Liquibase liquibase = entry.getKey();
                Contexts contexts = entry.getValue();
                // 转换成:LiquibaseExecute对象.
                LiquibaseExecute execute = new LiquibaseExecute(latch, liquibase, contexts);
                executes.add(execute);
            }

            for (int i = 0; i < executes.size(); i++) {
                // 这种任务只执行一次,不需要建线程池
                // 为每一个Liquibase构建线程
                // TODO lixin 测试发现:Liquibase不支持并发创建表.
                Thread thread = new Thread(executes.get(i));
                thread.setDaemon(true);
                thread.setName("liquibase-generate-table-" + i);
                thread.run();
            }

            try {
                // 等待所有的count
                latch.await();

                // 是否所有的Executer的结果都是成功的
                boolean isAllSuccess = false;
                // 获取成功的数据
                long successCount = executes.stream().filter(execute -> execute.getIsSuccess().get()).count();
                if (successCount == executes.size()) {
                    isAllSuccess = true;
                }

                if (isAllSuccess) {
                    // 发布事件,触发向注册中心进行注册.
                    applicationContext.publishEvent(new RegisterServiceStartEvent("TriggerRegister"));
                    running.set(Boolean.TRUE);
                } else {
                    // 获取所有失败的:Execute
                    List<LiquibaseExecute> failExecutes = executes.stream().filter(execute -> execute.getIsSuccess().get() == false).collect(Collectors.toList());
                    for (LiquibaseExecute execute : failExecutes) {
                        log.error("execute liquibase FAIL,description:[{}]", execute);
                    }
                }
            } catch (InterruptedException ignore) {
            }
        }
    }

    @Override
    public void stop() {
    }

    @Override
    public boolean isRunning() {
        return running.get();
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}

class LiquibaseExecute implements Runnable {
    private final Logger log = LoggerFactory.getLogger(LiquibaseExecute.class);

    private Liquibase liquibase;
    private Contexts context;
    private AtomicBoolean isSuccess = new AtomicBoolean(false);
    private Throwable exception = null;
    private CountDownLatch latch;

    public LiquibaseExecute(CountDownLatch latch, Liquibase liquibase, Contexts context) {
        this.latch = latch;
        this.liquibase = liquibase;
        this.context = context;
    }

    public Contexts getContext() {
        return context;
    }

    public Liquibase getLiquibase() {
        return liquibase;
    }

    public AtomicBoolean getIsSuccess() {
        return isSuccess;
    }

    public Throwable getException() {
        return exception;
    }

    @Override
    public void run() {
        try {
            log.info("START execute generate table for liquibase:[{}]", liquibase);
            liquibase.update(context);
            isSuccess.compareAndSet(false, true);
            log.info("END execute generate table for liquibase:[{}]", liquibase);
        } catch (Throwable e) {
            exception = e;
            log.warn("FAIL execute generate table for liquibase:[{}],exception:[{}]", liquibase, e);
        }
        // 放在代码,最后面:latch只是用来记录线程是否执行完毕.不做成功或异常处理.
        latch.countDown();
    }

    @Override
    public String toString() {
        return "Execute{" +
                "liquibase=" + liquibase +
                ", context=" + context +
                '}';
    }
}
```


### (4). root.changelog.xml
```
<?xml version="1.0" encoding="UTF-8"?>

<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-2.0.xsd">
    <!-- 逻辑表context="LogicalTable" -->
    <!-- 在编写逻辑表的XML时,(必须)要注意三点: -->
    <!-- 1. tableName="${t_order}",这里的变量名称是你在sharding-jdbc中你声明的逻辑表名称 -->
    <!-- 2. 如果是逻辑表,要指定这个值:context="LogicalTable" -->
    <!-- 3. liquibase 是根据:id+author计算(check sum),所以,要在auth上增加多一个标识(_${t_order}),否则,同一个库不同的表,会创建失败 -->
	<!-- 比如: md5(1+lixin) = 123 ==> t_order_1 -->
	<!-- 比如: md5(1+lixin) = 123 ==> t_order_2 -->
	<!-- 上面的例子就会造成,在db1库,只会创建一张表成功,另一张表不成功,提示:已经存在. -->
    <changeSet id="1" author="lixin_${t_order}" context="LogicalTable">
        <createTable tableName="${t_order}">
            <column name="order_id" type="bigint(20)">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="price" type="decimal(10,2)"/>
            <column name="user_id" type="bigint(20)"/>
            <column name="status" type="varchar(50)"/>
        </createTable>
    </changeSet>
	
    <!-- 物理表(context="PhysicalTable") -->
	<!-- 物理表注意事项: -->
	<!-- 1. context="PhysicalTable" 必须指定为物理表 -->
	<!-- 2. 表名称不再需要变量了. -->
    <changeSet id="2" author="lixin" context="PhysicalTable">
        <createTable tableName="t_user">
            <column name="user_id" type="bigint(20)">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="fullname" type="varchar(255)"/>
            <column name="user_type" type="char(1)"/>
        </createTable>
    </changeSet>
</databaseChangeLog>

```
### (5). 总结
> 需要做好规划,在编写changelog时,开发要清楚使用物理表和逻辑表.   