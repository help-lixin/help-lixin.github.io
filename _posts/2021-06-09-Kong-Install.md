---
layout: post
title: 'Kong 安装(二)'
date: 2021-06-09
author: 李新
tags:  Kong
---

### (1). PostgreSQL配置
> ["PostgreSQL安装参考文档"](/2021/06/09/PostgreSQL-Install.html)    

```
# 1. 登录psql
[lixin@postgre-sql pgsql]$ psql  -h 127.0.0.1 -d postgres
# 2. 创建用户
postgres=# CREATE USER kong WITH PASSWORD '123456';
CREATE ROLE
# 3. 创建数据库
postgres=# CREATE DATABASE kong OWNER kong;
CREATE DATABASE
# 4. 授权
postgres=# GRANT ALL PRIVILEGES ON DATABASE kong to kong;
GRANT
```
### (2). kong安装
> ["kong官网"](https://docs.konghq.com/gateway-oss/2.4.x/getting-started/quickstart/)

```
# https://bintray.com/kong/kong-community-edition-rpm/centos#files/centos%2F7
> wget https://bintray.com/kong/kong-community-edition-rpm/download_file?file_path=centos%2F7%2Fkong-community-edition-1.1.2.el7.noarch.rpm
> yum -y  install epel-release
> yum -y install  kong-community-edition-1.1.2.el7.noarch.rpm
```
### (3). kong配置(/etc/kong/kong.conf)
```
[root@tomcat-1 ~]# cp /etc/kong/kong.conf.default  /etc/kong/kong.conf
[root@tomcat-1 ~]# vi /etc/kong/kong.conf
#------------------------------------------------------------------------------
# DATASTORE
#------------------------------------------------------------------------------
# 配置如下项即可.

database = postgres             # Determines which of PostgreSQL or Cassandra
								# this node will use as its datastore.
								# Accepted values are `postgres`,
								# `cassandra`, and `off`.

pg_host = 10.211.55.101         # Host of the Postgres server.
pg_port = 5432                  # Port of the Postgres server.
pg_timeout = 5000               # Defines the timeout (in ms), for connecting,
								# reading and writing.

pg_user = kong                  # Postgres user.
pg_password = 123456            # Postgres user's password.
pg_database = kong              # The database name to connect to.
```
### (4). 初始化表结构信息
```
[root@tomcat-1 ~]# kong migrations bootstrap -c /etc/kong/kong.conf
bootstrapping database...
migrating core on database 'kong'...
core migrated up to: 000_base (executed)
core migrated up to: 001_14_to_15 (executed)
core migrated up to: 002_15_to_1 (executed)
core migrated up to: 003_100_to_110 (executed)
migrating oauth2 on database 'kong'...
oauth2 migrated up to: 000_base_oauth2 (executed)
oauth2 migrated up to: 001_14_to_15 (executed)
oauth2 migrated up to: 002_15_to_10 (executed)
migrating acl on database 'kong'...
acl migrated up to: 000_base_acl (executed)
acl migrated up to: 001_14_to_15 (executed)
migrating jwt on database 'kong'...
jwt migrated up to: 000_base_jwt (executed)
jwt migrated up to: 001_14_to_15 (executed)
migrating basic-auth on database 'kong'...
basic-auth migrated up to: 000_base_basic_auth (executed)
basic-auth migrated up to: 001_14_to_15 (executed)
migrating key-auth on database 'kong'...
key-auth migrated up to: 000_base_key_auth (executed)
key-auth migrated up to: 001_14_to_15 (executed)
migrating rate-limiting on database 'kong'...
rate-limiting migrated up to: 000_base_rate_limiting (executed)
rate-limiting migrated up to: 001_14_to_15 (executed)
rate-limiting migrated up to: 002_15_to_10 (executed)
rate-limiting migrated up to: 003_10_to_112 (executed)
migrating hmac-auth on database 'kong'...
hmac-auth migrated up to: 000_base_hmac_auth (executed)
hmac-auth migrated up to: 001_14_to_15 (executed)
migrating response-ratelimiting on database 'kong'...
response-ratelimiting migrated up to: 000_base_response_rate_limiting (executed)
response-ratelimiting migrated up to: 001_14_to_15 (executed)
response-ratelimiting migrated up to: 002_15_to_10 (executed)
24 migrations processed
24 executed
database is up-to-date
```
### (5). 验证表是否创建成功
```
# 1. 通过kong登录
[lixin@postgre-sql pgsql]$ psql  -h 127.0.0.1 -d postgres -U kong -W
Password for user kong:
psql.bin (10.17)
Type "help" for help.

# 2. 查看有哪些库
postgres=> \l
                          List of databases
   Name    |  Owner  | Encoding | Collate | Ctype | Access privileges
-----------+---------+----------+---------+-------+-------------------
 kong      | kong    | UTF8     | C       | C     | =Tc/kong         +
           |         |          |         |       | kong=CTc/kong
 postgres  | lixin   | UTF8     | C       | C     |
 template0 | lixin   | UTF8     | C       | C     | =c/lixin         +
           |         |          |         |       | lixin=CTc/lixin
 template1 | lixin   | UTF8     | C       | C     | =c/lixin         +
           |         |          |         |       | lixin=CTc/lixin
 test2     | devuser | UTF8     | C       | C     |
(5 rows)

# 3. 切换到kong库
postgres=> \c kong
Password:
You are now connected to database "kong" as user "kong".

# 4. 查看有哪些表
kong=> \dt
                   List of relations
 Schema |             Name              | Type  | Owner
--------+-------------------------------+-------+-------
 public | acls                          | table | kong
 public | apis                          | table | kong
 public | basicauth_credentials         | table | kong
 public | certificates                  | table | kong
 public | cluster_ca                    | table | kong
 public | cluster_events                | table | kong
 public | consumers                     | table | kong
 public | hmacauth_credentials          | table | kong
 public | jwt_secrets                   | table | kong
 public | keyauth_credentials           | table | kong
 public | locks                         | table | kong
 public | oauth2_authorization_codes    | table | kong
 public | oauth2_credentials            | table | kong
 public | oauth2_tokens                 | table | kong
 public | plugins                       | table | kong
 public | ratelimiting_metrics          | table | kong
 public | response_ratelimiting_metrics | table | kong
 public | routes                        | table | kong
 public | schema_meta                   | table | kong
 public | services                      | table | kong
 public | snis                          | table | kong
 public | tags                          | table | kong
 public | targets                       | table | kong
 public | ttls                          | table | kong
 public | upstreams                     | table | kong
(25 rows)
```
### (6). 启动kong
```
# -vv  : 打印详细日志

[root@tomcat-1 ~]# kong start -c /etc/kong/kong.conf --vv
2021/06/10 13:43:53 [verbose] Kong: 1.1.2
2021/06/10 13:43:53 [debug] ngx_lua: 10013
2021/06/10 13:43:53 [debug] nginx: 1013006
2021/06/10 13:43:53 [debug] Lua: LuaJIT 2.1.0-beta3
2021/06/10 13:43:53 [verbose] reading config file at /etc/kong/kong.conf
2021/06/10 13:43:53 [debug] reading environment variables
2021/06/10 13:43:53 [debug] admin_access_log = "logs/admin_access.log"
2021/06/10 13:43:53 [debug] admin_error_log = "logs/error.log"
2021/06/10 13:43:53 [debug] admin_listen = {"127.0.0.1:8001","127.0.0.1:8444 ssl"}
2021/06/10 13:43:53 [debug] anonymous_reports = true
2021/06/10 13:43:53 [debug] cassandra_consistency = "ONE"
2021/06/10 13:43:53 [debug] cassandra_contact_points = {"127.0.0.1"}
2021/06/10 13:43:53 [debug] cassandra_data_centers = {"dc1:2","dc2:3"}
2021/06/10 13:43:53 [debug] cassandra_keyspace = "kong"
2021/06/10 13:43:53 [debug] cassandra_lb_policy = "RequestRoundRobin"
2021/06/10 13:43:53 [debug] cassandra_port = 9042
2021/06/10 13:43:53 [debug] cassandra_repl_factor = 1
2021/06/10 13:43:53 [debug] cassandra_repl_strategy = "SimpleStrategy"
2021/06/10 13:43:53 [debug] cassandra_schema_consensus_timeout = 10000
2021/06/10 13:43:53 [debug] cassandra_ssl = false
2021/06/10 13:43:53 [debug] cassandra_ssl_verify = false
2021/06/10 13:43:53 [debug] cassandra_timeout = 5000
2021/06/10 13:43:53 [debug] cassandra_username = "kong"
2021/06/10 13:43:53 [debug] client_body_buffer_size = "8k"
2021/06/10 13:43:53 [debug] client_max_body_size = "0"
2021/06/10 13:43:53 [debug] client_ssl = false
2021/06/10 13:43:53 [debug] database = "postgres"
2021/06/10 13:43:53 [debug] db_cache_ttl = 0
2021/06/10 13:43:53 [debug] db_resurrect_ttl = 30
2021/06/10 13:43:53 [debug] db_update_frequency = 5
2021/06/10 13:43:53 [debug] db_update_propagation = 0
2021/06/10 13:43:53 [debug] dns_error_ttl = 1
2021/06/10 13:43:53 [debug] dns_hostsfile = "/etc/hosts"
2021/06/10 13:43:53 [debug] dns_no_sync = false
2021/06/10 13:43:53 [debug] dns_not_found_ttl = 30
2021/06/10 13:43:53 [debug] dns_order = {"LAST","SRV","A","CNAME"}
2021/06/10 13:43:53 [debug] dns_resolver = {}
2021/06/10 13:43:53 [debug] dns_stale_ttl = 4
2021/06/10 13:43:53 [debug] error_default_type = "text/plain"
2021/06/10 13:43:53 [debug] headers = {"server_tokens","latency_tokens"}
2021/06/10 13:43:53 [debug] log_level = "notice"
2021/06/10 13:43:53 [debug] lua_package_cpath = ""
2021/06/10 13:43:53 [debug] lua_package_path = "./?.lua;./?/init.lua;"
2021/06/10 13:43:53 [debug] lua_socket_pool_size = 30
2021/06/10 13:43:53 [debug] lua_ssl_verify_depth = 1
2021/06/10 13:43:53 [debug] mem_cache_size = "128m"
2021/06/10 13:43:53 [debug] nginx_admin_directives = {}
2021/06/10 13:43:53 [debug] nginx_daemon = "on"
2021/06/10 13:43:53 [debug] nginx_http_directives = {}
2021/06/10 13:43:53 [debug] nginx_optimizations = true
2021/06/10 13:43:53 [debug] nginx_proxy_directives = {}
2021/06/10 13:43:53 [debug] nginx_sproxy_directives = {}
2021/06/10 13:43:53 [debug] nginx_stream_directives = {}
2021/06/10 13:43:53 [debug] nginx_user = "nobody nobody"
2021/06/10 13:43:53 [debug] nginx_worker_processes = "auto"
2021/06/10 13:43:53 [debug] origins = {}
2021/06/10 13:43:53 [debug] pg_database = "kong"
2021/06/10 13:43:53 [debug] pg_host = "10.211.55.101"
2021/06/10 13:43:53 [debug] pg_password = "******"
2021/06/10 13:43:53 [debug] pg_port = 5432
2021/06/10 13:43:53 [debug] pg_ssl = false
2021/06/10 13:43:53 [debug] pg_ssl_verify = false
2021/06/10 13:43:53 [debug] pg_timeout = 5000
2021/06/10 13:43:53 [debug] pg_user = "kong"
2021/06/10 13:43:53 [debug] plugins = {"bundled"}
2021/06/10 13:43:53 [debug] prefix = "/usr/local/kong/"
2021/06/10 13:43:53 [debug] proxy_access_log = "logs/access.log"
2021/06/10 13:43:53 [debug] proxy_error_log = "logs/error.log"
2021/06/10 13:43:53 [debug] proxy_listen = {"0.0.0.0:8000","0.0.0.0:8443 ssl"}
2021/06/10 13:43:53 [debug] real_ip_header = "X-Real-IP"
2021/06/10 13:43:53 [debug] real_ip_recursive = "off"
2021/06/10 13:43:53 [debug] ssl_cipher_suite = "modern"
2021/06/10 13:43:53 [debug] ssl_ciphers = "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256"
2021/06/10 13:43:53 [debug] stream_listen = {"off"}
2021/06/10 13:43:53 [debug] trusted_ips = {}
2021/06/10 13:43:53 [debug] upstream_keepalive = 60
2021/06/10 13:43:53 [verbose] prefix in use: /usr/local/kong
2021/06/10 13:43:53 [debug] loading subsystems migrations...
2021/06/10 13:43:53 [verbose] retrieving database schema state...
2021/06/10 13:43:53 [verbose] schema state retrieved
2021/06/10 13:43:53 [verbose] preparing nginx prefix directory at /usr/local/kong
2021/06/10 13:43:53 [verbose] SSL enabled, no custom certificate set: using default certificate
2021/06/10 13:43:53 [verbose] generating default SSL certificate and key
2021/06/10 13:43:53 [verbose] Admin SSL enabled, no custom certificate set: using default certificate
2021/06/10 13:43:53 [verbose] generating admin SSL certificate and key
2021/06/10 13:43:53 [warn] ulimit is currently set to "1024". For better performance set it to at least "4096" using "ulimit -n"
2021/06/10 13:43:54 [debug] searching for OpenResty 'nginx' executable
2021/06/10 13:43:54 [debug] /usr/local/openresty/nginx/sbin/nginx -v: 'nginx version: openresty/1.13.6.2'
2021/06/10 13:43:54 [debug] found OpenResty 'nginx' executable at /usr/local/openresty/nginx/sbin/nginx
2021/06/10 13:43:54 [debug] testing nginx configuration: KONG_NGINX_CONF_CHECK=true /usr/local/openresty/nginx/sbin/nginx -t -p /usr/local/kong -c nginx.conf
2021/06/10 13:43:54 [debug] searching for OpenResty 'nginx' executable
2021/06/10 13:43:54 [debug] /usr/local/openresty/nginx/sbin/nginx -v: 'nginx version: openresty/1.13.6.2'
2021/06/10 13:43:54 [debug] found OpenResty 'nginx' executable at /usr/local/openresty/nginx/sbin/nginx
2021/06/10 13:43:54 [debug] sending signal to pid at: /usr/local/kong/pids/nginx.pid
2021/06/10 13:43:54 [debug] kill -0 `cat /usr/local/kong/pids/nginx.pid` >/dev/null 2>&1
2021/06/10 13:43:54 [debug] starting nginx: /usr/local/openresty/nginx/sbin/nginx -p /usr/local/kong -c nginx.conf
2021/06/10 13:43:54 [debug] nginx started
2021/06/10 13:43:54 [info] Kong started
```
### (7). 检查kong
```
[root@tomcat-1 ~]# kong health
nginx.......running
Kong is healthy at /usr/local/kong
```
### (8). 查看些网络状态
```
# 注意:8000/8443不受网络限制.
#     8001/8444只允许本机可以访问.
#  我之所以看这个点,是因为:8001是管理员端口,如果也允许外网访问的话,安全又是个大问题.   
#  8000: 侦听来自客户端的传入HTTP流量,并将其转发到您的上游服务.
#  8443: 在其上侦听传入的HTTPS流量.此端口的行为与:8000端口相似,不同之处在于它仅需要HTTPS流量.可以通过配置文件禁用此端口.
#  8001: 用于配置Kong的Admin API在其上侦听.
#  8444: Admin API 在其上侦听HTTPS流量.
[root@tomcat-1 ~]# netstat -tlnp|grep nginx
tcp        0      0 0.0.0.0:8443            0.0.0.0:*               LISTEN      19215/nginx: master
tcp        0      0 127.0.0.1:8444          0.0.0.0:*               LISTEN      19215/nginx: master
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      19215/nginx: master
tcp        0      0 127.0.0.1:8001          0.0.0.0:*               LISTEN      19215/nginx: master
```
### (9). 停止kong
```
[root@tomcat-1 ~]# kong stop
Kong stopped
```