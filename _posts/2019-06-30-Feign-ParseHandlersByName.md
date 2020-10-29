---
layout: post
title: 'Feign源码(ParseHandlersByName)'
date: 2019-06-30
author: 李新
tags: Feign
---

### (1).ParseHandlersByName
```
static final class ParseHandlersByName {
    public Map<String, MethodHandler> apply(Target key) {
        // *******************************************
        // 委托给:Contract$Default进行解析并运回元数据
        // 具体内容请查看:Contract$Default源码解析
        // *******************************************
        List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
        
        Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
        for (MethodMetadata md : metadata) {
            // 构建出一个请求模板
            BuildTemplateByResolvingArgs buildTemplate;
            // @Param会调用:MethodMetadata.formParams().add(...)
            // @Body才会让template().bodyTemplate() != null
            if ( !md.formParams().isEmpty() && 
                 md.template().bodyTemplate() == null) { // false
                 
              buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
            } else if (md.bodyIndex() != null) {
              // 当对参数一个解析都没有完成的情况下,才会设置:bodyIndex/bodyType
              buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
            } else { // true
              // ****************************************
              // 构建模板解析器(后面进行详细讲解)
              // ****************************************
              buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder);
            }
            
            // *****************************************************
            // value[MethodHandler] = SynchronousMethodHandler.Factory.create(...)
            // key = className#methodName(paramType...)
            // *****************************************************
            result.put(
                // key = className#methodName(paramType...)
                md.configKey(),
                // SynchronousMethodHandler.Factory.create
                // value = SynchronousMethodHandler
                factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
        }// end for
        return result;
    }// end apply  
}// end ParseHandlersByName
```
### (2).总结
> ParseHandlersByName的apply方法执行过程:
> 1. 委托给Contract$Default的parseAndValidatateMetadata进行解析.返回:List<MethodMetadata>
> 2. 对List<MethodMetadata>进行遍历,变成换:Map<String, MethodHandler>