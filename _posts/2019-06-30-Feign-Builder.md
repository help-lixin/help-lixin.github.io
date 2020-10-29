---
layout: post
title: 'Feign源码(Builder)'
date: 2019-06-30
author: 李新
tags: Feign
---

### (1).StringDecoder 
```
package help.lixin.samples.feign;

import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Type;

import org.apache.commons.io.IOUtils;

import com.fasterxml.jackson.databind.RuntimeJsonMappingException;

import feign.Response;
import feign.codec.Decoder;

public class StringDecoder implements Decoder {
   // 实现解码器decode方法
	@Override
	public Object decode(Response response, Type type) throws IOException {
		if (response.body() == null) {
			return null;
		}
		InputStream inputStream = response.body().asInputStream();
		try {
			byte[] buffer = new byte[inputStream.available()];
			IOUtils.readFully(inputStream, buffer);
			return new String(buffer);
		} catch (RuntimeJsonMappingException e) {
			if (e.getCause() != null && e.getCause() instanceof IOException) {
				throw IOException.class.cast(e.getCause());
			}
			throw e;
		}
	}
}
```
### (2).FeignTest
```
package help.lixin.samples.feign;

import feign.Feign;

public class FeignTest {
	public static void main(String[] args) {
        // 创建解码器
        Decoder decoder = new StringDecoder();
        // 通过builder.创建包裹target
        HelloService helloService = Feign
            .builder() //
            .decoder(decoder) //
            .target(HelloService.class, "http://127.0.0.1:7070");
        String result = helloService.hello4("李新", 28);
        System.out.println(result);
	}
}
```
### (3).Feign
```
// 1.Feign是一个抽象类,所以不能new.
public abstract class Feign {
    // 2.通过Build方法,构建.
    public static Builder builder() {
      return new Builder();
    }
    
    public static class Builder {

    private final List<RequestInterceptor> requestInterceptors =
        new ArrayList<RequestInterceptor>();
    private Logger.Level logLevel = Logger.Level.NONE;
    private Contract contract = new Contract.Default();
    private Client client = new Client.Default(null, null);
    private Retryer retryer = new Retryer.Default();
    private Logger logger = new NoOpLogger();
    private Encoder encoder = new Encoder.Default();
    private Decoder decoder = new Decoder.Default();
    private QueryMapEncoder queryMapEncoder = new QueryMapEncoder.Default();
    private ErrorDecoder errorDecoder = new ErrorDecoder.Default();
    private Options options = new Options();
    private InvocationHandlerFactory invocationHandlerFactory =
        new InvocationHandlerFactory.Default();
    private boolean decode404;
    private boolean closeAfterDecode = true;
    private ExceptionPropagationPolicy propagationPolicy = NONE;

    // 配置日志级别
    public Builder logLevel(Logger.Level logLevel) {
      this.logLevel = logLevel;
      return this;
    }
    
    // 配置解析器(对target进行解析,并转换成:MethodMetadata)
    public Builder contract(Contract contract) {
      this.contract = contract;
      return this;
    }
    
    // 提交http请求的Client(默认为:HttpURLConnection)
    public Builder client(Client client) {
      this.client = client;
      return this;
    }

    // 重试
    public Builder retryer(Retryer retryer) {
      this.retryer = retryer;
      return this;
    }

    public Builder logger(Logger logger) {
      this.logger = logger;
      return this;
    }

    // 编码器
    public Builder encoder(Encoder encoder) {
      this.encoder = encoder;
      return this;
    }
    // 解码器
    public Builder decoder(Decoder decoder) {
      this.decoder = decoder;
      return this;
    }
    // 
    public Builder queryMapEncoder(QueryMapEncoder queryMapEncoder) {
      this.queryMapEncoder = queryMapEncoder;
      return this;
    }

    public Builder mapAndDecode(ResponseMapper mapper, Decoder decoder) {
      this.decoder = new ResponseMappingDecoder(mapper, decoder);
      return this;
    }

    // 解码404
    public Builder decode404() {
      this.decode404 = true;
      return this;
    }

    // 配置发生错误时的解码器
    public Builder errorDecoder(ErrorDecoder errorDecoder) {
      this.errorDecoder = errorDecoder;
      return this;
    }

    // 其它选项
    public Builder options(Options options) {
      this.options = options;
      return this;
    }

    // 配置拦截器
    public Builder requestInterceptor(RequestInterceptor requestInterceptor) {
      this.requestInterceptors.add(requestInterceptor);
      return this;
    }

   // 配置多个:RequestInterceptor
    public Builder requestInterceptors(Iterable<RequestInterceptor> requestInterceptors) {
      this.requestInterceptors.clear();
      for (RequestInterceptor requestInterceptor : requestInterceptors) {
        this.requestInterceptors.add(requestInterceptor);
      }
      return this;
    }

    // ****************************************************
    // 配置反射:InvocationHandler的工厂
    // ****************************************************
    public Builder invocationHandlerFactory(InvocationHandlerFactory invocationHandlerFactory) {
      this.invocationHandlerFactory = invocationHandlerFactory;
      return this;
    }

    public Builder doNotCloseAfterDecode() {
      this.closeAfterDecode = false;
      return this;
    }

    public Builder exceptionPropagationPolicy(ExceptionPropagationPolicy propagationPolicy) {
      this.propagationPolicy = propagationPolicy;
      return this;
    }

    // 构建Feign的子类
    public <T> T target(Class<T> apiType, String url) {
      // 构建:Target
      return target(
                   new HardCodedTarget<T>(apiType, url)
                   );
    }

    // **********************************************
    // 构建Feign子类,并调用:newInstance(target)
    // 会根据target生成对应的子类.
    // **********************************************
    public <T> T target(Target<T> target) {
      // Feign feign = build();
      // feign.newInstance(target);  
      return build().newInstance(target);
    }

    public Feign build() {
      // 创建:MethodHandler的工厂
      SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
                  new SynchronousMethodHandler.Factory(
                         // client HttpConnection
                         client, 
                         // 重试处理
                         retryer, 
                         // 请求拦截器
                         requestInterceptors, 
                         // 日志
                         logger,
                         // 日志级别
                         logLevel, 
                         // 解码404
                         decode404, 
                         closeAfterDecode, 
                         propagationPolicy 
                    ); //end 
              
      // 解析Handler到名称处理
      ParseHandlersByName handlersByName =
              new ParseHandlersByName(
                     // 真正的解析器
                     contract, 
                     // 其它参数(读写超时等)
                     options, 
                     // 编码器
                     encoder,
                     // 解码器 
                     decoder,
                     // 对参数进行编码成map 
                     queryMapEncoder,
                     // 错误的解码器
                     errorDecoder, 
                     // MethodHandlerFactory工厂
                     synchronousMethodHandlerFactory
              ); //end 
      // 创建了:ReflectiveFeign
      return 
             new ReflectiveFeign(
                     handlersByName, 
                     // InvocationHandler工厂
                     invocationHandlerFactory, 
                     queryMapEncoder
                 );
    } //end build
  } //endBuilder
} //end Feign
```
### (4).总结
> Feign是一个抽象接口,不能直接实例化.可以通过内部静态方法Builder进行构建.
> 为什么不直接使用Feign的子类(ReflectiveFeign),因为:ReflectiveFeign需要依赖的对象还是比较多,所以通过Builder进行构建出来(ReflectiveFeign).在target方法之后触发了ReflectiveFeign.newInstance()构建Proxy对象.
