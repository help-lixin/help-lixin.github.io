---
layout: post
title: 'Docker 自定义镜像仓库-Harbor'
date: 2021-01-10
author: 李新
tags: Docker Harbor
---

### (1). Docker安装(略)
```
[root@registry ~]# docker -v
	Docker version 18.06.1-ce, build e68fc7a

# 配置镜像加速器
[root@registry ~]# cat /etc/docker/daemon.json
{
   "registry-mirrors": ["http://hub-mirror.c.163.com"]
}

# 重启docker
[root@registry ~]# systemctl restart docker.service
```

### (2). 准备工作
```
# 配置域名(或者IP,我这里以域名为测试)
[root@registry ~]# hostname
	registry.lixin.help
```
### (3). Harbor是通过:Docker-Compose进行编排.
```
[root@registry ~]# curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
[root@registry ~]# chmod u+x /usr/local/bin/docker-compose
[root@registry ~]# docker-compose -v
	docker-compose version 1.27.4, build 40524192
```
### (4). 下载Harbor
> ["Harbor下载"](https://github.com/goharbor/harbor)
> 我下载的版本为:harbor-offline-installer-v2.1.2.tgz

```
# 下载
[root@registry ~]# wget https://github.com/goharbor/harbor/releases/download/v2.1.2/harbor-offline-installer-v2.1.2.tgz
# 解压到:/usr/local
[root@registry ~]# tar -zxvf harbor-offline-installer-v2.1.2.tgz -C /usr/local
```
### (5). 配置harbor.yml
> 我共改动了四处:   
> 1. https.certificate     
> 2. https.private_key     
> 3. harbor_admin_password     
> 4. hostname


```
[root@registry ~]# cd /usr/local/harbor/
# 重命名配置文件
[root@registry harbor]# mv harbor.yml.tmpl harbor.yml
```

```
# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: registry.lixin.help  # TODO 

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80

# https related config
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /data/cert/registry.lixin.help.crt  # TODO
  private_key: /data/cert/registry.lixin.help.key  # TODO

# # Uncomment following will enable tls communication between all harbor components
# internal_tls:
#   # set enabled to true means internal tls is enabled
#   enabled: true
#   # put your cert and key files on dir
#   dir: /etc/harbor/tls/internal

# Uncomment external_url if you want to enable external proxy
# And when it enabled the hostname will no longer used
# external_url: https://reg.mydomain.com:8433

# The initial password of Harbor admin
# It only works in first time to install harbor
# Remember Change the admin password from UI after launching Harbor.
harbor_admin_password: admin  # TODO

# Harbor DB configuration
database:
  # The password for the root user of Harbor DB. Change this before any production use.
  password: root123
  # The maximum number of connections in the idle connection pool. If it <=0, no idle connections are retained.
  max_idle_conns: 50
  # The maximum number of open connections to the database. If it <= 0, then there is no limit on the number of open connections.
  # Note: the default number of connections is 1024 for postgres of harbor.
  max_open_conns: 1000

# The default data volume
data_volume: /data

# Harbor Storage settings by default is using /data dir on local filesystem
# Uncomment storage_service setting If you want to using external storage
# storage_service:
#   # ca_bundle is the path to the custom root ca certificate, which will be injected into the truststore
#   # of registry's and chart repository's containers.  This is usually needed when the user hosts a internal storage with self signed certificate.
#   ca_bundle:

#   # storage backend, default is filesystem, options include filesystem, azure, gcs, s3, swift and oss
#   # for more info about this configuration please refer https://docs.docker.com/registry/configuration/
#   filesystem:
#     maxthreads: 100
#   # set disable to true when you want to disable registry redirect
#   redirect:
#     disabled: false

# Clair configuration
clair:
  # The interval of clair updaters, the unit is hour, set to 0 to disable the updaters.
  updaters_interval: 12

# Trivy configuration
#
# Trivy DB contains vulnerability information from NVD, Red Hat, and many other upstream vulnerability databases.
# It is downloaded by Trivy from the GitHub release page https://github.com/aquasecurity/trivy-db/releases and cached
# in the local file system. In addition, the database contains the update timestamp so Trivy can detect whether it
# should download a newer version from the Internet or use the cached one. Currently, the database is updated every
# 12 hours and published as a new release to GitHub.
trivy:
  # ignoreUnfixed The flag to display only fixed vulnerabilities
  ignore_unfixed: false
  # skipUpdate The flag to enable or disable Trivy DB downloads from GitHub
  #
  # You might want to enable this flag in test or CI/CD environments to avoid GitHub rate limiting issues.
  # If the flag is enabled you have to download the `trivy-offline.tar.gz` archive manually, extract `trivy.db` and
  # `metadata.json` files and mount them in the `/home/scanner/.cache/trivy/db` path.
  skip_update: false
  #
  # insecure The flag to skip verifying registry certificate
  insecure: false
  # github_token The GitHub access token to download Trivy DB
  #
  # Anonymous downloads from GitHub are subject to the limit of 60 requests per hour. Normally such rate limit is enough
  # for production operations. If, for any reason, it's not enough, you could increase the rate limit to 5000
  # requests per hour by specifying the GitHub access token. For more details on GitHub rate limiting please consult
  # https://developer.github.com/v3/#rate-limiting
  #
  # You can create a GitHub token by following the instructions in
  # https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line
  #
  # github_token: xxx

jobservice:
  # Maximum number of job workers in job service
  max_job_workers: 10

notification:
  # Maximum retry count for webhook job
  webhook_job_max_retry: 10

chart:
  # Change the value of absolute_url to enabled can enable absolute url in chart
  absolute_url: disabled

# Log configurations
log:
  # options are debug, info, warning, error, fatal
  level: info
  # configs for logs in local storage
  local:
    # Log files are rotated log_rotate_count times before being removed. If count is 0, old versions are removed rather than rotated.
    rotate_count: 50
    # Log files are rotated only if they grow bigger than log_rotate_size bytes. If size is followed by k, the size is assumed to be in kilobytes.
    # If the M is used, the size is in megabytes, and if G is used, the size is in gigabytes. So size 100, size 100k, size 100M and size 100G
    # are all valid.
    rotate_size: 200M
    # The directory on your host that store log
    location: /var/log/harbor

  # Uncomment following lines to enable external syslog endpoint.
  # external_endpoint:
  #   # protocol used to transmit log to external endpoint, options is tcp or udp
  #   protocol: tcp
  #   # The host of external endpoint
  #   host: localhost
  #   # Port of external endpoint
  #   port: 5140

#This attribute is for migrator to detect the version of the .cfg file, DO NOT MODIFY!
_version: 2.0.0

# Uncomment external_database if using external database.
# external_database:
#   harbor:
#     host: harbor_db_host
#     port: harbor_db_port
#     db_name: harbor_db_name
#     username: harbor_db_username
#     password: harbor_db_password
#     ssl_mode: disable
#     max_idle_conns: 2
#     max_open_conns: 0
#   clair:
#     host: clair_db_host
#     port: clair_db_port
#     db_name: clair_db_name
#     username: clair_db_username
#     password: clair_db_password
#     ssl_mode: disable
#   notary_signer:
#     host: notary_signer_db_host
#     port: notary_signer_db_port
#     db_name: notary_signer_db_name
#     username: notary_signer_db_username
#     password: notary_signer_db_password
#     ssl_mode: disable
#   notary_server:
#     host: notary_server_db_host
#     port: notary_server_db_port
#     db_name: notary_server_db_name
#     username: notary_server_db_username
#     password: notary_server_db_password
#     ssl_mode: disable

# Uncomment external_redis if using external Redis server
# external_redis:
#   # support redis, redis+sentinel
#   # host for redis: <host_redis>:<port_redis>
#   # host for redis+sentinel:
#   #  <host_sentinel1>:<port_sentinel1>,<host_sentinel2>:<port_sentinel2>,<host_sentinel3>:<port_sentinel3>
#   host: redis:6379
#   password:
#   # sentinel_master_set must be set to support redis+sentinel
#   #sentinel_master_set:
#   # db_index 0 is for core, it's unchangeable
#   registry_db_index: 1
#   jobservice_db_index: 2
#   chartmuseum_db_index: 3
#   clair_db_index: 4
#   trivy_db_index: 5
#   idle_timeout_seconds: 30

# Uncomment uaa for trusting the certificate of uaa instance that is hosted via self-signed cert.
# uaa:
#   ca_file: /path/to/ca

# Global proxy
# Config http proxy for components, e.g. http://my.proxy.com:3128
# Components doesn't need to connect to each others via http proxy.
# Remove component from `components` array if want disable proxy
# for it. If you want use proxy for replication, MUST enable proxy
# for core and jobservice, and set `http_proxy` and `https_proxy`.
# Add domain to the `no_proxy` field, when you want disable proxy
# for some special registry.
proxy:
  http_proxy:
  https_proxy:
  no_proxy:
  components:
    - core
    - jobservice
    - clair
    - trivy
```
### (6). 生成证书脚本(create_cert.sh)

```
#!/bin/bash

# 在该目录下操作生成证书.
mkdir -p /data/cert
cd /data/cert

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=registry.lixin.help" -key ca.key -out ca.crt
openssl genrsa -out registry.lixin.help.key 4096
openssl req -sha512 -new -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=registry.lixin.help" -key registry.lixin.help.key -out registry.lixin.help.csr

# 基于IP配置
# cat > v3.ext <<-EOF
# authorityKeyIdentifier=keyid,issuer
# basicConstraints=CA:FALSE
# keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
# extendedKeyUsage = serverAuth
# subjectAltName = IP:10.211.55.90
# EOF

# 基于domain配置
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=registry.lixin.help
DNS.2=harbor
DNS.3=ks-allinone
EOF

openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial -in registry.lixin.help.csr -out registry.lixin.help.crt
    
openssl x509 -inform PEM -in registry.lixin.help.crt -out registry.lixin.help.cert

cp registry.lixin.help.crt /etc/pki/ca-trust/source/anchors/registry.lixin.help.crt 
update-ca-trust
```
### (7). 执行证书(create_cert.sh)

```
# 给脚本添加可执行权限
[root@registry ~]# chmod u+x create_cert.sh   
# 执行证书
[root@registry ~]# ./create_cert.sh
Generating RSA private key, 4096 bit long modulus
.....................................................................................................................................................................................................................++
...........................................................................................................................................................................................++
e is 65537 (0x10001)
Generating RSA private key, 4096 bit long modulus
.....++
.....................................................................................................................................++
e is 65537 (0x10001)
Signature ok
subject=/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=registry.lixin.help
Getting CA Private Key
```

### (8). 检查环境
>  记得配置加速器,否则,会报错(Error response from daemon: Get https://registry-1.docker.io/v2/: net/http: TLS handshake timeout.)     
>  记得在harbor.yml中配置hostname,否则会报错(ERROR:root:Please specify hostname)

```
[root@registry harbor]# ./prepare
prepare base dir is set to /usr/local/harbor
Generated configuration file: /config/portal/nginx.conf
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /data/secret/keys/secretkey
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir
```
### (9). 安装(install.sh)
```
[root@registry harbor]# ./install.sh

[Step 0]: checking if docker is installed ...

Note: docker version: 18.06.1

[Step 1]: checking docker-compose is installed ...

Note: docker-compose version: 1.27.4

[Step 2]: loading Harbor images ...
94095a8e8b3f: Loading layer [==================================================>]  4.833MB/4.833MB
1108e2ba7cec: Loading layer [==================================================>]  66.44MB/66.44MB
6c2b0a255ab5: Loading layer [==================================================>]  3.072kB/3.072kB
e7622726ff27: Loading layer [==================================================>]  4.096kB/4.096kB
048dac647335: Loading layer [==================================================>]  67.27MB/67.27MB
Loaded image: goharbor/chartmuseum-photon:v2.1.2
Loaded image: goharbor/prepare:v2.1.2
8f90e0f7e20e: Loading layer [==================================================>]  74.79MB/74.79MB
9122480e6e2b: Loading layer [==================================================>]  3.584kB/3.584kB
96b81bcc41aa: Loading layer [==================================================>]  3.072kB/3.072kB
25a66d60d4fc: Loading layer [==================================================>]   2.56kB/2.56kB
3b73d0b0cf0a: Loading layer [==================================================>]  3.072kB/3.072kB
cbbfae699a86: Loading layer [==================================================>]  3.584kB/3.584kB
0de4ef80b26f: Loading layer [==================================================>]  12.29kB/12.29kB
dfd3240a23f9: Loading layer [==================================================>]  3.584kB/3.584kB
Loaded image: goharbor/harbor-log:v2.1.2
c6034ea793b1: Loading layer [==================================================>]  4.834MB/4.834MB
e5d1ea04bec1: Loading layer [==================================================>]  4.096kB/4.096kB
e11295fabb86: Loading layer [==================================================>]  20.51MB/20.51MB
b68370e98b64: Loading layer [==================================================>]  3.072kB/3.072kB
8807070be0a2: Loading layer [==================================================>]  25.91MB/25.91MB
bd876d3a13ce: Loading layer [==================================================>]  47.24MB/47.24MB
Loaded image: goharbor/harbor-registryctl:v2.1.2
575b7bd5c8d3: Loading layer [==================================================>]  4.834MB/4.834MB
47f2dab58d3d: Loading layer [==================================================>]  4.096kB/4.096kB
35ca9e39ce9d: Loading layer [==================================================>]  3.072kB/3.072kB
29f4d5d02c19: Loading layer [==================================================>]  9.427MB/9.427MB
4bf398616171: Loading layer [==================================================>]  10.25MB/10.25MB
Loaded image: goharbor/clair-adapter-photon:v2.1.2
def1f319f1d4: Loading layer [==================================================>]  63.66MB/63.66MB
ac5078f0ac3a: Loading layer [==================================================>]  77.63MB/77.63MB
1c4925a03102: Loading layer [==================================================>]  6.144kB/6.144kB
5fd4eef8ee1f: Loading layer [==================================================>]   2.56kB/2.56kB
8e2da20b4af4: Loading layer [==================================================>]   2.56kB/2.56kB
b107144b2972: Loading layer [==================================================>]   2.56kB/2.56kB
7a25e864c403: Loading layer [==================================================>]   2.56kB/2.56kB
52f6a7163db3: Loading layer [==================================================>]  11.26kB/11.26kB
Loaded image: goharbor/harbor-db:v2.1.2
f6d4191e1a29: Loading layer [==================================================>]  7.973MB/7.973MB
645299ad0a78: Loading layer [==================================================>]  3.584kB/3.584kB
c55c25e7d454: Loading layer [==================================================>]   2.56kB/2.56kB
a8c4eb133a5c: Loading layer [==================================================>]  63.94MB/63.94MB
445d8e5f27ac: Loading layer [==================================================>]  64.76MB/64.76MB
Loaded image: goharbor/harbor-jobservice:v2.1.2
71c0f1e1c665: Loading layer [==================================================>]  111.7MB/111.7MB
fa5c4de407dc: Loading layer [==================================================>]  12.59MB/12.59MB
ec3a9ab38eee: Loading layer [==================================================>]  3.072kB/3.072kB
e60d9fb56586: Loading layer [==================================================>]  49.15kB/49.15kB
6964bfece355: Loading layer [==================================================>]  4.096kB/4.096kB
49123e60ab32: Loading layer [==================================================>]  13.46MB/13.46MB
Loaded image: goharbor/clair-photon:v2.1.2
514fec0dd861: Loading layer [==================================================>]  4.828MB/4.828MB
6b3b8694b24f: Loading layer [==================================================>]  6.343MB/6.343MB
3f29a3da5d0f: Loading layer [==================================================>]  14.43MB/14.43MB
30d6dcebac1c: Loading layer [==================================================>]  27.97MB/27.97MB
a2a9cfe73704: Loading layer [==================================================>]  22.02kB/22.02kB
f856babcaf4c: Loading layer [==================================================>]  14.43MB/14.43MB
Loaded image: goharbor/notary-signer-photon:v2.1.2
af9061bbce22: Loading layer [==================================================>]  6.681MB/6.681MB
b437d20b85e6: Loading layer [==================================================>]  8.993MB/8.993MB
b2b5b1747c83: Loading layer [==================================================>]  173.6kB/173.6kB
3f0778faeb54: Loading layer [==================================================>]  152.6kB/152.6kB
8a24a83859c7: Loading layer [==================================================>]  66.56kB/66.56kB
21044216add7: Loading layer [==================================================>]  17.41kB/17.41kB
bbb02c17475d: Loading layer [==================================================>]  15.36kB/15.36kB
Loaded image: goharbor/harbor-portal:v2.1.2
af841b7c820a: Loading layer [==================================================>]  35.84MB/35.84MB
cf51963a6cd7: Loading layer [==================================================>]  3.072kB/3.072kB
4df8dac59587: Loading layer [==================================================>]   59.9kB/59.9kB
49e7121a2126: Loading layer [==================================================>]  61.95kB/61.95kB
Loaded image: goharbor/redis-photon:v2.1.2
0948acb86fb5: Loading layer [==================================================>]  6.681MB/6.681MB
Loaded image: goharbor/nginx-photon:v2.1.2
9167979fc241: Loading layer [==================================================>]  6.138MB/6.138MB
4003c95457a4: Loading layer [==================================================>]  4.096kB/4.096kB
1f8c2e639b58: Loading layer [==================================================>]  3.072kB/3.072kB
6fab1a3752ba: Loading layer [==================================================>]  23.51MB/23.51MB
1eb7dbd74a34: Loading layer [==================================================>]  9.432MB/9.432MB
488e011e2c0c: Loading layer [==================================================>]  33.76MB/33.76MB
Loaded image: goharbor/trivy-adapter-photon:v2.1.2
f4813c3208bf: Loading layer [==================================================>]  7.974MB/7.974MB
45759140eb84: Loading layer [==================================================>]  3.584kB/3.584kB
1015a890d0ff: Loading layer [==================================================>]   2.56kB/2.56kB
dabb8c8d8330: Loading layer [==================================================>]  54.27MB/54.27MB
f73ff896d59e: Loading layer [==================================================>]  5.632kB/5.632kB
9ce8bd7c9cc2: Loading layer [==================================================>]  60.42kB/60.42kB
d9762d29a79e: Loading layer [==================================================>]  11.78kB/11.78kB
d84c44c9e421: Loading layer [==================================================>]  55.09MB/55.09MB
c224ac6b2bbd: Loading layer [==================================================>]   2.56kB/2.56kB
Loaded image: goharbor/harbor-core:v2.1.2
04e23463e700: Loading layer [==================================================>]  4.834MB/4.834MB
acc8dce6e4b4: Loading layer [==================================================>]  4.096kB/4.096kB
7b1d2c403f11: Loading layer [==================================================>]  3.072kB/3.072kB
20dbd2eb3a74: Loading layer [==================================================>]  20.51MB/20.51MB
caa073fbc55c: Loading layer [==================================================>]  21.33MB/21.33MB
Loaded image: goharbor/registry-photon:v2.1.2
684bb69f09ce: Loading layer [==================================================>]  4.828MB/4.828MB
508b0f09817b: Loading layer [==================================================>]  6.343MB/6.343MB
3404e1d9b9ab: Loading layer [==================================================>]  15.84MB/15.84MB
611369079424: Loading layer [==================================================>]  27.97MB/27.97MB
20e35fb6876b: Loading layer [==================================================>]  22.02kB/22.02kB
b79027d2d589: Loading layer [==================================================>]  15.84MB/15.84MB
Loaded image: goharbor/notary-server-photon:v2.1.2


[Step 3]: preparing environment ...

[Step 4]: preparing harbor configs ...
prepare base dir is set to /usr/local/harbor
Clearing the configuration file: /config/registry/config.yml
Clearing the configuration file: /config/registry/passwd
Clearing the configuration file: /config/db/env
Clearing the configuration file: /config/jobservice/env
Clearing the configuration file: /config/jobservice/config.yml
Clearing the configuration file: /config/core/env
Clearing the configuration file: /config/core/app.conf
Clearing the configuration file: /config/nginx/nginx.conf
Clearing the configuration file: /config/registryctl/env
Clearing the configuration file: /config/registryctl/config.yml
Clearing the configuration file: /config/portal/nginx.conf
Clearing the configuration file: /config/log/logrotate.conf
Clearing the configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/portal/nginx.conf
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
loaded secret from file: /data/secret/keys/secretkey
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir

[Step 5]: starting Harbor ...
Creating network "harbor_harbor" with the default driver
Creating harbor-log ... done
Creating registryctl   ... done
Creating harbor-db     ... done
Creating registry      ... done
Creating redis         ... done
Creating harbor-portal ... done
Creating harbor-core   ... done
Creating nginx             ... done
Creating harbor-jobservice ... done
✔ ----Harbor has been installed and started successfully.----
```
### (10). 登录
> https://10.211.55.90(admin/admin)   

!["Harbor登录界面"](/assets/docker/imgs/harbor-login.jpg)

!["Harbor首页"](/assets/docker/imgs/harbor-home.png)

