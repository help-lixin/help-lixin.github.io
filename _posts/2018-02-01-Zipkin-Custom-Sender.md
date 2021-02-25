---
layout: post
title: 'Zipkin 自定义Sender(二)'
date: 2018-02-01
author: 李新
tags: Zipkin
---

### (1). 概述
> 在前面的分析,我们知道:Sender主要职责是通过传输层,发送数据给:Collector(即:zipkin-server).   
> 有这样一个需求:  
> 1. 把参生的跨度数据发送给:log/rocket-mq/redis...   
> 2. 读取log/rocket-mq/redis中的数据.  
> 3. 向zipkin-server提交跨度数据.   

### (2). 自定义CustomSender写日志
```
package help.lixin.zipkin;

import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.util.List;

import zipkin2.Call;
import zipkin2.Callback;
import zipkin2.codec.Encoding;
import zipkin2.reporter.Sender;

public class CustomSender extends Sender {
	private int messageMaxBytes = 500000;
	private Encoding encoding = Encoding.JSON;

	public void setEncoding(Encoding encoding) {
		if (null != encoding) {
			this.encoding = encoding;
		}
	}

	public void setMessageMaxBytes(int messageMaxBytes) {
		if (messageMaxBytes >= 0) {
			this.messageMaxBytes = messageMaxBytes;
		} else {
			messageMaxBytes = 500000;
		}
	}

	@Override
	public Encoding encoding() {
		return encoding;
	}

	@Override
	public int messageMaxBytes() {
		return messageMaxBytes;
	}

	@Override
	public int messageSizeInBytes(List<byte[]> encodedSpans) {
		return encoding.listSizeInBytes(encodedSpans);
	}

	@Override
	public Call<Void> sendSpans(List<byte[]> encodedSpans) {
		return new LogCall<Void>(encodedSpans);
	}

	class LogCall<Void> extends Call<Void> {
		private final List<byte[]> encodedSpans;

		public LogCall(List<byte[]> encodedSpans) {
			this.encodedSpans = encodedSpans;
		}

		@Override
		public Void execute() throws IOException {
			for (int i = 0, length = encodedSpans.size(); i < length;) {
				byte[] span = encodedSpans.get(i++);
				try {
					// *******************************************
					// 把byte转换成字符串.
					// *******************************************
					String spanStr = new String(span, "UTF-8");
					// [
					//		{ 
					//		"traceId":"da0332f26d0100dc",
					//		"id":"da0332f26d0100dc",
					//		"name":"brave-span",
					//		"timestamp":1614238604327714,
					//		"duration":10519,
					//		"localEndpoint": {
					//			"serviceName":"brave-service",
					//			"ipv4":"172.17.0.253"
					//		},
					//		"tags":{
					//			"request-id":"2826d6f7-b4d6-410f-a644-f4338f8eca72"}
					//		}
					// 	]
					System.out.println(spanStr);
				} catch (UnsupportedEncodingException e) {
					e.printStackTrace();
				}
			}
			return null;
		}

		@Override
		public void enqueue(Callback<Void> callback) {
		}

		@Override
		public void cancel() {
		}

		@Override
		public boolean isCanceled() {
			return false;
		}

		@Override
		public Call<Void> clone() {
			return this;
		}
	}
}
```
### (3). 向zipkin-server汇报跨度信息
```
// 产生的跨度信息如下:
[
    { 
    "traceId":"da0332f26d0100dc",
    "id":"da0332f26d0100dc",
    "name":"brave-span",
    "timestamp":1614238604327714,
    "duration":10519,
    "localEndpoint": {
        "serviceName":"brave-service",
        "ipv4":"172.17.0.253"
    },
    "tags":{
        "request-id":"2826d6f7-b4d6-410f-a644-f4338f8eca72"}
    }
]

```
!["通过Rest API向zipkin-server汇报数据"](/assets/zipkin/imgs/zipkin-rest-report-data.jpg)

### (4). zipkin-server验证结果
!["zipkin-server"](/assets/zipkin/imgs/zipkin-ui.jpg)