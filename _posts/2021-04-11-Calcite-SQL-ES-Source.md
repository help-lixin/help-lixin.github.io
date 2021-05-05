---
layout: post
title: 'Calcite ES源码深入(五)'
date: 2021-04-11
author: 李新
tags:  Calcite
---

### (1). 概述
> 在这一小节,部析Calcite是如何通过SQL检索ES的过程.

### (2). ElasticsearchSchema
```
package org.apache.calcite.adapter.elasticsearch;

public class ElasticsearchSchema extends AbstractSchema {

  private final RestClient client;

  private final ObjectMapper mapper;

  private final Map<String, Table> tableMap;
  
  // Query最大返回数量
  private final int fetchSize;
  
  // 1. 构造器
  public ElasticsearchSchema(RestClient client, ObjectMapper mapper, String index) {
      this(client, mapper, index, ElasticsearchTransport.DEFAULT_FETCH_SIZE);
  } // end 
  
  @VisibleForTesting
  ElasticsearchSchema(
                RestClient client, 
				ObjectMapper mapper,
				String index, 
				int fetchSize) {
      super();
      this.client = Objects.requireNonNull(client, "client");
      this.mapper = Objects.requireNonNull(mapper, "mapper");
      Preconditions.checkArgument(fetchSize > 0, "invalid fetch size. Expected %s > 0", fetchSize);
      this.fetchSize = fetchSize;
  
      if (index == null) {
        try {
           // *************************************************************
		   // 2. 加载ES中所有的索引库
		   // *************************************************************
          this.tableMap = createTables(indicesFromElastic());
        } catch (IOException e) {
          throw new UncheckedIOException("Couldn't get indices", e);
        }
      } else {
         // 加载具体某一个索引库
        this.tableMap = createTables(Collections.singleton(index));
      }
  } // end  

  
  // *********************************************************************
  // 3.向ES服务器,发送GET请求,获得所有的索引库名称.
  // *********************************************************************
  private Set<String> indicesFromElastic() throws IOException {
      final String endpoint = "/_alias";
      final Response response = client.performRequest(new Request("GET", endpoint));
      try (InputStream is = response.getEntity().getContent()) {
        final JsonNode root = mapper.readTree(is);
        if (!(root.isObject() && root.size() > 0)) {
          final String message = String.format(Locale.ROOT, "Invalid response for %s/%s "
              + "Expected object of at least size 1 got %s (of size %d)", response.getHost(),
              response.getRequestLine(), root.getNodeType(), root.size());
          throw new IllegalStateException(message);
        }
		// 获得所有索引库的名称
        Set<String> indices = Sets.newHashSet(root.fieldNames());
        return indices;
      }
  }// end indicesFromElastic

  
  // *******************************************************************
  // 4. 创建所有的表
  // *******************************************************************
  private Map<String, Table> createTables(Iterable<String> indices) {
	final ImmutableMap.Builder<String, Table> builder = ImmutableMap.builder();
	for (String index : indices) {
		// 4.1 为每一个索引库创建一个:ElasticsearchTransport,专门负责与ES通信的
		final ElasticsearchTransport transport = new ElasticsearchTransport(client, mapper, index, fetchSize);
		// 4.2 创建ES Table,并把请求委托给:ElasticsearchTransport.
		// 4.3 ElasticsearchTransport主要是实现了对:_search的封装.
		builder.put(index, new ElasticsearchTable(transport));
	}
	return builder.build();
  } // end 
  
}

```
### (3). ElasticsearchTransport
```
final class ElasticsearchTransport {
	static final int DEFAULT_FETCH_SIZE = 5196;
	
	  private final ObjectMapper mapper;
	  // es提供的client
	  private final RestClient restClient;
	
	  final String indexName;
	  // 调用ES接口(http://localhost:9200/),获得es版本信息
	  final ElasticsearchVersion version;
	  // 调用ES接口(http://localhost:9200/${index}/_mapping),获得索引库的mapping信息.
	  final ElasticsearchMapping mapping;
	  // fetch大小
	  final int fetchSize;
	
	// 1. 构造器
	ElasticsearchTransport(final RestClient restClient,
	                         final ObjectMapper mapper,
	                         final String indexName,
	                         final int fetchSize) {
		this.mapper = Objects.requireNonNull(mapper, "mapper");
		this.restClient = Objects.requireNonNull(restClient, "restClient");
		this.indexName = Objects.requireNonNull(indexName, "indexName");
		this.fetchSize = fetchSize;
		// 调用ES接口(http://localhost:9200/),获得es版本信息
		this.version = version(); // cache version
		// 调用ES接口(http://localhost:9200/${index}/_mapping),获得索引库的mapping信息.
		this.mapping = fetchAndCreateMapping(); // cache mapping
	} // end
	
	
	// 2. 检索 
	Function<ObjectNode, ElasticsearchJson.Result> search() {
	    return search(Collections.emptyMap());
	} // end search
	
	// 3. 带条件的检索
	Function<ObjectNode, ElasticsearchJson.Result> search(final Map<String, String> httpParams) {
	    Objects.requireNonNull(httpParams, "httpParams");
		
	    return query -> {
	      // 钩子回调(你可以实现:java.util.function.Consumer,并调用Hook.addThread)
	      Hook.QUERY_PLAN.run(query);
		  
		  // 请求ES的URL
		  // indexName = "books"
	      String path = String.format(Locale.ROOT, "/%s/_search", indexName);
	      final HttpPost post;
	      try {
	        URIBuilder builder = new URIBuilder(path);
	        httpParams.forEach(builder::addParameter);
	        post = new HttpPost(builder.build());
			
			// 把所有的请求参数,转换成:json字符串
			// SELECT _MAP['id'],_MAP['title'],_MAP['price'] FROM es.books WHERE _MAP['price'] > 60 LIMIT 2
			// {"query":{"constant_score":{"filter":{"range":{"price":{"gt":60}}}}},"_source":["id","title","price"],"size":2}
			
			// SELECT * FROM es.books WHERE _MAP['price'] > 10 offset 0 fetch next 10 rows only
			// {"query":{"constant_score":{"filter":{"range":{"price":{"gt":10}}}}},"from":0,"size":10}
			
			// 统计SQL
			// SELECT count(*) FROM es.books WHERE _MAP['price'] > 50
			// {"query":{"constant_score":{"filter":{"range":{"price":{"gt":50}}}}},"_source":false,"size":0,"stored_fields":"_none_","track_total_hits":true}
	        final String json = mapper.writeValueAsString(query);
			
	        LOGGER.debug("Elasticsearch Query: {}", json);
	        post.setEntity(new StringEntity(json, ContentType.APPLICATION_JSON));
	      } catch (URISyntaxException e) {
	        throw new RuntimeException(e);
	      } catch (JsonProcessingException e) {
	        throw new UncheckedIOException(e);
	      }
		  // 发送HTTP请求
	      return rawHttp(ElasticsearchJson.Result.class).apply(post);
	    };
	} // end search
}
```
### (4). 总结
> Calcite通过对SQL的解析,转换成HTTP请求,但是,对ES Client的设计,感觉不咋的.  
> ElasticsearchTable类在这里不进行深入,后面还要讲:TranslatableTable/ScannableTable/FilterableTable的区别,到时回来再深入.  