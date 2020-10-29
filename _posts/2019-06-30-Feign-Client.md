---
layout: post
title: 'Feign源码(Client)'
date: 2019-06-30
author: 李新
tags: Feign
---

### (1).Client
```
public interface Client {
    Response execute(Request request, Options options) throws IOException;
}
```
### (2).Client$Default
```
public static class Default implements Client {
    
    @Override
    public Response execute(Request request, Options options) throws IOException {
      HttpURLConnection connection = convertAndSend(request, options);
      return convertResponse(connection, request);
    }
    
    // 转换request并发送请求
    HttpURLConnection convertAndSend(Request request, Options options) throws IOException {
      final HttpURLConnection connection =
          (HttpURLConnection) new URL(request.url()).openConnection();
          
      if (connection instanceof HttpsURLConnection) { // false
        HttpsURLConnection sslCon = (HttpsURLConnection) connection;
        if (sslContextFactory != null) {
          sslCon.setSSLSocketFactory(sslContextFactory);
        }
        if (hostnameVerifier != null) {
          sslCon.setHostnameVerifier(hostnameVerifier);
        }
      }
      // 设置连接相关信息
      connection.setConnectTimeout(options.connectTimeoutMillis());
      connection.setReadTimeout(options.readTimeoutMillis());
      connection.setAllowUserInteraction(false);
      connection.setInstanceFollowRedirects(options.isFollowRedirects());
      connection.setRequestMethod(request.httpMethod().name());
      
      // contentEncodingValues = null
      Collection<String> contentEncodingValues = request.headers().get(CONTENT_ENCODING);
      boolean gzipEncodedRequest =
          contentEncodingValues != null && contentEncodingValues.contains(ENCODING_GZIP);
      boolean deflateEncodedRequest =
          contentEncodingValues != null && contentEncodingValues.contains(ENCODING_DEFLATE);

      boolean hasAcceptHeader = false;
      Integer contentLength = null;
      for (String field : request.headers().keySet()) {
        if (field.equalsIgnoreCase("Accept")) {
          hasAcceptHeader = true;
        }
        
        for (String value : request.headers().get(field)) {
          if (field.equals(CONTENT_LENGTH)) {
            if (!gzipEncodedRequest && !deflateEncodedRequest) {
              contentLength = Integer.valueOf(value);
              connection.addRequestProperty(field, value);
            }
          } else {
            connection.addRequestProperty(field, value);
          }
        }// end for
        
      } //end for
      
      // Some servers choke on the default accept string.
      if (!hasAcceptHeader) { // true
        connection.addRequestProperty("Accept", "*/*");
      }
        
      if (request.body() != null) { // 有请求内容的情况下
        if (contentLength != null) {
          connection.setFixedLengthStreamingMode(contentLength);
        } else {
          connection.setChunkedStreamingMode(8196);
        }
        // 设置连接输出
        connection.setDoOutput(true);
        // 从connection中获得输出流信息
        OutputStream out = connection.getOutputStream();
        // 是否开启压缩
        if (gzipEncodedRequest) {
          out = new GZIPOutputStream(out);
        } else if (deflateEncodedRequest) {
          out = new DeflaterOutputStream(out);
        }
        try {
          // 读取request.body()流中的写到:connection.getOutputStream()里
          out.write(request.body());
        } finally {
          try {
            out.close();
          } catch (IOException suppressed) { // NOPMD
          }
        }
        
      }
      return connection;
    }// end convertAndSend

   // 把connection中的内容转化成:Response
  Response convertResponse(HttpURLConnection connection, Request request) throws IOException {
      // 获取请求返回后的状态码
      int status = connection.getResponseCode();
      String reason = connection.getResponseMessage();

      if (status < 0) {
        throw new IOException(format("Invalid status(%s) executing %s %s", status,
            connection.getRequestMethod(), connection.getURL()));
      }
      
      // 获得所有的请求头信息
      Map<String, Collection<String>> headers = new LinkedHashMap<String, Collection<String>>();
      for (Map.Entry<String, List<String>> field : connection.getHeaderFields().entrySet()) {
        // response message
        if (field.getKey() != null) {
          headers.put(field.getKey(), field.getValue());
        }
      }
      // 获取返回体的长度
      Integer length = connection.getContentLength();
      if (length == -1) {
        length = null;
      }
      
      InputStream stream;
      if (status >= 400) {
        stream = connection.getErrorStream();
      } else {
        stream = connection.getInputStream();
      }
      // 构建Response
      return Response.builder()
          .status(status)
          .reason(reason)
          .headers(headers)
          .request(request)
          .body(stream, length)
          .build();
    } //end convertResponse            
    
}
```
### (3).Request
```
public final class Request {
    private final HttpMethod httpMethod;
    private final String url;
    private final Map<String, Collection<String>> headers;
    private final Body body;
}
```