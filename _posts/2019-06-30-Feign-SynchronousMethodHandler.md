---
layout: post
title: 'Feign源码(SynchronousMethodHandler)'
date: 2019-06-30
author: 李新
tags: Feign
---

### (1).SynchronousMethodHandler
```
final class SynchronousMethodHandler implements MethodHandler {
    public Object invoke(Object[] argv) throws Throwable {
        // 1. 委托给:ReflectiveFeign$BuildTemplateByResolvingArgs 创建:RequestTemplate
        RequestTemplate template = buildTemplateFromArgs.create(argv);
        // 重试对象
        Retryer retryer = this.retryer.clone();
        while (true) {
          try {
             // 5. 执行并且解码 
            return executeAndDecode(template);
          } catch (RetryableException e) {
            try {
              retryer.continueOrPropagate(e);
            } catch (RetryableException th) {
              Throwable cause = th.getCause();
              if (propagationPolicy == UNWRAP && cause != null) {
                throw cause;
              } else {
                throw th;
              }
            }
            if (logLevel != Logger.Level.NONE) {
              logger.logRetry(metadata.configKey(), logLevel);
            }
            continue;
          }
        }
  } //end invoke
  
  // 5. 执行并且解码
  Object executeAndDecode(RequestTemplate template) throws Throwable {
    // 6. 委托给:HardCodedTarget 创建Request
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      // ****************************************************************
      // 7.通过HttpURLConnection执行请求
      // 这一部份内容参考另一部份的源码解析
      // ****************************************************************
      response = client.execute(request, options);
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
    boolean shouldClose = true;
    try {
      if (logLevel != Logger.Level.NONE) {
        response = logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
      }
      // 如果返回类型是:Response,直接返回:Response
      if (Response.class == metadata.returnType()) {
            if (response.body() == null) {
              return response;
            }
            if (response.body().length() == null ||
                response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
              shouldClose = false;
              return response;
            }
            // Ensure the response body is disconnected
            byte[] bodyData = Util.toByteArray(response.body().asInputStream());
            return response.toBuilder().body(bodyData).build();
      } // end if
      
      // 如果编码是200~300之间
      if (response.status() >= 200 && response.status() < 300) {
         // 返回类型是:void
        if (void.class == metadata.returnType()) {
          return null;
        } else {
          // **********************************************************
          // 调用解码器进行解码  
          // **********************************************************
          Object result = decode(response);
          shouldClose = closeAfterDecode;
          return result;
        }
      } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
        Object result = decode(response);
        shouldClose = closeAfterDecode;
        return result;
      } else {
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
      }
      throw errorReading(request, response, e);
    } finally {
      if (shouldClose) {
        ensureClosed(response.body());
      }
    }
  } //end executeAndDecode
  
  
  Request targetRequest(RequestTemplate template) {
    for (RequestInterceptor interceptor : requestInterceptors) {
      interceptor.apply(template);
    }
    // 委托给:HardCodedTarget创建:Request
    // 从请求模板(RequestTemplate)创建出一个Request
    return target.apply(template);
  }// end targetRequest
  
}
```
### (2).ReflectiveFeign$BuildTemplateByResolvingArgs
```
private static class BuildTemplateByResolvingArgs 
               implements RequestTemplate.Factory {
                              
    public RequestTemplate create(Object[] argv) {
      // 2. 基于metadata.template拷贝出一份新的 RequestTemplate
      RequestTemplate mutable = RequestTemplate.from(metadata.template());
      
      // 只有当初数为:URI时,这个代码块才会进入
      if (metadata.urlIndex() != null) { // false
        int urlIndex = metadata.urlIndex();
        checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
        mutable.target(String.valueOf(argv[urlIndex]));
      }
      
      // 获取参数变量
      Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
      for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
        // 0/1/2...  
        int i = entry.getKey();
        // 请求方法时传递的参数( hello("zhangsan",25) )
        Object value = argv[entry.getKey()];
        if (value != null) { // Null values are skipped.
          
          if (indexToExpander.containsKey(i)) { 
            value = expandElements(indexToExpander.get(i), value);
          }
          // @Param(value="name")
          // entry.getValue = name
          for (String name : entry.getValue()) {
            // name / zhangsan
            varBuilder.put(name, value);
          } //end if
        } // end if
      } //end for

      // 对参数进行解析
      RequestTemplate template = resolve(argv, mutable, varBuilder);
      if (metadata.queryMapIndex() != null) { // false
        Object value = argv[metadata.queryMapIndex()];
        Map<String, Object> queryMap = toQueryMap(value);
        template = addQueryMapQueryParameters(queryMap, template);
      }

      if (metadata.headerMapIndex() != null) { // flase
        template =
            addHeaderMapHeaders((Map<String, Object>) argv[metadata.headerMapIndex()], template);
      }
      return template;
    } //end create
    
    protected RequestTemplate resolve(Object[] argv,
                                      RequestTemplate mutable,
                                      Map<String, Object> variables) {
      // 重新委托给:RequestTemplate进行变量解析                                    
      return mutable.resolve(variables);
    } //end resolve
    
}
```
### (3).RequestTemplate
```
public final class RequestTemplate implements Serializable {
    // 3. 解析URL上的变量并进行替换
    public RequestTemplate resolve(Map<String, ?> variables) {
        StringBuilder uri = new StringBuilder();
    
        // 拷贝一份新的RequestTemplate
        RequestTemplate resolved = RequestTemplate.from(this);
        
        if (this.uriTemplate == null) { // false
          this.uriTemplate = UriTemplate.create("", !this.decodeSlash, this.charset);
        }
        
        // 委托给UriTemplate进行变量替换
        // /hello4/%E6%9D%8E%E6%96%B0/28
        uri.append(this.uriTemplate.expand(variables));
    
    
        if (!this.queries.isEmpty()) { // false
          resolved.queries(Collections.emptyMap());
          StringBuilder query = new StringBuilder();
          Iterator<QueryTemplate> queryTemplates = this.queries.values().iterator();
    
          while (queryTemplates.hasNext()) {
            QueryTemplate queryTemplate = queryTemplates.next();
            String queryExpanded = queryTemplate.expand(variables);
            if (Util.isNotBlank(queryExpanded)) {
              query.append(queryTemplate.expand(variables));
              if (queryTemplates.hasNext()) {
                query.append("&");
              }
            }
          }
    
          String queryString = query.toString();
          if (!queryString.isEmpty()) {
            Matcher queryMatcher = QUERY_STRING_PATTERN.matcher(uri);
            if (queryMatcher.find()) {
              /* the uri already has a query, so any additional queries should be appended */
              uri.append("&");
            } else {
              uri.append("?");
            }
            uri.append(queryString);
          }
        }
    
        /* add the uri to result */
        resolved.uri(uri.toString());
    
        /* headers */
        if (!this.headers.isEmpty()) { // false
          resolved.headers(Collections.emptyMap());
          for (HeaderTemplate headerTemplate : this.headers.values()) {
            /* resolve the header */
            String header = headerTemplate.expand(variables);
            if (!header.isEmpty()) {
              /* split off the header values and add it to the resolved template */
              String headerValues = header.substring(header.indexOf(" ") + 1);
              if (!headerValues.isEmpty()) {
                resolved.header(headerTemplate.getName(), headerValues);
              }
            }
          }
        }
    
        resolved.body(this.body.expand(variables));
    
        /* mark the new template resolved */
        resolved.resolved = true;
        return resolved;
  } //end resolve
    
}
```
### (4).UriTemplate
```
// 4. 委托给UriTemplate进行expand处理
public String expand(Map<String, ?> variables) {
    
    if (variables == null) {
      throw new IllegalArgumentException("variable map is required.");
    }

    
    // ************************************************
    // templateChunks 保存着URL信息.
    // 有表达式的URL(/{name}) == Expression
    // 普通文本URL(/hello4/) == Literal
    // ************************************************
    StringBuilder resolved = new StringBuilder();
    for (TemplateChunk chunk : this.templateChunks) {
      if (chunk instanceof Expression) {  // 是否为表达式
        Expression expression = (Expression) chunk;
        Object value = variables.get(expression.getName());
        if (value != null) {
          String expanded = expression.expand(value, this.encode);
          if (!this.encodeSlash) {
            logger.fine("Explicit slash decoding specified, decoding all slashes in uri");
            expanded = expanded.replaceAll("\\%2F", "/");
          }
          resolved.append(expanded);
        } else {
          if (this.allowUnresolved) {
            /* unresolved variables are treated as literals */
            resolved.append(encode(expression.toString()));
          }
        }
      } else {
        /* chunk is a literal value */
        resolved.append(chunk.getValue());
      }
    }
    return resolved.toString();
}
```