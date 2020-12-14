---
layout: post
title: 'ElasticSearch 源码Settings(三)'
date: 2020-12-13
author: 李新
tags: ElasticSearch源码
---

### (1). Settings类图
> 从Settings类的结构图分析,该类应该是:存取配置文件的载体.

!["Settings类图"](/assets/elasticsearch/imgs/Settings-class.jpg)

### (2). Settings测试
```
// *************************************************
// 3. builder(构建者模式)
// *************************************************
Settings settings = 
    Settings.builder()
            .put("foobar.0", "bar")
            .put("foobar.1", "test")
            .put("bar", "hello world")
            // *********************************************
            // 4. Settings$Builder.build
            // *********************************************
            .build();
```
### (3). Settings.builder
> 构建出Settings$Builder对象. 

```
public final class Settings implements ToXContentFragment {

    public static class Builder {

        public static final Settings EMPTY_SETTINGS = new Builder().build();

        private final Map<String, Object> map = new TreeMap<>();
        
        private final SetOnce<SecureSettings> secureSettings = new SetOnce<>();

        // 构造器私有化
        private Builder() {
        } //end 构造器

        // 添加key:value
        public Builder put(String key, String value) {
            map.put(key, value);
            return this;
        }

    } //end  Builder

    // ***********************************************
    // 3.1 builder()
    // ***********************************************
    public static Builder builder() {
        // 创建了一个内部的:Builder出来
        return new Builder();
    } // end builder

} // end Settings
```
### (4). Settings$Builder.build
```
public Settings build() {
    // 调用:processLegacyLists方法之前
    // map = {
    //    "bar" : "hello world" ,
    //    "foobar.0" : "bar" ,
    //    "foobar.1" : "test"
    // }
    // 处理map中的List
    processLegacyLists(map);
    // 调用:processLegacyLists方法之后
    // map = {
    //    "bar" : "hello world",
    //    "foobar" : ["bar","test"]
    // }

    return new Settings(map, secureSettings.get());
} // end build

// 处理Map中key带有下标的内容,将这些内容转换成:List
private void processLegacyLists(Map<String, Object> map) {
    // map = {
    //    "foobar.0":"bar",
    //    "foobar.1":"test",
    //    "bar":"hello world"
    //    }
    // map.size = 3
    // array = map.keys

    String[] array = map.keySet().toArray(new String[map.size()]);

    for (String key : array) {
        // 判断key是否由:".0"结尾.代表是数组
        if (key.endsWith(".0")) {  // foobar.0 | foobar.1
            int counter = 0;
            // 截取最后一个点的前缀
            String prefix = key.substring(0, key.lastIndexOf('.'));

            // 在Map中判断前缀是否存在,若存以,则抛出异常,因为这个前缀对应的数据结果应该要是:List
            if (map.containsKey(prefix)) {
                throw new IllegalStateException("settings builder can't contain values for [" + prefix + "=" + map.get(prefix)
                    + "] and [" + key + "=" + map.get(key) + "]");
            }


            List<String> values = new ArrayList<>();
            // *************************************************
            // 注意:whie循环
            // *************************************************
            while (true) {
                // ***********************************************
                // 通过前缀+自增,从Map中获取对应的Value,直到获取不到Value为止.
                // ***********************************************
                // listKey="foobar.0" | "foobar.1" | "foobar.2"
                String listKey = prefix + '.' + (counter++);
                // 根据key从map中获得对应的value
                String value = get(listKey);

                if (value == null) { //"foobar.2"在map中不存在
                    // map中对应的下标前缀已经被移除.
                    // 往map中添加前缀("foobar"=["bar","test"]);
                    map.put(prefix, values);
                    break;
                } else {
                    // 添加foobar.0对应的内容到临时集合中
                    values.add(value);
                    // 从map中移除这个key("foobar.0"|"foobar.1")
                    map.remove(listKey);
                }
            }// end while
        }
    }
} // end processLegacyLists
```
### (5). 总结
> 从类结构和一个测试案例来分析,Settings是配置项的载体.它类似于:Properties,只是比Properties增加了更多的功能. 
