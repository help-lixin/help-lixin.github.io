---
layout: post
title: 'ElasticSearch 订单与订单详细真实案例(十八)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 需求
> 以订单和订单详细为模型,把数据同步到ES中,进行检索.  

!["订单模块应用ES"](/assets/elasticsearch/imgs/es-order.png)
### (2). 创建索引库
```
DELETE /order_index

# 创建索引库
PUT /order_index
{
  "settings": {
    "index":{
      "number_of_shards":"1",
      "number_of_replicas":"0"
    }
  },
  "mappings": {
    "properties":{
		"order" : {
			"properties":{
				"order_id": { "type": "long"  },
				"order_no": { "type": "text" , "analyzer" : "ik_max_word"   },
				"customer_id": { "type": "long"  },
				"customer_name": { "type": "keyword"  },
				"customer_phone": { "type": "keyword"  },
				"customer_address": { "type": "text" , "analyzer" : "ik_max_word"  },
				"merchant_id": { "type": "long"  },
				"total_money" : { "type" : "double" },
				"order_status" : { "type" : "keyword" }	
			}
		},
		"order_detail" : {
			"properties":{
				"order_detail_id": { "type": "long"  },
				"order_id": { "type": "long"  },
				"goods_id": { "type": "long"  },
				"goods_no": { "type": "text" , "analyzer" : "ik_max_word"   },
				"goods_name" : { "type": "text" , "analyzer" : "ik_max_word"   },
				"goods_img" : { "type" : "keyword" },
				"price" : { "type" : "double" },
				"buy_num" : { "type": "integer"  }
			}
		},
		"relation" : {
			"type" : "join",
			"relations" : {
				"order" : ["order_detail"]
			}
		}
    }
  }
}
```

### (3). 模拟下订单业务

