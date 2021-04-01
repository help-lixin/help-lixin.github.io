---
layout: post
title: 'ElasticSearch Java Client案例(十九)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 创建index和mapping信息
> ik_max_word:是IK分词器.(请参照前面的内容,安装好)  

```
PUT /products
{
  "settings": {
    "index":{
      "number_of_shards":"5",
      "number_of_replicas":"1"
    }
  },
  "mappings": {
    "properties":{
      "id": { "type": "integer"  },
      "name" : { "type": "text" , "analyzer" : "ik_max_word"  },
      "catalog" : { "type" : "integer"  },
      "catalog_name" : { "type" : "keyword"},
	  "price" : { "type" : "double" },
	  "number" : { "type" : "integer" },
	  "description" : { "type" : "text" , "analyzer" : "ik_max_word"  }
    }
  }
}
```
### (2). 构造手机数据
> 注意:需要预留最后一行回车.

```
POST _bulk
{"index" : { "_index":"products","_id":1 }}
{"id" : 1 , "name" : "Apple iPhone 11 (A2223) 128GB 黑色 移动联通电信4G手机 双卡双待" , "catalog" : 17 , "catalog_name" : "手机" , "price" : 4899.00 , "number" : 100 ,"description":"<ul class='parameter2 p-parameter-list'><li title='AppleiPhone 11'>商品名称：AppleiPhone 11</li><li title='100008348542'>商品编号：100008348542</li><li title='368.00g'>商品毛重：368.00g</li><li title='中国大陆'>商品产地：中国大陆</li><li title='其他'>CPU型号：其他</li><li title='其他'>运行内存：其他</li><li title='128GB'>机身存储：128GB</li><li title='不支持存储卡'>存储卡：不支持存储卡</li><li title='后置双摄'>摄像头数量：后置双摄</li><li title='1200万像素'>后摄主摄像素：1200万像素</li><li title='1200万像素'>前摄主摄像素：1200万像素</li><li title='6.1英寸'>主屏幕尺寸（英寸）：6.1英寸</li><li title='其它分辨率'>分辨率：其它分辨率</li><li title='其它屏幕比例'>屏幕比例：其它屏幕比例</li><li title='刘海屏'>屏幕前摄组合：刘海屏</li><li title='其他'>充电器：其他</li><li title='人脸识别'>热点：人脸识别</li><li title='语音命令'>特殊功能：语音命令</li><li title='iOS(Apple)'>操作系统：iOS(Apple)</li></ul>" }
{"index" : { "_index":"products","_id":2 }}
{"id" : 2 , "name" : "Redmi 9A 5000mAh大电量 大屏幕大字体大音量 1300万AI相机 八核处理器 人脸解锁 4GB+64GB 砂石黑 游戏智能手机 小米 红米" , "catalog" : 17 , "catalog_name" : "手机" , "price" : 599.00 , "number" : 150 ,"description":"<ul class='parameter2 p-parameter-list'><li title='小米Redmi 9A'>商品名称：小米Redmi 9A</li><li title='100014348478'>商品编号：100014348478</li><li title='367.00g'>商品毛重：367.00g</li><li title='中国大陆'>商品产地：中国大陆</li><li title='其他'>CPU型号：其他</li><li title='4GB'>运行内存：4GB</li><li title='64GB'>机身存储：64GB</li><li title='支持MicroSD(TF)'>存储卡：支持MicroSD(TF)</li><li title='后置单摄'>摄像头数量：后置单摄</li><li title='1300万像素'>后摄主摄像素：1300万像素</li><li title='500万像素'>前摄主摄像素：500万像素</li><li title='6.53英寸'>主屏幕尺寸（英寸）：6.53英寸</li><li title='其它分辨率'>分辨率：其它分辨率</li><li title='其它屏幕比例'>屏幕比例：其它屏幕比例</li><li title='水滴屏'>屏幕前摄组合：水滴屏</li><li title='5V/2A'>充电器：5V/2A</li><li title='超大字体，超大音量'>特殊功能：超大字体，超大音量</li><li title='<90%'>屏占比：&lt;90%</li><li title='Android(安卓)'>操作系统：Android(安卓)</li></ul>" }
{"index" : { "_index":"products","_id":3 }}
{"id" : 3 , "name" : "Redmi Note 9 5G 天玑800U  18W快充 4800万超清三摄 云墨灰 6GB+128GB 游戏智能手机 小米 红米" , "catalog" : 17 , "catalog_name" : "手机" , "price" : 1299.00 , "number" : 300 ,"description":"<ul class='parameter2 p-parameter-list'><li title='小米Redmi  Note9 5G'>商品名称：小米Redmi  Note9 5G</li><li title='100016813958'>商品编号：100016813958</li><li title='460.00g'>商品毛重：460.00g</li><li title='中国大陆'>商品产地：中国大陆</li><li title='天玑800U'>CPU型号：天玑800U</li><li title='6GB'>运行内存：6GB</li><li title='128GB'>机身存储：128GB</li><li title='支持MicroSD(TF)'>存储卡：支持MicroSD(TF)</li><li title='后置三摄'>摄像头数量：后置三摄</li><li title='4800万像素'>后摄主摄像素：4800万像素</li><li title='1300万像素'>前摄主摄像素：1300万像素</li><li title='6.53英寸'>主屏幕尺寸（英寸）：6.53英寸</li><li title='全高清FHD+'>分辨率：全高清FHD+</li><li title='19.1~19.5:9'>屏幕比例：19.1~19.5:9</li><li title='其他'>屏幕前摄组合：其他</li><li title='其他'>充电器：其他</li><li title='18-19W'>充电功率：18-19W</li><li title='≥90%'>屏占比：≥90%</li><li title='Android(安卓)'>操作系统：Android(安卓)</li></ul>" }
{"index" : { "_index":"products","_id":4 }}
{"id" : 4 , "name" : "Redmi Note 9 Pro 5G 一亿像素 骁龙750G 33W快充 120Hz刷新率 碧海星辰 8GB+256GB 游戏智能手机 小米 红米" , "catalog" : 17 , "catalog_name" : "手机" , "price" : 1999.00 , "number" : 80 ,"description":"<ul class='parameter2 p-parameter-list'><li title='小米Redmi Note9 Pro'>商品名称：小米Redmi Note9 Pro</li><li title='100016799350'>商品编号：100016799350</li><li title='500.00g'>商品毛重：500.00g</li><li title='中国大陆'>商品产地：中国大陆</li><li title='骁龙750G'>CPU型号：骁龙750G</li><li title='8GB'>运行内存：8GB</li><li title='256GB'>机身存储：256GB</li><li title='支持MicroSD(TF)'>存储卡：支持MicroSD(TF)</li><li title='后置四摄'>摄像头数量：后置四摄</li><li title='1亿像素'>后摄主摄像素：1亿像素</li><li title='1600万像素'>前摄主摄像素：1600万像素</li><li title='6.67英寸'>主屏幕尺寸（英寸）：6.67英寸</li><li title='全高清FHD+'>分辨率：全高清FHD+</li><li title='19.6~20:9'>屏幕比例：19.6~20:9</li><li title='盲孔屏'>屏幕前摄组合：盲孔屏</li><li title='其他'>充电器：其他</li><li title='人脸识别，快速充电，NFC，5G'>热点：人脸识别，快速充电，NFC，5G</li><li title='超大字体，超大音量，语音命令'>特殊功能：超大字体，超大音量，语音命令</li><li title='≥92%'>屏占比：≥92%</li><li title='Android(安卓)'>操作系统：Android(安卓)</li><li title='发烧级'>游戏性能：发烧级</li><li title='30-39W'>充电功率：30-39W</li></ul>" }
{"index" : { "_index":"products","_id":5 }}
{"id" : 5 , "name" : "realme 真我Q2 4800万像素 120Hz畅速屏 双5G天玑800U 冲浪蓝孩 6GB+128GB 30W闪充 手机 OPPO提供售后支持" , "catalog" : 17 , "catalog_name" : "手机" , "price" : 1238.00 , "number" : 1000 ,"description":"<ul class='parameter2 p-parameter-list'><li title='真我Q2'>商品名称：真我Q2</li><li title='100008900999'>商品编号：100008900999</li><li title='485.00g'>商品毛重：485.00g</li><li title='中国大陆'>商品产地：中国大陆</li><li title='其他'>CPU型号：其他</li><li title='6GB'>运行内存：6GB</li><li title='128GB'>机身存储：128GB</li><li title='支持MicroSD(TF)'>存储卡：支持MicroSD(TF)</li><li title='后置三摄'>摄像头数量：后置三摄</li><li title='4800万像素'>后摄主摄像素：4800万像素</li><li title='1600万像素'>前摄主摄像素：1600万像素</li><li title='6.5英寸'>主屏幕尺寸（英寸）：6.5英寸</li><li title='全高清FHD+'>分辨率：全高清FHD+</li><li title='19.6~20:9'>屏幕比例：19.6~20:9</li><li title='盲孔屏'>屏幕前摄组合：盲孔屏</li><li title='5V6A'>充电器：5V6A</li><li title='Android(安卓)'>操作系统：Android(安卓)</li></ul>" }
{"index" : { "_index":"products","_id":6 }}
{"id" : 6 , "name" : "Redmi Note 9 4G 6000mAh大电池 骁龙662处理器  18W快充 羽墨黑 6GB+128GB 游戏智能手机 小米 红米" , "catalog" : 17 , "catalog_name" : "手机" , "price" : 1099.00 , "number" : 1200 ,"description":"<ul class='parameter2 p-parameter-list'><li title='小米Redmi Note 9 4G'>商品名称：小米Redmi Note 9 4G</li><li title='100016784100'>商品编号：100016784100</li><li title='420.00g'>商品毛重：420.00g</li><li title='中国大陆'>商品产地：中国大陆</li><li title='骁龙662'>CPU型号：骁龙662</li><li title='6GB'>运行内存：6GB</li><li title='128GB'>机身存储：128GB</li><li title='支持MicroSD(TF)'>存储卡：支持MicroSD(TF)</li><li title='后置三摄'>摄像头数量：后置三摄</li><li title='4800万像素'>后摄主摄像素：4800万像素</li><li title='800万像素'>前摄主摄像素：800万像素</li><li title='6.53英寸'>主屏幕尺寸（英寸）：6.53英寸</li><li title='全高清FHD+'>分辨率：全高清FHD+</li><li title='19.1~19.5:9'>屏幕比例：19.1~19.5:9</li><li title='水滴屏'>屏幕前摄组合：水滴屏</li><li title='其他'>充电器：其他</li><li title='18-19W'>充电功率：18-19W</li><li title='≥90%'>屏占比：≥90%</li><li title='Android(安卓)'>操作系统：Android(安卓)</li></ul>" }
{"index" : { "_index":"products","_id":7 }}
{"id" : 7 , "name" : "金立 K7 5000mAh大电池 后置三摄 6.53英寸水滴屏 微信8开 全网通5G 魅海蓝 6+64GB" , "catalog" : 17 , "catalog_name" : "手机" , "price" : 999.00 , "number" : 2000 ,"description":"<ul class='parameter2 p-parameter-list'><li title='金立（GiONEE）金立K7'>商品名称：金立（GiONEE）金立K7</li><li title='10026862980582'>商品编号：10026862980582</li><li title='450.00g'>商品毛重：450.00g</li><li title='其他'>CPU型号：其他</li><li title='6GB'>运行内存：6GB</li><li title='64GB'>机身存储：64GB</li><li title='支持MicroSD(TF)'>存储卡：支持MicroSD(TF)</li><li title='后置三摄'>摄像头数量：后置三摄</li><li title='1600万像素'>后摄主摄像素：1600万像素</li><li title='1300万像素'>前摄主摄像素：1300万像素</li><li title='6.53英寸'>主屏幕尺寸（英寸）：6.53英寸</li><li title='高清HD+'>分辨率：高清HD+</li><li title='其它屏幕比例'>屏幕比例：其它屏幕比例</li><li title='水滴屏'>屏幕前摄组合：水滴屏</li><li title='5V/2A'>充电器：5V/2A</li><li title='Android(安卓)'>操作系统：Android(安卓)</li></ul>" }
{"index" : { "_index":"products","_id":8 }}
{"id" : 8 , "name" : "Apple iPhone 12 (A2404) 128GB 黑色 支持移动联通电信5G 双卡双待手机" , "catalog" : 17 , "catalog_name" : "手机" , "price" : 6799.00 , "number" : 500 ,"description":"<ul class='parameter2 p-parameter-list'><li title='AppleiPhone 12'>商品名称：AppleiPhone 12</li><li title='100016034400'>商品编号：100016034400</li><li title='350.00g'>商品毛重：350.00g</li><li title='中国大陆'>商品产地：中国大陆</li><li title='其他'>CPU型号：其他</li><li title='其他'>运行内存：其他</li><li title='128GB'>机身存储：128GB</li><li title='不支持存储卡'>存储卡：不支持存储卡</li><li title='后置双摄'>摄像头数量：后置双摄</li><li title='1200万像素'>后摄主摄像素：1200万像素</li><li title='1200万像素'>前摄主摄像素：1200万像素</li><li title='6.1英寸'>主屏幕尺寸（英寸）：6.1英寸</li><li title='其它分辨率'>分辨率：其它分辨率</li><li title='其它屏幕比例'>屏幕比例：其它屏幕比例</li><li title='其他'>屏幕前摄组合：其他</li><li title='其他'>充电器：其他</li><li title='iOS(Apple)'>操作系统：iOS(Apple)</li></ul>" }

```
### (3). 构造电脑数据
> 注意:需要预留最后一行回车.

