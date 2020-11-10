---
layout: post
title: 'Canal Client'
date: 2019-10-01
author: 李新
tags: Canal
---

### (1). pom.xml(Canal Client)
```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin.canal</groupId>
	<artifactId>canal-demo</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>canal-demo</name>
	<url>http://maven.apache.org</url>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<java_source_version>1.8</java_source_version>
		<java_target_version>1.8</java_target_version>
		<file_encoding>UTF-8</file_encoding>
	</properties>

	<dependencies>
		<dependency>
			<groupId>com.alibaba.otter</groupId>
			<artifactId>canal.client</artifactId>
			<version>1.1.4</version>
		</dependency>
	</dependencies>
</project>

```
### (2). Canal Client
```
package help.lixin.canal;

import java.io.Serializable;
import java.net.InetSocketAddress;
import java.util.List;

import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.common.utils.AddressUtils;
import com.alibaba.otter.canal.protocol.CanalEntry.Column;
import com.alibaba.otter.canal.protocol.CanalEntry.Entry;
import com.alibaba.otter.canal.protocol.CanalEntry.EntryType;
import com.alibaba.otter.canal.protocol.CanalEntry.EventType;
import com.alibaba.otter.canal.protocol.CanalEntry.RowChange;
import com.alibaba.otter.canal.protocol.CanalEntry.RowData;
import com.alibaba.otter.canal.protocol.Message;

public class CanalClient {
	public static void main(String args[]) {
		
		// canal.mq.topic=example
		// 创建链接
		CanalConnector connector = CanalConnectors
				.newSingleConnector(new InetSocketAddress(AddressUtils.getHostIp(), 11111), "example", "", "");
		int batchSize = 1000;
		int emptyCount = 0;
		try {
			connector.connect();
			connector.subscribe(".*\\..*");
			connector.rollback();
			int totalEmptyCount = 120;
			while (emptyCount < totalEmptyCount) {
				Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据,但是不提交
				long batchId = message.getId();
				int size = message.getEntries().size();
				// 空统计
				if (batchId == -1 || size == 0) {
					emptyCount++;
					System.out.println("empty count : " + emptyCount);
					try {
						Thread.sleep(1000);
					} catch (InterruptedException e) {
					}
				} else {
					// 重置空统计
					emptyCount = 0;
					// System.out.printf("message[batchId=%s,size=%s] \n", batchId, size);
					printEntry(message.getEntries());
				}
				connector.ack(batchId); // 提交确认
				// connector.rollback(batchId); // 处理失败, 回滚数据
			}

			System.out.println("empty too many times, exit");
		} finally {
			connector.disconnect();
		}
	}

	private static void printEntry(List<Entry> entrys) {
		for (Entry entry : entrys) {
			// 事务相关的类型跳过
			if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN
					|| entry.getEntryType() == EntryType.TRANSACTIONEND) {
				continue;
			}
			
			RowChange rowChage = null;
			try {
				rowChage = RowChange.parseFrom(entry.getStoreValue());
			} catch (Exception e) {
				throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(),
						e);
			}

			EventType eventType = rowChage.getEventType();
			System.out.println(String.format("================&gt; binlog[%s:%s] , name[%s,%s] , eventType : %s",
					entry.getHeader().getLogfileName(), entry.getHeader().getLogfileOffset(),
					entry.getHeader().getSchemaName(), entry.getHeader().getTableName(), eventType));

			for (RowData rowData : rowChage.getRowDatasList()) {
				if (eventType == EventType.DELETE) {
					printColumn(rowData.getBeforeColumnsList());
				} else if (eventType == EventType.INSERT) {
					printColumn(rowData.getAfterColumnsList());
				} else { // 更新
					System.out.println("-------&gt; before");
					printColumn(rowData.getBeforeColumnsList());
					System.out.println("-------&gt; after");
					printColumn(rowData.getAfterColumnsList());
				}
			}
		}
	}

	private static void printColumn(List<Column> columns) {
		for (Column column : columns) {
			System.out.println(column.getName() + " : " + column.getValue() + "    update=" + column.getUpdated());
		}
	}
}

// CREATE TABLE user ( id INT PRIMARY KEY AUTO_INCREMENT,nick VARCHAR(20),phone VARCHAR(20),password VARCHAR(30),email VARCHAR(30),account VARCHAR(30) );
// INSERT INTO user(nick,phone,password,email,account) VALUES('张三','12345678','12345678','123@126.com','admin');
// INSERT INTO user(nick,phone,password,email,account) VALUES('李四','12345678','12345678','123@126.com','client');
// INSERT INTO user(nick,phone,password,email,account) VALUES('王五','12345678','12345678','123@126.com','super-admin');

// UPDATE user SET password = '88888888' WHERE account = 'super-admin';
// 
class UserDTO implements Serializable {
	private static final long serialVersionUID = -7486086785079929643L;
	private Long id;
	private String nick;
	private String phone;
	private String password;
	private String email;
	private String account;

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	public String getNick() {
		return nick;
	}

	public void setNick(String nick) {
		this.nick = nick;
	}

	public String getPhone() {
		return phone;
	}

	public void setPhone(String phone) {
		this.phone = phone;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	public String getEmail() {
		return email;
	}

	public void setEmail(String email) {
		this.email = email;
	}

	public String getAccount() {
		return account;
	}

	public void setAccount(String account) {
		this.account = account;
	}
}

```
### (3). 

### (4). 

### (5). 

### (6). 

### (7). 

### (8). 

### (9). 

### (10). 

### (11). 
