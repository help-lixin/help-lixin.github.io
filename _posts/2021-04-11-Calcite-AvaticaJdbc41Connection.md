---
layout: post
title: 'Calcite 通过DriverManager获取Connection过程(二)'
date: 2021-04-11
author: 李新
tags:  Calcite源码
---

### (1). 概述
> 在这里,主要剖析:DriverManager.getConnection("jdbc:calcite:", info)方法的内部实现.   

### (2). DriverManager.getConnection

```
public static Connection getConnection(
                  String url,
				  java.util.Properties info) throws SQLException {
    // 1. 委托给了另一个方法
	// Reflection.getCallerClass() 获得当前调用Class
	return (getConnection(url, info, Reflection.getCallerClass()));
} // end getConnection


private static Connection getConnection(
			String url, 
			java.util.Properties info, 
			// org.apache.calcite.test.CsvTest2.class
			Class<?> caller) throws SQLException {
			
    // ... ...
	
	// 2. 获得当前类(org.apache.calcite.test.CsvTest2.class)的ClassLoader
	ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
	
	// 这里的:registeredDrivers为上一章节所说的Driver容器.
    for(DriverInfo aDriver : registeredDrivers) {
		// 3. 通过:org.apache.calcite.test.CsvTest2类的ClassLoader尝试加载:Driver
		// 如果,能加载到,返回:true
		// 如果,不能加载,返回:false
		// 如果,连ClassLoader都不能加载,就不能谈能调用:Driver的方法了.
		if(isDriverAllowed(aDriver.driver, callerCL)) {
			try {
				println("trying " + aDriver.driver.getClass().getName());
				// 4. 创建连接
				Connection con = aDriver.driver.connect(url, info);
				if (con != null) {
					println("getConnection returning " + aDriver.driver.getClass().getName());
					return (con);
				}
			} catch (SQLException ex) {
				if (reason == null) {
					reason = ex;
				}
			}
		} else {
			println("    skipping: " + aDriver.getClass().getName());
		}
	}// end for	
}
```
### (3). org.apache.calcite.jdbc.Driver
```
public abstract class UnregisteredDriver implements java.sql.Driver {
  final DriverVersion version;
  protected final AvaticaFactory factory;
  public final Handler handler;

	public Connection connect(
					  // jdbc:calcite:
	                  String url, 
					  // { model = "/model.json" }
					  Properties info) throws SQLException {
		// 判断url是否支持的URL				  
		if (!acceptsURL(url)) {
		  return null;
		}
		
		// 
		final String prefix = getConnectStringPrefix();
		assert url.startsWith(prefix);
		final String urlSuffix = url.substring(prefix.length());
		final Properties info2 = ConnectStringParser.parse(urlSuffix, info);
		// ********************************************************
		// 通过AvaticaJdbc41Factory工厂,创建连接.
		// connection在Drvier的构造器时,初始化的.
		// ********************************************************
		final AvaticaConnection connection = factory.newConnection(this, factory, url, info2);
		handler.onConnectionInit(connection);
		return connection;
	} //end connect
}
```
### (4). AvaticaJdbc41Factory.newConnection
```
public AvaticaConnection newConnection(
      UnregisteredDriver driver,
      AvaticaFactory factory,
      String url,
      Properties info) {
		// 创建了:AvaticaJdbc41Connection(是java.sql.Connection的实现)包裹:Driver/AvaticaFactory/url/{ model = "/model.json" }
    return new AvaticaJdbc41Connection(driver, factory, url, info);
}
```
### (5). 我们看下Calcite相关的UML图
> Calcite对Driver/Connection/PreparedStatement,ResulSet 进行了自己的实现. 

!["Calcite UML图"](/assets/calcite/imgs/calcite-get-connection.jpg)

### (6). 总结
> 1. 当我们调用:DriverManager.getConnection时,DriverManager内部有一个Drvier容器,会判断,当前类(XXXMapper/XXXTest)的ClassLoader能否加载:Driver,如果能加载,就通过这个Driver做处理.  
> 2. 委托给:AvaticaJdbc41Factory类,去创建一个:java.sql.Connection的实现类(AvaticaJdbc41Connection).  
> 3. 我们仍然没有分析到:Calcite是如何分析SQL的,还需要继往下分析.   