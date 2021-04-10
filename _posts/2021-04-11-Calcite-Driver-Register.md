---
layout: post
title: 'Calcite Driver是如何向DriverManager进行注册的(一)'
date: 2021-04-11
author: 李新
tags:  Calcite源码
---

### (1). 概述
> 前面通过SQL可以读取CSV文件,那么,现在开始对源码进行分析.入口自然是:DriverManager,凭什么,我通过SQL就能对CSV进行Query? 

### (2). DriverManager
> DriverManager是JDK提供的一个类,我们看一下它的源码.  

```
package java.sql;

public class DriverManager {
	// 1. 静态代码块,当我们通过ClassLoader加载这个类的时候,该静态代码块会调用一次,也仅此一次.
	static {
		// 2. 加载Driver
		loadInitialDrivers();
		// ... ...
	} // end static
	
	private static void loadInitialDrivers() {
		
		AccessController.doPrivileged(new PrivilegedAction<Void>() {
			public Void run() {
				// *****************************************
				// 3. 通过SPI加载:java.sql.Driver的所有实现,但是,在代码上,好像什么都没做,就只是纯粹的遍历了一次.
				//    核心逻辑就在:Driver的实现类上了,以:org.apache.calcite.jdbc.Driver为例
				//    org.apache.calcite.jdbc.Driver的静态方法会创建一个实例,调用: DriverManager.registerDriver(this);
				//    所以,只要通过SPI加载,就会初始化这个静态代码块,驱动着java.sql.Driver的实现类,调用:DriverManager.registerDriver进行Drvier注册.
				// *****************************************
				ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
				Iterator<Driver> driversIterator = loadedDrivers.iterator();
				try{
					while(driversIterator.hasNext()) {
						driversIterator.next();
					}
				} catch(Throwable t) {

				}
				return null;
			}
		});
		
		// ... ...
	}// end loadInitialDrivers
	
	
	// ***********************************************
	// 4. 注册Driver
	// ***********************************************
	public static synchronized void registerDriver(java.sql.Driver driver)
	        throws SQLException {
		// 委托给另一个方法:registerDriver(driver, null)
		registerDriver(driver, null);
	} // end registerDriver
	
	
	// 注册Drider
	public static synchronized void registerDriver(java.sql.Driver driver,
	            DriverAction da) throws SQLException {
				
		if(driver != null) {
			// ***********************************************
			// 5. DrvierManager内部有一个容器(map),会保存所有:java.sql.Driver类的实现.
			// ***********************************************
			registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
		} else {
			// This is for compatibility with the original DriverManager
			throw new NullPointerException();
		}
		// ... ...
	}
}
```
### (3). org.apache.calcite.jdbc.Driver
```
public class Driver extends UnregisteredDriver {
	// 1. 通过SPI加载后,会触发静态代码块的加载
	static {
		// 向DrvierManger进行注册
	    new Driver().register();
	}
	
	// UnregisteredDriver代码
	protected void register() {
	    try {
			// ***********************************************
			// 向DriverManager注册自己.
			// ***********************************************
	      DriverManager.registerDriver(this);
	    } catch (SQLException e) {
	      System.out.println(
	          "Error occurred while registering JDBC driver "
	          + this + ": " + e.toString());
	    }
	  }
}
```
### (4). 总结
> 1. 当我们(开发)使用:DriverManager这个类时,静态代码块会先加载,在静态方法内部会触发SPI加载所有的Driver的实现类.    
> 2. 而Driver的实现,也有静态代码,会把自己构建出来,然后,调用:DriverManager.registerDriver.    
> 3. 此时,DriverManager有一个容器(静态Map),会Hold住所有:Drvier的实现类.   
> 4. 说了这么多,好像还只是停留在JDBC这一层,更深一步的内容,还待继续分析,先分篇做总结.  