```
package help.lixin.erp;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import help.lixin.erp.entity.OrderDetail;
import help.lixin.erp.entity.Orders;
import help.lixin.erp.service.IOrdersService;

@RunWith(SpringRunner.class)
@SpringBootTest
public class BuyTest {

	@Autowired
	private IOrdersService ordersService;

	@Test
	public void testBuy1() {
		long orderId = 1L;

		List<OrderDetail> details = new ArrayList<OrderDetail>();
		OrderDetail detail1 = new OrderDetail();
		detail1.setOrderDetailId(11L);
		detail1.setOrderId(orderId);
		detail1.setGoodsId(1L);
		detail1.setGoodsName("Apple iPhone 11 (A2223) 128GB 黑色 移动联通电信4G手机 双卡双待");
		detail1.setGoodsNo("202004020000001");
		detail1.setGoodsImg("/goods1.jpg");
		detail1.setPrice(new BigDecimal("4899.00"));
		detail1.setBuyNum(1);

		OrderDetail detail2 = new OrderDetail();
		detail2.setOrderDetailId(22L);
		detail2.setOrderId(orderId);
		detail2.setGoodsId(103L);
		detail2.setGoodsName("联想(Lenovo)小新Air14锐龙版14英寸全面屏轻薄笔记本电脑(6核12线程R5-5500U 16G 512G 高色域)深空灰");
		detail2.setGoodsNo("20200402000000103");
		detail2.setGoodsImg("/goods103.jpg");
		detail2.setPrice(new BigDecimal("4699.00"));
		detail2.setBuyNum(1);

		details.add(detail1);
		details.add(detail2);
		// 计算总金额
		BigDecimal totalMoney = details.stream()
		.map(detail -> detail.getPrice().multiply(new BigDecimal(detail.getBuyNum())))
		.reduce(BigDecimal.ZERO, BigDecimal::add);

		Orders orders = new Orders();
		orders.setOrderId(orderId);
		orders.setOrderNo("O00000000001");
		orders.setVersion(1);
		orders.setOrderStatus("1");

		orders.setCustomerId(77777L);
		orders.setCustomerName("张三");
		orders.setCustomerPhone("13799999999");
		orders.setCustomerAddress("广东省深圳市宝安区");

		orders.setMerchantId(88888L);
		orders.setTotalMoney(totalMoney);
		
		ordersService.createOrder(orders, details);
	}
	
	
	@Test
	public void testBuy2() {
		long orderId = 2L;
		List<OrderDetail> details = new ArrayList<OrderDetail>();
		OrderDetail detail1 = new OrderDetail();
		detail1.setOrderDetailId(33L);
		detail1.setOrderId(orderId);
		detail1.setGoodsId(2L);
		detail1.setGoodsName("Redmi 9A 5000mAh大电量 大屏幕大字体大音量 1300万AI相机 八核处理器 人脸解锁 4GB+64GB 砂石黑 游戏智能手机 小米 红米");
		detail1.setGoodsNo("202004020000002");
		detail1.setGoodsImg("/goods2.jpg");
		detail1.setPrice(new BigDecimal("599.00"));
		detail1.setBuyNum(2);

		OrderDetail detail2 = new OrderDetail();
		detail2.setOrderDetailId(44L);
		detail2.setOrderId(orderId);
		detail2.setGoodsId(3L);
		detail2.setGoodsName("Redmi Note 9 5G 天玑800U  18W快充 4800万超清三摄 云墨灰 6GB+128GB 游戏智能手机 小米 红米");
		detail2.setGoodsNo("202004020000003");
		detail2.setGoodsImg("/goods3.jpg");
		detail2.setPrice(new BigDecimal("1299.00"));
		detail2.setBuyNum(3);

		details.add(detail1);
		details.add(detail2);
		// 计算总金额
		BigDecimal totalMoney = details.stream()
		.map(detail -> detail.getPrice().multiply(new BigDecimal(detail.getBuyNum())))
		.reduce(BigDecimal.ZERO, BigDecimal::add);

		Orders orders = new Orders();
		orders.setOrderId(orderId);
		orders.setOrderNo("O00000000002");
		orders.setVersion(1);
		orders.setOrderStatus("1");

		orders.setCustomerId(66666L);
		orders.setCustomerName("张三");
		orders.setCustomerPhone("13788888888");
		orders.setCustomerAddress("广东省深圳市宝安区");

		orders.setMerchantId(88888L);
		orders.setTotalMoney(totalMoney);
		
		ordersService.createOrder(orders, details);
	}
	
	
	@Test
	public void testBuy3() {
		long orderId = 3L;
		List<OrderDetail> details = new ArrayList<OrderDetail>();
		OrderDetail detail1 = new OrderDetail();
		detail1.setOrderDetailId(55L);
		detail1.setOrderId(orderId);
		detail1.setGoodsId(102L);
		detail1.setGoodsName("戴尔DELL成就3400 14英寸轻薄性能商务全面屏笔记本电脑(11代i5-1135G7 16G 512G)一年上门+7x24远程服务");
		detail1.setGoodsNo("20200402000000102");
		detail1.setGoodsImg("/goods102.jpg");
		detail1.setPrice(new BigDecimal("4999.00"));
		detail1.setBuyNum(1);

		OrderDetail detail2 = new OrderDetail();
		detail2.setOrderDetailId(66L);
		detail2.setOrderId(orderId);
		detail2.setGoodsId(101L);
		detail2.setGoodsName("联想小新Air15 2021锐龙R7 金属超轻薄笔记本电脑 高色域全面屏 学生网课办公 设计师游戏本 八核R7-4800U 16G 512G固态 标配 高色域全面屏【低蓝光护眼无闪烁】");
		detail2.setGoodsNo("20200402000000101");
		detail2.setGoodsImg("/goods101.jpg");
		detail2.setPrice(new BigDecimal("5799.00"));
		detail2.setBuyNum(1);

		details.add(detail1);
		details.add(detail2);
		// 计算总金额
		BigDecimal totalMoney = details.stream()
		.map(detail -> detail.getPrice().multiply(new BigDecimal(detail.getBuyNum())))
		.reduce(BigDecimal.ZERO, BigDecimal::add);

		Orders orders = new Orders();
		orders.setOrderId(orderId);
		orders.setOrderNo("O00000000003");
		orders.setVersion(1);
		orders.setOrderStatus("1");

		orders.setCustomerId(55555L);
		orders.setCustomerName("李四");
		orders.setCustomerPhone("13755555");
		orders.setCustomerAddress("湖南省长沙市");

		orders.setMerchantId(999999L);
		orders.setTotalMoney(totalMoney);
		
		ordersService.createOrder(orders, details);
	}
}
```
### (4). 手动测试,同步MySQL数据到ES
> 注意:父子文档的ID不能相同,否则,会覆盖文档的.

