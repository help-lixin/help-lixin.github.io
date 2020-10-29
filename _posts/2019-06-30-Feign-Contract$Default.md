---
layout: post
title: 'Feign源码(Contract$Default)'
date: 2019-06-30
author: 李新
tags: Feign
---

### (1).Contract$Default
```
public interface Contract {
    // 解析targetType的所有Method,并返回:List<MethodMetadata>
    List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType);
}
```
### (2).Contract$BaseContract
```
abstract class BaseContract implements Contract {
    public List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType) {
      checkState(targetType.getTypeParameters().length == 0, "Parameterized types unsupported: %s",
          targetType.getSimpleName());
      checkState(targetType.getInterfaces().length <= 1, "Only single inheritance supported: %s",
          targetType.getSimpleName());
      if (targetType.getInterfaces().length == 1) {
        checkState(targetType.getInterfaces()[0].getInterfaces().length == 0,
            "Only single-level inheritance supported: %s",
            targetType.getSimpleName());
      }
      
      Map<String, MethodMetadata> result = new LinkedHashMap<String, MethodMetadata>();
      for (Method method : targetType.getMethods()) {
        // 针对静态方法和Object对象,跳过该方法处理
        if (method.getDeclaringClass() == Object.class ||
            (method.getModifiers() & Modifier.STATIC) != 0 ||
            Util.isDefault(method)) {
          continue;
        }
        
        // ***************************************************
        // 交给子类,进行:parseAndValidateMetadata
        // ***************************************************
        MethodMetadata metadata = parseAndValidateMetadata(targetType, method);
        checkState(!result.containsKey(metadata.configKey()), "Overrides unsupported: %s",
            metadata.configKey());
        result.put(metadata.configKey(), metadata);
      } //end for
      return new ArrayList<>(result.values());
    } // end 
    
    // @RequestLine("GET /hello4/{name}/{age}")
    protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
      MethodMetadata data = new MethodMetadata();
      // 设置方法返回的类型
      data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
      data.configKey(Feign.configKey(targetType, method));

      // 如果targetType实现了多个接口
      if (targetType.getInterfaces().length == 1) { // false
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
      }
      
      // ***********************************************************
      // 交给子类:Contract$Default解析targetType类上的注解@Headers
      // ***********************************************************
      processAnnotationOnClass(data, targetType);
      
      // 解析"方法"上的注解
      for (Annotation methodAnnotation : method.getAnnotations()) {
         // ***********************************************************
         // 交给子类:Contract$Default解析targetType上"方法"的所有注解:
         // @RequestLine / @Body / @Headers
         // ***********************************************************
         processAnnotationOnMethod(data, methodAnnotation, method);
      }
      
      
      checkState(data.template().method() != null,
          "Method %s not annotated with HTTP method type (ex. GET, POST)",
          method.getName());

      // @RequestLine("GET /hello4/{name}/{age}")
      // String hello4(@Param("name") String name, @Param("age") Integer age);
      // 获得方法上的所有入参类型 parameterTypes= [ String.class,Integer.class ]
      Class<?>[] parameterTypes = method.getParameterTypes();
      // genericParameterTypes = [String.class,Integer.class]
      // Type包含有泛型
      Type[] genericParameterTypes = method.getGenericParameterTypes();
      
      // 获取方法上所有入参的注解
      Annotation[][] parameterAnnotations = method.getParameterAnnotations();
      // count = 2
      int count = parameterAnnotations.length;
      for (int i = 0; i < count; i++) {
        boolean isHttpAnnotation = false;
        if (parameterAnnotations[i] != null) {
           // ***********************************************************
           // 交给子类:Contract$Default解析targetType上"方法"的所有"入参"注解:
           // 设置:indexToName/formParams/indexToExpanderClass/indexToEncoded/
           // ***********************************************************
          isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
        }
        
        if (parameterTypes[i] == URI.class) { // 设置参数为:URI
          //  设置urlIndex
          data.urlIndex(i);
        } else if (!isHttpAnnotation) { 
           // processAnnotationsOnParameter(...) 方法没有解析到注解的话,
           // 则设置bodyIndex/bodyType
          checkState(data.formParams().isEmpty(),
              "Body parameters cannot be used with form parameters.");
          checkState(data.bodyIndex() == null, "Method has too many Body parameters: %s", method);
          data.bodyIndex(i);
          data.bodyType(Types.resolve(targetType, targetType, genericParameterTypes[i]));
        } //end else if
        
      } //end for

      //  如果有配置注解@HeaderMap,则需要判断是否为HashMap
      if (data.headerMapIndex() != null) {
        checkMapString("HeaderMap", parameterTypes[data.headerMapIndex()],
            genericParameterTypes[data.headerMapIndex()]);
      }
      // 如果有配置注解@QueryMap,则需要判断:标注的类型是否为:Map
      if (data.queryMapIndex() != null) {
        if (Map.class.isAssignableFrom(parameterTypes[data.queryMapIndex()])) {
          checkMapKeys("QueryMap", genericParameterTypes[data.queryMapIndex()]);
        }
      }
      return data;
    } //end parseAndValidateMetadata
}
```
### (3).Contract$Default
```
class Default extends BaseContract {
    static final Pattern REQUEST_LINE_PATTERN = Pattern.compile("^([A-Z]+)[ ]*(.*)$");
    
    // 解析类上的注解@Header
    // @Header( value={ "X-Ping: {token}" } )
    protected void processAnnotationOnClass(MethodMetadata data, Class<?> targetType) {
      // 如果类上标注有:@Header  
      if (targetType.isAnnotationPresent(Headers.class)) {
         // 获得注解上的信息
        String[] headersOnType = targetType.getAnnotation(Headers.class).value();
        checkState(headersOnType.length > 0, "Headers annotation was empty on type %s.",
            targetType.getName());
        Map<String, Collection<String>> headers = toMap(headersOnType);
        // 把MethodMetadata里已经存的headers与刚解析的headers进行合并
        headers.putAll(data.template().headers());
        // 把MethodMetadata的headers进行清空
        data.template().headers(null); // to clear
        // 重新设置成新的:headers
        data.template().headers(headers);
      }
    } //end processAnnotationOnClass
    
    
    // 解析方法上的注解
    // @Headers / @Body / @RequestLine
    protected void processAnnotationOnMethod(MethodMetadata data,
                                             Annotation methodAnnotation,
                                             Method method) {
      Class<? extends Annotation> annotationType = methodAnnotation.annotationType();
      if (annotationType == RequestLine.class) { 
        String requestLine = RequestLine.class.cast(methodAnnotation).value();
        checkState(emptyToNull(requestLine) != null,
            "RequestLine annotation was empty on method %s.", method.getName());

        Matcher requestLineMatcher = REQUEST_LINE_PATTERN.matcher(requestLine);
        if (!requestLineMatcher.find()) {
          throw new IllegalStateException(String.format(
              "RequestLine annotation didn't start with an HTTP verb on method %s",
              method.getName()));
        } else {
          // @RequestLine("GET /hello4/{name}/{age}")
          // 解析请求的方法(GET)
          data.template().method(HttpMethod.valueOf(requestLineMatcher.group(1)));
          // 解析请求的URL(/hello4/{name}/{age})
          data.template().uri(requestLineMatcher.group(2));
        }
        // 是否编码"/"
        data.template().decodeSlash(RequestLine.class.cast(methodAnnotation).decodeSlash());
        data.template()
            .collectionFormat(RequestLine.class.cast(methodAnnotation).collectionFormat());

      } else if (annotationType == Body.class) {
        String body = Body.class.cast(methodAnnotation).value();
        checkState(emptyToNull(body) != null, "Body annotation was empty on method %s.",
            method.getName());
        if (body.indexOf('{') == -1) {
          data.template().body(body);
        } else {
          data.template().bodyTemplate(body);
        }
      } else if (annotationType == Headers.class) {
        String[] headersOnMethod = Headers.class.cast(methodAnnotation).value();
        checkState(headersOnMethod.length > 0, "Headers annotation was empty on method %s.",
            method.getName());
        data.template().headers(toMap(headersOnMethod));
      }
    } //end processAnnotationOnMethod
    
    
    // 解析方法上的入参
    // @Param / @QueryMap / @HeaderMap
    protected boolean processAnnotationsOnParameter(
                          MethodMetadata data,
                          Annotation[] annotations,
                          int paramIndex) {
      boolean isHttpAnnotation = false;
      for (Annotation annotation : annotations) {
        Class<? extends Annotation> annotationType = annotation.annotationType();
        if (annotationType == Param.class) {
          Param paramAnnotation = (Param) annotation;
          // name = name
          String name = paramAnnotation.value();
          checkState(emptyToNull(name) != null, "Param annotation was empty on param %s.",
              paramIndex);
          // 往MethodMetadata的下标[paramIndex]增加name
          nameParam(data, name, paramIndex);
          
          // 对参数的内容是否进行编码
          Class<? extends Param.Expander> expander = paramAnnotation.expander();
          if (expander != Param.ToStringExpander.class) {
            data.indexToExpanderClass().put(paramIndex, expander);
          }
          data.indexToEncoded().put(paramIndex, paramAnnotation.encoded());
          
          isHttpAnnotation = true;
          
          if (!data.template().hasRequestVariable(name)) {
            //向MethodMetadata中添加:formParams变量
            data.formParams().add(name);
          }
        } else if (annotationType == QueryMap.class) {
          checkState(data.queryMapIndex() == null,
              "QueryMap annotation was present on multiple parameters.");
          data.queryMapIndex(paramIndex);
          data.queryMapEncoded(QueryMap.class.cast(annotation).encoded());
          isHttpAnnotation = true;
        } else if (annotationType == HeaderMap.class) {
          checkState(data.headerMapIndex() == null,
              "HeaderMap annotation was present on multiple parameters.");
          data.headerMapIndex(paramIndex);
          isHttpAnnotation = true;
        }
      }
      return isHttpAnnotation;
    }// end  processAnnotationsOnParameter
    
    protected void nameParam(MethodMetadata data, String name, int i) {
      // 判断下标[i]是否存在,如果存在:则返回下标对应的Collection
      // 如果不存在,则创建一个新的Collection,并把下标和Collections增加到MethoMetaData
      Collection<String> names =
          data.indexToName().containsKey(i) ? 
          data.indexToName().get(i) : new ArrayList<String>();
      // 往集合中添加元数
      names.add(name);
      // 往MethodMetadata的指定下标[i],增加Collection
      data.indexToName().put(i, names);
    } //end nameParam
    
    // input= ["X-Ping: {token}"]
    private static Map<String, Collection<String>> toMap(String[] input) {
      Map<String, Collection<String>> result =
          new LinkedHashMap<String, Collection<String>>(input.length);
      for (String header : input) {
        int colon = header.indexOf(':');
        // name = "X-Ping"
        String name = header.substring(0, colon);
        // 如果名称不存在,则给对应的name创建一个Collection
        // 一个Header是可以包含多个value
        if (!result.containsKey(name)) {
          result.put(name, new ArrayList<String>(1));
        }
        // value = 从":"的位置("{token}")
        result.get(name).add(header.substring(colon + 1).trim());
      }
      return result;
    } //end toMap
}
```
### (4).MethodMetadata 模型结构
```
public final class MethodMetadata implements Serializable {
  private String configKey;
  private transient Type returnType;
  private Integer urlIndex;
  private Integer bodyIndex;
  private Integer headerMapIndex;
  private Integer queryMapIndex;
  private boolean queryMapEncoded;
  private transient Type bodyType;
 // @RequestLine/@Headers/@Body 解析后的内容存储
 // method/url/body/headers...
  private RequestTemplate template = new RequestTemplate();
  private List<String> formParams = new ArrayList<String>();
  private Map<Integer, Collection<String>> indexToName =
      new LinkedHashMap<Integer, Collection<String>>();
  private Map<Integer, Class<? extends Expander>> indexToExpanderClass =
      new LinkedHashMap<Integer, Class<? extends Expander>>();
  private Map<Integer, Boolean> indexToEncoded = new LinkedHashMap<Integer, Boolean>();
  private transient Map<Integer, Expander> indexToExpander;
}
```
### (5).总结
> Contract主要是针对注解进行解析,并转换成模型:MethodMetadata