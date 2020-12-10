---
layout: post
title: 'ElasticSearch Logstash导入CSV文件(六)'
date: 2020-11-29
author: 李新
tags: ElasticSearch
---

### (1). 下载logstash(略)
> https://artifacts.elastic.co/downloads/logstash/logstash-7.10.0-darwin-x86_64.tar.gz   

```
# 工作目录
lixin-macbook:bin lixin$ pwd
/Users/lixin/Developer/elastic-search/logstash-7.10.0/bin
```
### (2). 创建配置文件(logstash.conf)
> /Users/lixin/Developer/elastic-search/work/logstash.conf

```
input {
  file {
    path => "/Users/lixin/Developer/elastic-search/work/movies.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}
filter {
  csv {
    separator => ","
    columns => ["id","content","genre"]
  }

  mutate {
    split => { "genre" => "|" }
    remove_field => ["path", "host","@timestamp","message"]
  }

  mutate {

    split => ["content", "("]
    add_field => { "title" => "%{[content][0]}"}
    add_field => { "year" => "%{[content][1]}"}
  }

  mutate {
    convert => {
      "year" => "integer"
    }
    strip => ["title"]
    remove_field => ["path", "host","@timestamp","message","content"]
  }

}
output {
   elasticsearch {
     hosts => "http://localhost:9200"
     index => "movies"
     document_id => "%{id}"
   }
  stdout {}
}
```

### (3). csv文件

["movies.csv"](/assets/elasticsearch/movies.csv)

### (4). logstash执行导入
```
# logstash工作目录
lixin-macbook:bin lixin$ pwd
/Users/lixin/Developer/elastic-search/logstash-7.10.0/bin

# 导入数据
lixin-macbook:bin lixin$ ./logstash -f /Users/lixin/Developer/elastic-search/work/logstash.conf
```

### (5). kibana检查是否导入成功
!["kibana查看movies是否导入成功"](/assets/elasticsearch/imgs/logstash-import.jpg)