```
package help.lixin.erp;

import java.io.IOException;
import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import org.apache.http.HttpHost;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.elasticsearch.client.RestHighLevelClient;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;

import help.lixin.erp.entity.OrderDetail;
import help.lixin.erp.entity.Orders;
import help.lixin.erp.service.IOrderDetailService;
import help.lixin.erp.service.IOrdersService;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SyncDataToESTest {
	@Autowired
	private IOrdersService ordersService;

	@Autowired
	private IOrderDetailService orderDetailService;

	private RestHighLevelClient client;

	@Before
	public void init() {
		RestClientBuilder restClientBuilder = RestClient.builder( //
				new HttpHost("127.0.0.1", 9200, "http")
		);
		this.client = new RestHighLevelClient(restClientBuilder);
	}

	@Test
	public void testSyncDataToEs() {
		List<Orders> list = ordersService.list();
		list.stream().map(order -> {
			Long orderId = order.getOrderId();
			List<OrderDetail> details = orderDetailService.list(Wrappers.<OrderDetail>query().eq("order_id", orderId));
			order.setDetails(details);
			return order;
		}).forEach(item -> {
			IndexRequest parentIndex = buildOrder(item);
			List<IndexRequest> parentIndexs = Collections.singletonList(parentIndex);
			List<IndexRequest> childIndexs = buildOrderDetails(item);
			execute(parentIndexs);
			execute(childIndexs);
		});
	}

	public void execute(Collection<IndexRequest> indexRequests) {
		indexRequests.stream().forEach(indexRequest -> {
			try {
				IndexResponse response = client.index(indexRequest, RequestOptions.DEFAULT);
				System.out.println("index:" + response.getIndex());
				System.out.println("type:" + response.getType());
				System.out.println("id:" + response.getId());
			} catch (IOException e) {
				throw new RuntimeException(e.getMessage());
			}
		});
	}

	public IndexRequest buildOrder(Orders order) {
		Map<String, Object> relation = new HashMap<String, Object>();
		relation.put("name", "order");

		Map<String, Object> source = new HashMap<String, Object>();
		source.put("order_id", order.getOrderId());

		source.put("order_id", order.getOrderId());
		source.put("order_no", order.getOrderNo());
		source.put("customer_id", order.getCustomerId());
		source.put("customer_name", order.getCustomerName());
		source.put("customer_phone", order.getCustomerPhone());
		source.put("customer_address", order.getCustomerAddress());
		source.put("merchant_id", order.getMerchantId());
		source.put("total_money", order.getTotalMoney());
		source.put("order_status", order.getOrderStatus());

		// setting relation
		source.put("relation", relation);

		IndexRequest indexRequest = new IndexRequest();
		indexRequest.index("order_index");
		indexRequest.id(order.getOrderId().toString());
		indexRequest.source(source);
		return indexRequest;
	}

	public List<IndexRequest> buildOrderDetails(Orders order) {
		String orderId = order.getOrderId().toString();
		return order.getDetails().stream() //
				.filter(item -> null != item) //
				.map(detail -> {
					Map<String, Object> relation = new HashMap<String, Object>();
					relation.put("name", "order_detail");
					relation.put("parent", orderId);

					Map<String, Object> source = new HashMap<String, Object>();
					source.put("order_detail_id", detail.getOrderDetailId());
					source.put("order_id", detail.getOrderId().toString());
					source.put("goods_id", detail.getGoodsId());
					source.put("goods_no", detail.getGoodsNo());
					source.put("goods_name", detail.getGoodsName());
					source.put("goods_img", detail.getGoodsImg());
					source.put("price", detail.getPrice().doubleValue());
					source.put("buy_num", detail.getBuyNum());

					// set releation
					source.put("relation", relation);

					IndexRequest indexRequest = new IndexRequest();
					indexRequest.index("order_index");
					indexRequest.id(detail.getOrderDetailId().toString());
					indexRequest.routing(orderId);
					indexRequest.source(source);
					return indexRequest;
				}).collect(Collectors.toList());
	}
}
```
### (5). 通过DSL检索文档
```
# 需求:查询商户("88888")下,客户地址是:"深圳",并且,客户购买的商品名称包括有"联想"或"大电量"的数据.
# SELECT  o.* 
# FROM t_order o , t_order_detail d
# WHERE o.order_id = d.order_id
# AND o.merchant_id = 88888 
# AND o.customer_address LIKE '%深圳%'
# AND ( d.goods_name LIKE '%联想%'  OR d.goods_name LIKE '%大电量%' )

GET /order_index/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term" : { "merchant_id" : 88888 } } ,
        { "match" : { "customer_address" : "深圳" } }
      ],
      "must": [
        {
          "has_child": {
          "type": "order_detail",
          "query": {
            "bool": {
              "should": [
                { 
                  "match": {
                    "goods_name": "大电量"
                  }
                },
                { 
                  "match": {
                    "goods_name": "联想"
                  }
                }
              ]
            }
          },
            "inner_hits" : {}
        }
      }
      ]
    }
  }
}

# 需求:查询商户("88888")下,客户地址是:"深圳",并且,客户购买的商品名称包括有"联想"或"大电量"的数据.
# 与上面的功能一样,只是换了另一种写法而已(inner_hits只有:has_child支持).
# 1. 对子文档进行检索,返回,父文档信息
# 2. 对父文档进行二次检索.
GET /order_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "has_child": {
            "type": "order_detail",
            "query": {
              "bool": {
                "should": [
                  {
                    "match_phrase": {
                      "goods_name": "联想"
                     }
                  },
                  {
                    "match_phrase": {
                      "goods_name": "大电量"
                     }
                  }
                ]
              }
            }
            ,"inner_hits" : {}
          }
        },
        {
         "term": {
           "merchant_id": {
             "value": "88888"
           }
         }
        },
        {
          "match_phrase": {
            "customer_address": "深圳市"
          }
        }
      ]
    }
  },
  "from": 0,
  "size": 10
}
```

