---
layout: post
title: 'Docker /var/run/docker.sock'
date: 2020-11-01
author: 李新
tags: Docker
---

### (1). /var/run/docker.sock
> /var/run/docker.sock是docker的守护进程(Docker daemon),它负责与Docker容器进行通信. 

### (2). 为什么纠结这个?
> K8S的编排的对象是Docker,那么K8S是如何与Docker通信的呢?      

### (3). 通过curl来访问:docker.sock
> [API参考:https://docs.docker.com/engine/api/v1.41/#section/Versioning](https://docs.docker.com/engine/api/v1.41/#section/Versioning)    
> 通过curl可以访问,代表着docker.sock是一个HTTP协议.   
> 那么也代表着K8S也是与docker.sock进行通信的.   

```
lixin-macbook:~ lixin$ curl -v --unix-socket /var/run/docker.sock http:/v1.41/containers/json
*   Trying /var/run/docker.sock...
* Connected to v1.41 (docker.sock) port 80 (#0)
> GET /containers/json HTTP/1.1
> Host: v1.41
> User-Agent: curl/7.64.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< Api-Version: 1.41
< Content-Type: application/json
< Date: Mon, 11 Jan 2021 10:09:51 GMT
< Docker-Experimental: false
< Ostype: linux
< Server: Docker/20.10.2 (linux)
< Transfer-Encoding: chunked
< 
[{"Id":"5c00f102a174d229458d3878aa0c4b097ad555146d24c8b138e2de28f5121c54","Names":["/dockercompose_web-1_1"],"Image":"nginx:laster","ImageID":"sha256:82f8d4e7d5f150a469b12423674c5cd35d676cb55891a7d47704e7dbc9a29c1c","Command":"/bin/sh -c /entrypoint.sh","Created":1610358470,"Ports":[{"IP":"0.0.0.0","PrivatePort":80,"PublicPort":8080,"Type":"tcp"}],"Labels":{"com.docker.compose.config-hash":"ec40990c0ed945ce189e6cebbe9160a9ce40b1f329cfc2d952bb71282642c05d","com.docker.compose.container-number":"1","com.docker.compose.oneoff":"False","com.docker.compose.project":"dockercompose","com.docker.compose.project.config_files":"docker-compose.yml","com.docker.compose.project.working_dir":"/Users/lixin/DockerWorkspace/DockerCompose","com.docker.compose.service":"web-1","com.docker.compose.version":"1.27.4","org.label-schema.build-date":"20201113","org.label-schema.license":"GPLv2","org.label-schema.name":"CentOS Base Image","org.label-schema.schema-version":"1.0","org.label-schema.vendor":"CentOS","org.opencontainers.image.created":"2020-11-13 00:00:00+00:00","org.opencontainers.image.licenses":"GPL-2.0-only","org.opencontainers.image.title":"CentOS Base Image","org.opencontainers.image.vendor":"CentOS"},"State":"running","Status":"Up 22 minutes","HostConfig":{"NetworkMode":"dockercompose_default"},"NetworkSettings":{"Networks":{"dockercompose_default":{"IPAMConfig":null,"Links":null,"Aliases":null,"NetworkID":"8efac64a763846537d00e0429bb752ea2c871bee4ddd61adcada21619335197b","EndpointID":"5fa6a2cb23d9b519c0643ada02acbf7fae4e07ed7f87f9e3972485c5a62263c7","Gateway":"172.18.0.1","IPAddress":"172.18.0.3","IPPrefixLen":16,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"MacAddress":"02:42:ac:12:00:03","DriverOpts":null}}},"Mounts":[]},{"Id":"21b37c04f71c08883210eba47b64e8bdc497b7a173d5be902221bea322bb8289","Names":["/dockercompose_web-2_1"],"Image":"nginx:laster","ImageID":"sha256:82f8d4e7d5f150a469b12423674c5cd35d676cb55891a7d47704e7dbc9a29c1c","Command":"/bin/sh -c /entrypoint.sh","Created":1610358470,"Ports":[{"IP":"0.0.0.0","PrivatePort":80,"PublicPort":8081,"Type":"tcp"}],"Labels":{"com.docker.compose.config-hash":"d8ec64c2112c20892406091aaec0d0415e660515a2e303bd58c16bf816d7b8dd","com.docker.compose.container-number":"1","com.docker.compose.oneoff":"False","com.docker.compose.project":"dockercompose","com.docker.compose.project.config_files":"docker-compose.yml","com.docker.compose.project.working_dir":"/Users/lixin/DockerWorkspace/DockerCompose","com.docker.compose.service":"web-2","com.docker.compose.version":"1.27.4","org.label-schema.build-date":"20201113","org.label-schema.license":"GPLv2","org.label-schema.name":"CentOS Base Image","org.label-schema.schema-version":"1.0","org.label-schema.vendor":"CentOS","org.opencontainers.image.created":"2020-11-13 00:00:00+00:00","org.opencontainers.image.licenses":"GPL-2.0-only","org.opencontainers.image.title":"CentOS Base Image","org.opencontainers.image.vendor":"CentOS"},"State":"running","Status":"Up 22 minutes","HostConfig":{"NetworkMode":"dockercompose_default"},"NetworkSettings":{"Networks":{"dockercompose_default":{"IPAMConfig":null,"Links":null,"Aliases":null,"NetworkID":"8efac64a763846537d00e0429bb752ea2c871bee4ddd61adcada21619335197b","EndpointID":"775f1703213cabaf87ba19fa3b234aadb46718a99388c576d02da5ef6b942e44","Gateway":"172.18.0.1","IPAddress":"172.18.0.2","IPPrefixLen":16,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"MacAddress":"02:42:ac:12:00:02","DriverOpts":null}}},"Mounts":[]}]
```
### (4). 总结
> docker.sock可以通过HTTP进行访问,Docker之所以不直接开放该端口,估计是出于安全考虑.  