```
POST _bulk
{"index" : { "_index":"products","_id":100 }}
{"id" : 100 , "name" : "联想(Lenovo)小新Air15轻薄本英特尔酷睿i5 15.6英寸全面屏笔记本电脑 (i5-1135G7 16G 512G MX450高色域 )银" , "catalog" : 18 , "catalog_name" : "电脑" , "price" : 5999.00 , "number" : 100 ,"description":"<ul class='parameter2 p-parameter-list'><li title='联想小新Air15'>商品名称：联想小新Air15</li><li title='100016086218'>商品编号：100016086218</li><li title='2.56kg'>商品毛重：2.56kg</li><li title='中国大陆'>商品产地：中国大陆</li><li title='独立显卡'>显卡类别：独立显卡</li><li title='全高清屏（1920×1080）'>分辨率：全高清屏（1920×1080）</li><li title='轻薄笔记本'>类型：轻薄笔记本</li><li title='windows 10 带Office'>系统：windows 10 带Office</li><li title='15.1mm—18.0mm'>厚度：15.1mm—18.0mm</li><li title='金属材质'>机身材质：金属材质</li><li title='其他'>显卡芯片供应商：其他</li><li title='15.0英寸-15.9英寸'>屏幕尺寸：15.0英寸-15.9英寸</li><li title='联想-小新Air'>系列：联想-小新Air</li><li title='1.5-2kg'>裸机重量：1.5-2kg</li><li title='100%sRGB'>屏幕色域：100%sRGB</li><li title='无机械硬盘'>机械硬盘：无机械硬盘</li><li title='Intel i5'>处理器：Intel i5</li><li title='其他'>屏幕刷新率：其他</li><li title='＞12小时'>待机时长：＞12小时</li><li title='指纹识别，全面屏，长续航电池'>特性：指纹识别，全面屏，长续航电池</li><li title='MX450'>显卡型号：MX450</li><li title='16GB'>内存容量：16GB</li><li title='512GB'>固态硬盘（SSD）：512GB</li><li title='银色'>颜色：银色</li></ul>" }
{"index" : { "_index":"products","_id":101 }}
{"id" : 101 , "name" : "联想小新Air15 2021锐龙R7 金属超轻薄笔记本电脑 高色域全面屏 学生网课办公 设计师游戏本 八核R7-4800U 16G 512G固态 标配 高色域全面屏【低蓝光护眼无闪烁】" , "catalog" : 18 , "catalog_name" : "电脑" , "price" : 5799.00 , "number" : 200 ,"description":"<ul class='parameter2 p-parameter-list'><li title='联想（Lenovo）Air15 2021款'>商品名称：联想（Lenovo）Air15 2021款</li><li title='33950552707'>商品编号：33950552707</li><li title='联想华东授权专卖店'>店铺： <a href='//lxnys.jd.com' target='_blank'>联想华东授权专卖店</a></li><li title='10.0kg'>商品毛重：10.0kg</li><li title='集成显卡'>显卡类别：集成显卡</li><li title='全高清屏（1920×1080）'>分辨率：全高清屏（1920×1080）</li><li title='高性能轻薄笔记本'>类型：高性能轻薄笔记本</li><li title='windows 10 带Office'>系统：windows 10 带Office</li><li title='15.1mm—18.0mm'>厚度：15.1mm—18.0mm</li><li title='金属材质'>机身材质：金属材质</li><li title='AMD'>显卡芯片供应商：AMD</li><li title='15.0英寸-15.9英寸'>屏幕尺寸：15.0英寸-15.9英寸</li><li title='联想-小新Air'>系列：联想-小新Air</li><li title='1.5-2kg'>裸机重量：1.5-2kg</li><li title='100%sRGB'>屏幕色域：100%sRGB</li><li title='无机械硬盘'>机械硬盘：无机械硬盘</li><li title='两年质保'>优选服务：两年质保</li><li title='AMD R7'>处理器：AMD R7</li><li title='60HZ'>屏幕刷新率：60HZ</li><li title='＞12小时'>待机时长：＞12小时</li><li title='硬盘可扩展，指纹识别，长续航电池'>特性：硬盘可扩展，指纹识别，长续航电池</li><li title='其他'>显卡型号：其他</li><li title='16GB'>内存容量：16GB</li><li title='512GB'>固态硬盘（SSD）：512GB</li><li title='灰色'>颜色：灰色</li></ul>" }
{"index" : { "_index":"products","_id":102 }}
{"id" : 102 , "name" : "戴尔DELL成就3400 14英寸轻薄性能商务全面屏笔记本电脑(11代i5-1135G7 16G 512G)一年上门+7x24远程服务" , "catalog" : 18 , "catalog_name" : "电脑" , "price" : 4999.00 , "number" : 80 ,"description":"<ul class='parameter2 p-parameter-list'><li title='戴尔Vostro 14-3400'>商品名称：戴尔Vostro 14-3400</li><li title='100016777690'>商品编号：100016777690</li><li title='2.4kg'>商品毛重：2.4kg</li><li title='中国大陆'>商品产地：中国大陆</li><li title='集成显卡'>显卡类别：集成显卡</li><li title='戴尔-成就系列'>系列：戴尔-成就系列</li><li title='全高清屏（1920×1080）'>分辨率：全高清屏（1920×1080）</li><li title='轻薄笔记本'>类型：轻薄笔记本</li><li title='Intel i5'>处理器：Intel i5</li><li title='windows 10 带Office'>系统：windows 10 带Office</li><li title='一年质保'>优选服务：一年质保</li><li title='18.1mm—20.0mm'>厚度：18.1mm—20.0mm</li><li title='其他'>屏幕刷新率：其他</li><li title='16GB'>内存容量：16GB</li><li title='512GB'>固态硬盘（SSD）：512GB</li><li title='其他'>显卡型号：其他</li><li title='14.0英寸-14.9英寸'>屏幕尺寸：14.0英寸-14.9英寸</li><li title='NVIDIA'>显卡芯片供应商：NVIDIA</li><li title='无机械硬盘'>机械硬盘：无机械硬盘</li></ul>" }
{"index" : { "_index":"products","_id":103 }}
{"id" : 103 , "name" : "联想(Lenovo)小新Air14锐龙版14英寸全面屏轻薄笔记本电脑(6核12线程R5-5500U 16G 512G 高色域)深空灰" , "catalog" : 18 , "catalog_name" : "电脑" , "price" : 4699.00 , "number" : 1000 ,"description":"<ul class='parameter2 p-parameter-list'><li title='联想小新Air'>商品名称：联想小新Air</li><li title='100018584132'>商品编号：100018584132</li><li title='2.16kg'>商品毛重：2.16kg</li><li title='中国大陆'>商品产地：中国大陆</li><li title='14.0英寸-14.9英寸'>屏幕尺寸：14.0英寸-14.9英寸</li><li title='联想-小新Air'>系列：联想-小新Air</li><li title='全高清屏（1920×1080）'>分辨率：全高清屏（1920×1080）</li><li title='AMD R5'>处理器：AMD R5</li><li title='轻薄笔记本'>类型：轻薄笔记本</li><li title='16GB'>内存容量：16GB</li><li title='windows 10 带Office'>系统：windows 10 带Office</li><li title='金属+复合材质'>机身材质：金属+复合材质</li><li title='15.1mm—18.0mm'>厚度：15.1mm—18.0mm</li><li title='其他'>屏幕刷新率：其他</li><li title='512GB'>固态硬盘（SSD）：512GB</li><li title='集成显卡'>显卡类别：集成显卡</li><li title='其他'>显卡型号：其他</li><li title='全面屏，WiFi 6'>特性：全面屏，WiFi 6</li><li title='其他'>显卡芯片供应商：其他</li><li title='无机械硬盘'>机械硬盘：无机械硬盘</li></ul>" }

```
### (4). 添加依赖
```
<dependency>
	<groupId>org.elasticsearch.client</groupId>
	<artifactId>elasticsearch-rest-client</artifactId>
	<version>7.1.0</version>
</dependency>
<dependency>
	<groupId>org.elasticsearch.client</groupId>
	<artifactId>elasticsearch-rest-high-level-client</artifactId>
	<version>7.1.0</version>
</dependency>
<dependency>
	<groupId>junit</groupId>
	<artifactId>junit</artifactId>
	<version>4.12</version>
</dependency>
<dependency>
	<groupId>com.fasterxml.jackson.core</groupId>
	<artifactId>jackson-databind</artifactId>
	<version>2.9.4</version>
</dependency>
```
### (5). 低级客户端案例
```
package help.lixin.es;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

import org.apache.http.HttpHost;
import org.apache.http.util.EntityUtils;
import org.elasticsearch.client.Node;
import org.elasticsearch.client.Request;
import org.elasticsearch.client.Response;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

/**
 * 低级客户端使用
 * @author lixin
 *
 */
public class TestEsLowRest {

	private RestClient restClient;

	private ObjectMapper objectMapper = new ObjectMapper();

	@Before
	public void init() {
		RestClientBuilder restClientBuilder = RestClient.builder( //
				new HttpHost("10.211.55.100", 9200, "http"), //
				new HttpHost("10.211.55.101", 9200, "http"), //
				new HttpHost("10.211.55.102", 9200, "http"));
		restClientBuilder.setFailureListener(new RestClient.FailureListener() {
			@Override
			public void onFailure(Node node) {
				System.out.println("error:" + node);
			}
		});
		restClient = restClientBuilder.build();
	}

	@Test
	public void testClusterState() throws Exception {
		Request request = new Request("GET", "_cluster/state");
		request.addParameter("pretty", "true");
		Response response = restClient.performRequest(request);
		System.out.println(response.getStatusLine());
		System.out.println(EntityUtils.toString(response.getEntity()));
	}

	@Test
	public void testQueryForDocumentId() throws Exception {
		Request request = new Request("GET", "/products/_doc/1");
		request.addParameter("pretty", "true");
		Response response = restClient.performRequest(request);
		System.out.println(response.getStatusLine());
		System.out.println(EntityUtils.toString(response.getEntity()));
	}

	@Test
	public void testSaveDocument() throws Exception {
		int id = 104;
		Request request = new Request("POST", "/products/_doc/" + id);

		String desc = "<ul class='parameter2 p-parameter-list'><li title='联想Yoga'>商品名称：联想Yoga</li><li title='100018630354'>商品编号：100018630354</li><li title='2.38kg'>商品毛重：2.38kg</li><li title='中国大陆'>商品产地：中国大陆</li><li title='14.0英寸-14.9英寸'>屏幕尺寸：14.0英寸-14.9英寸</li><li title='联想 - YOGA'>系列：联想 - YOGA</li><li title='超高清屏（2K/2.5K/3K/4K）'>分辨率：超高清屏（2K/2.5K/3K/4K）</li><li title='Intel i5'>处理器：Intel i5</li><li title='轻薄笔记本'>类型：轻薄笔记本</li><li title='16GB'>内存容量：16GB</li><li title='windows 10 带Office'>系统：windows 10 带Office</li><li title='金属材质'>机身材质：金属材质</li><li title='15.0mm及以下'>厚度：15.0mm及以下</li><li title='90Hz'>屏幕刷新率：90Hz</li><li title='512GB'>固态硬盘（SSD）：512GB</li><li title='集成显卡'>显卡类别：集成显卡</li><li title='其他'>显卡型号：其他</li><li title='全面屏，雷电接口，面部识别'>特性：全面屏，雷电接口，面部识别</li><li title='其他'>显卡芯片供应商：其他</li><li title='无机械硬盘'>机械硬盘：无机械硬盘</li></ul>";
		Map<String, Object> params = new HashMap<String, Object>();
		params.put("id", id);
		params.put("name", "戴尔笔记本电脑dell灵越15-3501 15.6英寸高性能轻薄商务笔记本电脑(11代英特尔酷睿i5-1135G7 16G 512GSSD)");
		params.put("catalog", 18);
		params.put("catalog_name", "电脑");
		params.put("price", 4299.00D);
		params.put("number", 202);
		params.put("description", desc);

		// 设置请求体信息
		request.setJsonEntity(objectMapper.writeValueAsString(params));

		Response response = restClient.performRequest(request);
		System.out.println(response.getStatusLine());
		System.out.println(EntityUtils.toString(response.getEntity()));
	}

	@Test
	public void testQueryDescription() throws Exception {
		String searchJson = "{\"query\": { \"match\": { \"description\": \" 1600万像素 \" } } }";
		Request request = new Request("POST", "/products/_doc/_search");
		request.addParameter("pretty", "true");
		request.setJsonEntity(searchJson);

		Response response = restClient.performRequest(request);
		System.out.println(response.getStatusLine());
		System.out.println(EntityUtils.toString(response.getEntity()));
	}

	@After
	public void close() {
		if (null != restClient) {
			try {
				restClient.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}

}
```
### (6). 高级客户端案例
```
package help.lixin.es;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

import org.apache.http.HttpHost;
import org.elasticsearch.action.get.GetRequest;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHits;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import com.fasterxml.jackson.databind.ObjectMapper;

/**
 * 高级客户端使用
 * 
 * @author lixin
 *
 */
public class TestEsHighRest {
	private RestHighLevelClient client;

	private ObjectMapper mapper = new ObjectMapper();

	@Before
	public void init() {
		RestClientBuilder restClientBuilder = RestClient.builder( //
				new HttpHost("10.211.55.100", 9200, "http"), //
				new HttpHost("10.211.55.101", 9200, "http"), //
				new HttpHost("10.211.55.102", 9200, "http"));

		this.client = new RestHighLevelClient(restClientBuilder);
	}

	@Test
	public void testSave() throws Exception {
		int id = 105;
		String desc = "<ul class='parameter2 p-parameter-list'><li title='联想Yoga'>商品名称：联想Yoga</li><li title='100018630355'>商品编号：100018630355</li><li title='2.38kg'>商品毛重：2.38kg</li><li title='中国大陆'>商品产地：中国大陆</li><li title='14.0英寸-14.9英寸'>屏幕尺寸：14.0英寸-14.9英寸</li><li title='联想 - YOGA'>系列：联想 - YOGA</li><li title='超高清屏（2K/2.5K/3K/4K）'>分辨率：超高清屏（2K/2.5K/3K/4K）</li><li title='Intel i5'>处理器：Intel i5</li><li title='轻薄笔记本'>类型：轻薄笔记本</li><li title='16GB'>内存容量：16GB</li><li title='windows 10 带Office'>系统：windows 10 带Office</li><li title='金属材质'>机身材质：金属材质</li><li title='15.0mm及以下'>厚度：15.0mm及以下</li><li title='90Hz'>屏幕刷新率：90Hz</li><li title='512GB'>固态硬盘（SSD）：512GB</li><li title='集成显卡'>显卡类别：集成显卡</li><li title='其他'>显卡型号：其他</li><li title='全面屏，雷电接口，面部识别'>特性：全面屏，雷电接口，面部识别</li><li title='其他'>显卡芯片供应商：其他</li><li title='无机械硬盘'>机械硬盘：无机械硬盘</li></ul>";
		Map<String, Object> params = new HashMap<String, Object>();
		params.put("id", id);
		params.put("name", "戴尔笔记本电脑dell灵越15-3501 16.6英寸高性能轻薄商务笔记本电脑(11代英特尔酷睿i5-1135G7 16G 512GSSD)");
		params.put("catalog", 18);
		params.put("catalog_name", "电脑");
		params.put("price", 4999.00D);
		params.put("number", 222);
		params.put("description", desc);

		IndexRequest indexRequest = new IndexRequest("products", "_doc", String.valueOf(id)).source(params);
		IndexResponse response = client.index(indexRequest, RequestOptions.DEFAULT);
		System.out.println("id->" + response.getId());
		System.out.println("index->" + response.getIndex());
		System.out.println("version->" + response.getVersion());
		System.out.println("type->" + response.getType());
		System.out.println("result->" + response.getResult());
		System.out.println("shardInfo->" + response.getShardInfo());
	}

	@Test
	public void testQueryForDocumentId() throws Exception {
		GetRequest request = new GetRequest("products", "_doc", "105");
		GetResponse response = client.get(request, RequestOptions.DEFAULT);
		System.out.println("id->" + response.getId());
		System.out.println("index->" + response.getIndex());
		System.out.println("version->" + response.getVersion());
		System.out.println("type->" + response.getType());
		System.out.println("source->" + response.getSourceAsString());
	}
	
	@Test
	public void testQueryDesc() throws Exception  {
		SearchRequest searchRequest = new SearchRequest("products");
		// searchRequest.types("_doc");
		
		SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
		sourceBuilder.query(QueryBuilders.matchQuery("description", "1600万像素"));
		sourceBuilder.from(0);
		sourceBuilder.size(100);
		sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
		
		searchRequest.source(sourceBuilder);
	
		SearchResponse response = client.search(searchRequest, RequestOptions.DEFAULT);
		System.out.println("搜索到:" + response.getHits().getTotalHits().value + " 条数据");
		SearchHits hits = response.getHits();
		for(SearchHit hit: hits) {
			String json = hit.getSourceAsString();
			System.out.println(json);
		}
	}

	@After
	public void close() {
		if (null != client) {
			try {
				client.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}
}

```