### (6). 通过Java api检索文档
```
package help.lixin.erp;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;

import org.apache.http.HttpHost;
import org.apache.lucene.search.join.ScoreMode;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.InnerHitBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.join.query.HasChildQueryBuilder;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHits;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class EsQueryTest {
	private RestHighLevelClient client;

	@Before
	public void init() {
		RestClientBuilder restClientBuilder = RestClient.builder( // 
				new HttpHost("127.0.0.1", 9200, "http")
		);
		this.client = new RestHighLevelClient(restClientBuilder);
	}

	/**
	 * 需求: <br/>
	 * SELECT o.* <br/>
	 * FROM t_order o , t_order_detail d <br/>
	 * WHERE o.order_id = d.order_id <br/>
	 * AND o.merchant_id = 88888 <br/>
	 * AND o.customer_address LIKE '%深圳市%' <br/>
	 * AND ( d.goods_name LIKE '%联想%' OR d.goods_name LIKE '%大电量%' )
	 */
	@SuppressWarnings("rawtypes")
	@Test
	public void query() {
		// 构建订单详细信息过滤条件.( AND ( d.goods_name LIKE '%联想%' OR d.goods_name LIKE '%大电量%' )
		// )
		BoolQueryBuilder orderDetailBool = QueryBuilders.boolQuery();
		orderDetailBool.should(QueryBuilders.matchQuery("goods_name", "大电量"));
		orderDetailBool.should(QueryBuilders.matchQuery("goods_name", "联想"));
		
		// 构建主从表查询
		HasChildQueryBuilder hasChild = new HasChildQueryBuilder("order_detail", orderDetailBool, ScoreMode.None);
		// fetch 子表(t_order_detail)的信息出来
		hasChild.innerHit(new InnerHitBuilder());

		// bool查询
		BoolQueryBuilder bool = QueryBuilders.boolQuery();
		// AND o.merchant_id = 88888
		bool.filter(QueryBuilders.termQuery("merchant_id", 88888L));
		// AND o.customer_address LIKE '%深圳市%'
		// matchPhraseQuery:短语精准match
		bool.filter(QueryBuilders.matchPhraseQuery("customer_address", "深圳市"));
		// AND ( d.goods_name LIKE '%联想%' OR d.goods_name LIKE '%大电量%' )
		bool.must(hasChild);

		//
		SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
		sourceBuilder.query(bool);
		sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));

		SearchRequest searchRequest = new SearchRequest("order_index");
		searchRequest.source(sourceBuilder);

		try {
			SearchResponse response = client.search(searchRequest, RequestOptions.DEFAULT);
			System.out.println("搜索到:" + response.getHits().getTotalHits().value + " 条数据");
			SearchHits hits = response.getHits();
			for (SearchHit hit : hits) {
				String json = hit.getSourceAsString();

				// 获取所有的子表(t_order_detail)信息
				List<Map> childItems = new ArrayList<>();
				hit.getInnerHits().forEach((k, v) -> {
					SearchHit[] searchHits = v.getHits();
					for (SearchHit searchHit : searchHits) {
						childItems.add(searchHit.getSourceAsMap());
					}
				});
				System.out.println(json);
			}
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```

