---
layout: post
title: 'redis-benchmark 基准测试'
date: 2021-07-01
author: 李新
tags:  Redis
---

### (1). 前言
> 对Redis进行基准测试,需要测出最终的QPS.

### (2). redis-benchmark命令格式
```
lixin-macbook:src lixin$ ./redis-benchmark -h
Usage: redis-benchmark [-h <host>] [-p <port>] [-c <clients>] [-n <requests>] [-k <boolean>]

 -h <hostname>      Server hostname (default 127.0.0.1)
 -p <port>          Server port (default 6379)
 -s <socket>        Server socket (overrides host and port)               # 指定socket的情况下,host和port将会忽略
 -a <password>      Password for Redis Auth                               # 指定Redis密码
 --user <username>  Used to send ACL style 'AUTH username pass'. Needs -a.
 -c <clients>       Number of parallel connections (default 50)           # 配置client的连接池数量(并发数)
 -n <requests>      Total number of requests (default 100000)             # 总共要发起多少请求
 -d <size>          Data size of SET/GET value in bytes (default 3)       # 以字节的形式指定 SET/GET 值的数据大小
 --dbnum <db>       SELECT the specified db number (default 0)            # 选择哪个DB
 --threads <num>    Enable multi-thread mode.
 --cluster          Enable cluster mode.
 --enable-tracking  Send CLIENT TRACKING on before starting benchmark.
 -k <boolean>       1=keep alive 0=reconnect (default 1)
 -r <keyspacelen>   Use random keys for SET/GET/INCR, random values for SADD,        # 设置随机KEY的大小
                    random members and scores for ZADD.
  Using this option the benchmark will expand the string __rand_int__
  inside an argument with a 12 digits number in the specified range
  from 0 to keyspacelen-1. The substitution changes every time a command
  is executed. Default tests use this to hit random keys in the
  specified range.
 -P <numreq>        Pipeline <numreq> requests. Default 1 (no pipeline).              # 通过管道传输<numreq>请求
 -e                 If server replies with errors, show them on stdout.
                    (no more than 1 error per second is displayed)
 -q                 Quiet. Just show query/sec values                                  # 压测退出显示吞吐量
 --precision        Number of decimal places to display in latency output (default 0)
 --csv              Output in CSV format                                               # CSV格式输出
 -l                 Loop. Run the tests forever                                        # 生成循环,永久执行测试
 -t <tests>         Only run the comma separated list of tests. The test               # 仅运行以逗号分隔的测试命令列表
                    names are the same as the ones produced as output.
 -I                 Idle mode. Just open N idle connections and wait.                  # Idle模式,仅打开N个idle连接并等待.

Examples:

 Run the benchmark with the default configuration against 127.0.0.1:6379:
   $ redis-benchmark

 Use 20 parallel clients, for a total of 100k requests, against 192.168.1.1:
   $ redis-benchmark -h 192.168.1.1 -p 6379 -n 100000 -c 20

 Fill 127.0.0.1:6379 with about 1 million keys only using the SET test:
   $ redis-benchmark -t set -n 1000000 -r 100000000

 Benchmark 127.0.0.1:6379 for a few commands producing CSV output:
   $ redis-benchmark -t ping,set,get -n 100000 --csv

 Benchmark a specific command line:
   $ redis-benchmark -r 10000 -n 10000 eval 'return redis.call("ping")' 0

 Fill a list with 10000 random elements:
   $ redis-benchmark -r 10000 -n 10000 lpush mylist __rand_int__

 On user specified command lines __rand_int__ is replaced with a random integer
 with a range of values selected by the -r option.
```
### (3). 压测过程
```
# 机器配置如下:
#   MacBook Pro (Retina, 13-inch, Early 2015)
#   2.9 GHz 双核Intel Core i5
#   8 GB 1867 MHz DDR3

# *************************每一次都跑三轮,取中间值为参考.*************************
# 10个连接,100W请求
# CPU占用率:71%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 10 -n 1000000 -t set,get -d 50
====== SET ======
34191.54 requests per second
35074.18 requests per second
35518.93 requests per second
====== GET ======  
35364.43 requests per second
35973.81 requests per second
36121.95 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:50:05 23 49 11   0 1.61MB 3.57MB    1 38.4k     0     0    100 38.4k     0    0B
16:50:06 22 48 11   0 1.61MB 3.57MB    1 37.1k     0     0    100 37.1k     0    0B
16:50:07 22 48 11   0 1.61MB 3.57MB    1 37.1k     0     0    100 37.1k     0    0B


# CPU占用率:75%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 20 -n 1000000 -t set,get -d 50
====== SET ======
37154.00 requests per second
38648.84 requests per second
39195.70 requests per second
====== GET ======
38119.93 requests per second
39373.18 requests per second
40293.34 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:52:02 27 50 21   0 2.15MB 3.61MB    1 40.6k     0     0      -     0     0    0B
16:52:03 27 50 21   0 2.15MB 3.61MB    1 41.4k     0     0      -     0     0    0B
16:52:05 27 51 21   0 2.15MB 3.61MB    1 41.1k     0     0      -     0     0    0B


# CPU占用率:76%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 30 -n 1000000 -t set,get -d 50
====== SET ======
40859.69 requests per second
41421.59 requests per second
41839.25 requests per second
====== GET ======
37639.27 requests per second
40866.37 requests per second
43306.92 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:53:18 25 54 31   0 2.68MB 3.83MB    1 47.5k     0     0    100 47.5k     0    0B
16:53:19 25 54 31   0 2.68MB 3.83MB    1 48.2k     0     0    100 48.2k     0    0B
16:53:20 25 55 31   0 2.68MB 3.83MB    1 49.0k     0     0    100 49.0k     0    0B



# CPU占用率:77%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 40 -n 1000000 -t set,get -d 50
====== SET ======
39560.09 requests per second
43775.17 requests per second
44648.84 requests per second
====== GET ======
44464.21 requests per second
45008.55 requests per second
45545.64 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:54:08 29 53 41   0 3.20MB 3.93MB    1 48.9k     0     0      -     0     0    0B
16:54:09 29 53 41   0 3.20MB 3.93MB    1 49.1k     0     0      -     0     0    0B
16:54:10 29 53 41   0 3.20MB 3.93MB    1 48.4k     0     0      -     0     0    0B
16:54:11 25 52 41   0 3.20MB 3.93MB    1 47.1k     0     0    100 34.8k     0    0B


# CPU占用率:78%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 50 -n 1000000 -t set,get -d 50
====== SET ======
44935.74 requests per second
45802.23 requests per second
46354.24 requests per second
====== GET ======
45985.47 requests per second
45369.99 requests per second
46959.38 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:54:46 30 54 51   0 3.73MB 3.98MB    1 51.6k     0     0      -     0     0    0B
16:54:47 30 54 51   0 3.73MB 3.98MB    1 52.4k     0     0      -     0     0    0B
16:54:48 30 55 51   0 3.73MB 3.98MB    1 50.5k     0     0      -     0     0    0B
16:54:49 30 54 51   0 3.73MB 3.98MB    1 50.1k     0     0      -     0     0    0B
16:54:50 29 53 51   0 3.73MB 3.98MB    1 50.0k     0     0      -     0     0    0B


# CPU占用率:78%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 60 -n 1000000 -t set,get -d 50
====== SET ======
46371.43 requests per second
47281.32 requests per second
47456.34 requests per second
====== GET ======
46620.05 requests per second
47865.21 requests per second
49062.90 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:55:51 30 54 61   0 4.24MB 4.35MB    1 53.5k     0     0      -     0     0    0B
16:55:53 30 55 61   0 4.24MB 4.35MB    1 54.5k     0     0      -     0     0    0B
16:55:54 30 54 61   0 4.24MB 4.35MB    1 53.6k     0     0      -     0     0    0B
16:55:55 31 55 61   0 4.24MB 4.35MB    1 55.8k     0     0      -     0     0    0B


# CPU占用率:80%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 70 -n 1000000 -t set,get -d 50
====== SET ======
46172.31 requests per second
48766.21 requests per second
49470.66 requests per second
====== GET ======
45601.71 requests per second
47846.89 requests per second
49441.31 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:56:48 30 54 71   0 4.76MB 4.68MB    1 51.1k     0     0      -     0     0    0B
16:56:49 30 55 71   0 4.76MB 4.68MB    1 54.2k     0     0      -     0     0    0B
16:56:50 31 55 71   0 4.76MB 4.68MB    1 55.2k     0     0      -     0     0    0B
16:56:51 31 55 71   0 4.76MB 4.68MB    1 55.1k     0     0      -     0     0    0B
16:56:52 31 55 71   0 4.76MB 4.68MB    1 54.4k     0     0      -     0     0    0B

# CPU占用率:80%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 80 -n 1000000 -t set,get -d 50
====== SET ======
47454.09 requests per second
49343.73 requests per second
49664.76 requests per second
====== GET ======
46763.93 requests per second
48654.70 requests per second
49719.09 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:57:34 31 56 81   0 5.29MB 4.84MB    1 52.8k     0     0      -     0     0    0B
16:57:35 31 56 81   0 5.29MB 4.85MB    1 53.4k     0     0      -     0     0    0B
16:57:36 30 55 81   0 5.29MB 4.85MB    1 52.6k     0     0      -     0     0    0B


# CPU占用率:80%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 90 -n 1000000 -t set,get -d 50
====== SET ======
47653.09 requests per second
48880.63 requests per second
49640.11 requests per second
====== GET ======
46174.45 requests per second
47456.34 requests per second
49046.05 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:58:32 31 55 91   0 5.81MB 5.13MB    1 55.3k     0     0      -     0     0    0B
16:58:33 31 55 91   0 5.81MB 5.13MB    1 55.9k     0     0      -     0     0    0B
16:58:34 31 56 91   0 5.81MB 5.13MB    1 57.6k     0     0      -     0     0    0B
16:58:35 31 56 91   0 5.81MB 5.13MB    1 54.3k     0     0      -     0     0    0B
16:58:36 31 55 91   0 5.81MB 5.13MB    1 53.1k     0     0      -     0     0    0B


# CPU占用率:80%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 100 -n 1000000 -t set,get -d 50
====== SET ======
49870.34 requests per second
49877.80 requests per second
51007.40 requests per second
====== GET ======
45526.97 requests per second
48797.15 requests per second
48763.84 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
16:59:24 32 56 101   0 6.33MB 5.68MB    1 57.9k     0     0      -     0     0    0B
16:59:25 31 56 101   0 6.33MB 5.68MB    1 58.0k     0     0      -     0     0    0B
16:59:26 31 56 101   0 6.33MB 5.68MB    1 57.9k     0     0      -     0     0    0B


# CPU占用率:80%
lixin-macbook:src lixin$ ./redis-benchmark -h 127.0.0.1 -p 6379 -c 110 -n 1000000 -t set,get -d 50
====== SET ======
46229.95 requests per second
48339.54 requests per second
49865.36 requests per second
====== GET ======
46214.99 requests per second
47386.62 requests per second
51305.73 requests per second

# redis-stat 监控数据(看第9列[cmd/s])
17:00:45 31 56 111   0 6.85MB 5.90MB    1 56.7k     0     0      -     0     0    0B
17:00:46 31 56 111   0 6.85MB 5.90MB    1 55.7k     0     0      -     0     0    0B
17:00:47 31 56 111   0 6.85MB 5.90MB    1 54.8k     0     0      -     0     0    0B
17:00:48 32 57 111   0 6.85MB 5.90MB    1 57.8k     0     0      -     0     0    0B

```
### (4). 压测结论
```
从上面的基准测试可以看出,当并发连接数达到:100之后,就开始出现了拐点(QPS不升,反降)
所以,我这台机器上Redis,最佳配置并发数(连接池数量),为100个,SET操作的QPS为:49877/s,GET操作的QPS为:48797/s. 
通地redis-stat监控的数据来看,Redis服务器,每秒的吞吐量为:5.7W左右.
```