### (7). DDL语句
```
DROP TABLE t_goods;

DROP TABLE t_orders;
CREATE TABLE t_orders(
    order_id BIGINT NOT NULL   COMMENT '订单id' ,
    order_no VARCHAR(128)    COMMENT '订单编号' ,
    customer_id BIGINT    COMMENT '买家id' ,
    customer_name VARCHAR(1024)    COMMENT '买家名称' ,
    customer_phone VARCHAR(32)    COMMENT '买家电话' ,
    customer_address VARCHAR(32)    COMMENT '买家地址' ,
    merchant_id BIGINT    COMMENT '商户id' ,
    total_money DECIMAL(32,8)    COMMENT '订单总金额' ,
    order_status CHAR(1)    COMMENT '订单状态 1:订单创建
2:订单支付中
3:订单出库中
4:订单完成
5:订单退货' ,
    version INT    COMMENT '乐观锁' ,
    created_by VARCHAR(32)    COMMENT '创建人' ,
    created_time DATETIME    COMMENT '创建时间' ,
    updated_by VARCHAR(32)    COMMENT '更新人' ,
    updated_time DATETIME    COMMENT '更新时间' ,
    PRIMARY KEY (order_id)
) COMMENT = '订单信息';

ALTER TABLE t_orders COMMENT '订单信息';
DROP TABLE t_order_detail;
CREATE TABLE t_order_detail(
    order_detail_id BIGINT NOT NULL   COMMENT '订单信息id' ,
    order_id BIGINT    COMMENT '订单id' ,
    goods_id BIGINT    COMMENT '商品id' ,
    goods_no VARCHAR(128)    COMMENT '商品编号' ,
    goods_name VARCHAR(1024)    COMMENT '商品名称' ,
    goods_img VARCHAR(128)    COMMENT '商品图片' ,
    price DECIMAL(32,8)    COMMENT '价格' ,
    buy_num INT    COMMENT '购买数量' ,
    version INT    COMMENT '乐观锁' ,
    created_by VARCHAR(32)    COMMENT '创建人' ,
    created_time DATETIME    COMMENT '创建时间' ,
    updated_by VARCHAR(32)    COMMENT '更新人' ,
    updated_time DATETIME    COMMENT '更新时间' ,
    PRIMARY KEY (order_detail_id)
) COMMENT = '订单详细信息';

ALTER TABLE t_order_detail COMMENT '订单详细信息';

```

### (8). pom.xml
>  由于SpringBoot依赖了6.7.2版本,所以,需要自己排除,并设置成:7.1.0

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>help.lixin.erp</groupId>
	<artifactId>erp-example</artifactId>
	<version>1.0.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>erp-example</name>
	<url>http://maven.apache.org</url>

	<properties>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<elasticsearch.version>7.1.0</elasticsearch.version>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Greenwich.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>2.1.0.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-netflix</artifactId>
				<version>2.1.1.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>com.baomidou</groupId>
			<artifactId>mybatis-plus-boot-starter</artifactId>
			<version>3.4.2</version>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.48</version>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid</artifactId>
			<version>1.2.4</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>elasticsearch-rest-client</artifactId>
			<version>7.1.0</version>
		</dependency>
		
		
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>elasticsearch-rest-high-level-client</artifactId>
			<version>7.1.0</version>
			<!-- 设置排除 -->
			<exclusions>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch-core</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch-cli</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch-secure-sm</artifactId>
				</exclusion>
				<exclusion>
					<groupId>org.elasticsearch</groupId>
					<artifactId>elasticsearch-x-content</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		
		<!-- 重新添加指定版本 -->
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch-core</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch-cli</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch-secure-sm</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch</groupId>
			<artifactId>elasticsearch-x-content</artifactId>
			<version>${elasticsearch.version}</version>
		</dependency>
		
		<dependency>
			<groupId>com.baomidou</groupId>
			<artifactId>mybatis-plus-generator</artifactId>
			<version>3.4.1</version>
		</dependency>
		<dependency>
			<groupId>org.freemarker</groupId>
			<artifactId>freemarker</artifactId>
			<version>2.3.31</version>
		</dependency>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
</project>
```

### (9). 总结
> ES并没有像MySQL那样,对Join查询支持比较好,
> 而且,官网或者第三方网站也好,对于父子文档只是用于案例,并没有做到真实案例来驱动,比如:既检索子文档,父文档也要参与检索.   
> 至于,如何把MySQL的数据同步到ES中,这个需要Canal了,后续会写相应的blog.  