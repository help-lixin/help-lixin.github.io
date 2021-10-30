---
layout: post
title: 'CAS源码下载以及工程目录介绍(三)' 
date: 2021-10-26
author: 李新
tags:  CAS
---

### (1). 概述
在上一节,我们通过脚手架(cas-overlay-template),搭建起了Cas Server,在这一篇,开始会对Cas源码目录进行介绍

### (2). Cas源码下载
```
# 1. 进入工作目录
lixin@lixin ~ % cd ~/GitWorkspace

# 2. 源码下载
lixin@lixin GitWorkspace % git clone https://github.com/help-lixin/cas.git

# 3. 经历漫长的编译
lixin@lixin cas % ./gradlew clean build -x test   
```
### (3). Cas源码目结构详解
+ api工程主要用于提供接口能力.  
+ core工程除了接口的定义,还有一部份是对api的实现.   
+ support工程提供一些扩展能力出来(比如:验证码).            
+ webapp工程和web相关了(从中能看出来,cas心中的目标不仅限于web).     

```
lixin@lixin GitWorkspace % cd cas
infinova@lixin cas % tree -L 2
.
├── api
│   ├── build
│   ├── build.gradle
│   ├── cas-server-core-api
│   ├── cas-server-core-api-audit
│   ├── cas-server-core-api-authentication
│   ├── cas-server-core-api-configuration
│   ├── cas-server-core-api-configuration-model
│   ├── cas-server-core-api-cookie
│   ├── cas-server-core-api-events
│   ├── cas-server-core-api-logout
│   ├── cas-server-core-api-mfa
│   ├── cas-server-core-api-monitor
│   ├── cas-server-core-api-protocol
│   ├── cas-server-core-api-services
│   ├── cas-server-core-api-throttle
│   ├── cas-server-core-api-ticket
│   ├── cas-server-core-api-util
│   ├── cas-server-core-api-validation
│   ├── cas-server-core-api-web
│   └── cas-server-core-api-webflow
├── core
│   ├── build
│   ├── build.gradle
│   ├── cas-server-core
│   ├── cas-server-core-audit
│   ├── cas-server-core-audit-api
│   ├── cas-server-core-authentication
│   ├── cas-server-core-authentication-api
│   ├── cas-server-core-authentication-attributes
│   ├── cas-server-core-authentication-mfa
│   ├── cas-server-core-authentication-mfa-api
│   ├── cas-server-core-authentication-throttle
│   ├── cas-server-core-configuration
│   ├── cas-server-core-configuration-api
│   ├── cas-server-core-configuration-metadata-repository
│   ├── cas-server-core-cookie
│   ├── cas-server-core-cookie-api
│   ├── cas-server-core-events
│   ├── cas-server-core-events-api
│   ├── cas-server-core-events-configuration
│   ├── cas-server-core-events-configuration-cloud-bus
│   ├── cas-server-core-logging
│   ├── cas-server-core-logging-api
│   ├── cas-server-core-logging-config
│   ├── cas-server-core-logout
│   ├── cas-server-core-logout-api
│   ├── cas-server-core-monitor
│   ├── cas-server-core-notifications
│   ├── cas-server-core-rest
│   ├── cas-server-core-services
│   ├── cas-server-core-services-api
│   ├── cas-server-core-services-authentication
│   ├── cas-server-core-services-registry
│   ├── cas-server-core-tickets
│   ├── cas-server-core-tickets-api
│   ├── cas-server-core-util
│   ├── cas-server-core-util-api
│   ├── cas-server-core-validation
│   ├── cas-server-core-validation-api
│   ├── cas-server-core-web
│   ├── cas-server-core-web-api
│   ├── cas-server-core-webflow
│   ├── cas-server-core-webflow-api
│   ├── cas-server-core-webflow-mfa
│   └── cas-server-core-webflow-mfa-api
├── support
│   ├── build
│   ├── build.gradle
│   ├── cas-server-support-acceptto-mfa
│   ├── cas-server-support-account-mgmt
│   ├── cas-server-support-account-mgmt-core
│   ├── cas-server-support-acme
│   ├── cas-server-support-actions
│   ├── cas-server-support-audit-couchbase
│   ├── cas-server-support-audit-couchdb
│   ├── cas-server-support-audit-dynamodb
│   ├── cas-server-support-audit-jdbc
│   ├── cas-server-support-audit-mongo
│   ├── cas-server-support-audit-redis
│   ├── cas-server-support-audit-rest
│   ├── cas-server-support-aup-core
│   ├── cas-server-support-aup-couchbase
│   ├── cas-server-support-aup-couchdb
│   ├── cas-server-support-aup-jdbc
│   ├── cas-server-support-aup-ldap
│   ├── cas-server-support-aup-mongo
│   ├── cas-server-support-aup-redis
│   ├── cas-server-support-aup-rest
│   ├── cas-server-support-aup-webflow
│   ├── cas-server-support-authy
│   ├── cas-server-support-authy-core
│   ├── cas-server-support-aws
│   ├── cas-server-support-aws-cognito-authentication
│   ├── cas-server-support-aws-s3-service-registry
│   ├── cas-server-support-azure-core
│   ├── cas-server-support-azuread-authentication
│   ├── cas-server-support-basic
│   ├── cas-server-support-bom
│   ├── cas-server-support-bootadmin-client
│   ├── cas-server-support-captcha
│   ├── cas-server-support-captcha-core
│   ├── cas-server-support-cassandra-authentication
│   ├── cas-server-support-cassandra-core
│   ├── cas-server-support-cassandra-service-registry
│   ├── cas-server-support-cassandra-ticket-registry
│   ├── cas-server-support-cloud-directory-authentication
│   ├── cas-server-support-configuration
│   ├── cas-server-support-configuration-cloud-amqp
│   ├── cas-server-support-configuration-cloud-aws-s3
│   ├── cas-server-support-configuration-cloud-aws-secretsmanager
│   ├── cas-server-support-configuration-cloud-aws-ssm
│   ├── cas-server-support-configuration-cloud-azure-keyvault
│   ├── cas-server-support-configuration-cloud-dynamodb
│   ├── cas-server-support-configuration-cloud-jdbc
│   ├── cas-server-support-configuration-cloud-kafka
│   ├── cas-server-support-configuration-cloud-mongo
│   ├── cas-server-support-configuration-cloud-rest
│   ├── cas-server-support-configuration-cloud-vault
│   ├── cas-server-support-configuration-cloud-zookeeper
│   ├── cas-server-support-consent-api
│   ├── cas-server-support-consent-core
│   ├── cas-server-support-consent-couchdb
│   ├── cas-server-support-consent-jdbc
│   ├── cas-server-support-consent-ldap
│   ├── cas-server-support-consent-mongo
│   ├── cas-server-support-consent-redis
│   ├── cas-server-support-consent-rest
│   ├── cas-server-support-consent-webflow
│   ├── cas-server-support-consul-client
│   ├── cas-server-support-cosmosdb-core
│   ├── cas-server-support-cosmosdb-service-registry
│   ├── cas-server-support-couchbase-authentication
│   ├── cas-server-support-couchbase-core
│   ├── cas-server-support-couchbase-service-registry
│   ├── cas-server-support-couchbase-ticket-registry
│   ├── cas-server-support-couchdb-authentication
│   ├── cas-server-support-couchdb-core
│   ├── cas-server-support-couchdb-service-registry
│   ├── cas-server-support-couchdb-ticket-registry
│   ├── cas-server-support-digest-authentication
│   ├── cas-server-support-discovery-profile
│   ├── cas-server-support-duo
│   ├── cas-server-support-duo-core
│   ├── cas-server-support-duo-core-mfa
│   ├── cas-server-support-dynamodb-core
│   ├── cas-server-support-dynamodb-service-registry
│   ├── cas-server-support-dynamodb-ticket-registry
│   ├── cas-server-support-ehcache-monitor
│   ├── cas-server-support-ehcache-ticket-registry
│   ├── cas-server-support-ehcache3-ticket-registry
│   ├── cas-server-support-electrofence
│   ├── cas-server-support-eureka-client
│   ├── cas-server-support-events-couchdb
│   ├── cas-server-support-events-dynamodb
│   ├── cas-server-support-events-influxdb
│   ├── cas-server-support-events-jpa
│   ├── cas-server-support-events-memory
│   ├── cas-server-support-events-mongo
│   ├── cas-server-support-events-redis
│   ├── cas-server-support-fortress
│   ├── cas-server-support-gauth
│   ├── cas-server-support-gauth-core
│   ├── cas-server-support-gauth-core-mfa
│   ├── cas-server-support-gauth-couchdb
│   ├── cas-server-support-gauth-dynamodb
│   ├── cas-server-support-gauth-jpa
│   ├── cas-server-support-gauth-ldap
│   ├── cas-server-support-gauth-mongo
│   ├── cas-server-support-gauth-redis
│   ├── cas-server-support-generic
│   ├── cas-server-support-generic-remote-webflow
│   ├── cas-server-support-geolocation
│   ├── cas-server-support-geolocation-googlemaps
│   ├── cas-server-support-geolocation-maxmind
│   ├── cas-server-support-git-core
│   ├── cas-server-support-git-service-registry
│   ├── cas-server-support-google-analytics
│   ├── cas-server-support-grouper
│   ├── cas-server-support-grouper-core
│   ├── cas-server-support-gua
│   ├── cas-server-support-hazelcast
│   ├── cas-server-support-hazelcast-core
│   ├── cas-server-support-hazelcast-discovery-aws
│   ├── cas-server-support-hazelcast-discovery-azure
│   ├── cas-server-support-hazelcast-discovery-jclouds
│   ├── cas-server-support-hazelcast-discovery-kubernetes
│   ├── cas-server-support-hazelcast-discovery-swarm
│   ├── cas-server-support-hazelcast-discovery-zookeeper
│   ├── cas-server-support-hazelcast-monitor
│   ├── cas-server-support-hazelcast-ticket-registry
│   ├── cas-server-support-ignite-ticket-registry
│   ├── cas-server-support-infinispan-ticket-registry
│   ├── cas-server-support-influxdb-core
│   ├── cas-server-support-interrupt-api
│   ├── cas-server-support-interrupt-core
│   ├── cas-server-support-interrupt-webflow
│   ├── cas-server-support-inwebo-mfa
│   ├── cas-server-support-javamelody
│   ├── cas-server-support-jdbc
│   ├── cas-server-support-jdbc-authentication
│   ├── cas-server-support-jdbc-drivers
│   ├── cas-server-support-jdbc-monitor
│   ├── cas-server-support-jms-ticket-registry
│   ├── cas-server-support-jmx
│   ├── cas-server-support-jpa-eclipselink
│   ├── cas-server-support-jpa-hibernate
│   ├── cas-server-support-jpa-service-registry
│   ├── cas-server-support-jpa-ticket-registry
│   ├── cas-server-support-jpa-util
│   ├── cas-server-support-json-service-registry
│   ├── cas-server-support-kafka-core
│   ├── cas-server-support-ldap
│   ├── cas-server-support-ldap-core
│   ├── cas-server-support-ldap-monitor
│   ├── cas-server-support-ldap-service-registry
│   ├── cas-server-support-logback
│   ├── cas-server-support-logging-config-cloudwatch
│   ├── cas-server-support-logging-config-splunk
│   ├── cas-server-support-logging-config-sqs
│   ├── cas-server-support-memcached-aws-elasticache
│   ├── cas-server-support-memcached-core
│   ├── cas-server-support-memcached-monitor
│   ├── cas-server-support-memcached-spy
│   ├── cas-server-support-memcached-ticket-registry
│   ├── cas-server-support-metrics
│   ├── cas-server-support-mongo
│   ├── cas-server-support-mongo-core
│   ├── cas-server-support-mongo-monitor
│   ├── cas-server-support-mongo-service-registry
│   ├── cas-server-support-mongo-ticket-registry
│   ├── cas-server-support-notifications-fcm
│   ├── cas-server-support-oauth
│   ├── cas-server-support-oauth-api
│   ├── cas-server-support-oauth-core
│   ├── cas-server-support-oauth-core-api
│   ├── cas-server-support-oauth-services
│   ├── cas-server-support-oauth-uma
│   ├── cas-server-support-oauth-uma-core
│   ├── cas-server-support-oauth-uma-jpa
│   ├── cas-server-support-oauth-webflow
│   ├── cas-server-support-oidc
│   ├── cas-server-support-oidc-core
│   ├── cas-server-support-oidc-core-api
│   ├── cas-server-support-oidc-services
│   ├── cas-server-support-okta-authentication
│   ├── cas-server-support-openid
│   ├── cas-server-support-openid-webflow
│   ├── cas-server-support-otp-mfa
│   ├── cas-server-support-otp-mfa-core
│   ├── cas-server-support-pac4j
│   ├── cas-server-support-pac4j-api
│   ├── cas-server-support-pac4j-authentication
│   ├── cas-server-support-pac4j-core
│   ├── cas-server-support-pac4j-core-clients
│   ├── cas-server-support-pac4j-webflow
│   ├── cas-server-support-passwordless
│   ├── cas-server-support-passwordless-jpa
│   ├── cas-server-support-passwordless-ldap
│   ├── cas-server-support-passwordless-mongo
│   ├── cas-server-support-passwordless-webflow
│   ├── cas-server-support-person-directory
│   ├── cas-server-support-person-directory-core
│   ├── cas-server-support-pm
│   ├── cas-server-support-pm-core
│   ├── cas-server-support-pm-jdbc
│   ├── cas-server-support-pm-ldap
│   ├── cas-server-support-pm-rest
│   ├── cas-server-support-pm-webflow
│   ├── cas-server-support-qr-authentication
│   ├── cas-server-support-radius
│   ├── cas-server-support-radius-core
│   ├── cas-server-support-radius-core-mfa
│   ├── cas-server-support-radius-mfa
│   ├── cas-server-support-redis-authentication
│   ├── cas-server-support-redis-core
│   ├── cas-server-support-redis-service-registry
│   ├── cas-server-support-redis-ticket-registry
│   ├── cas-server-support-reports
│   ├── cas-server-support-reports-core
│   ├── cas-server-support-rest
│   ├── cas-server-support-rest-authentication
│   ├── cas-server-support-rest-core
│   ├── cas-server-support-rest-service-registry
│   ├── cas-server-support-rest-services
│   ├── cas-server-support-rest-tokens
│   ├── cas-server-support-rest-x509
│   ├── cas-server-support-saml
│   ├── cas-server-support-saml-core
│   ├── cas-server-support-saml-core-api
│   ├── cas-server-support-saml-googleapps
│   ├── cas-server-support-saml-googleapps-core
│   ├── cas-server-support-saml-idp
│   ├── cas-server-support-saml-idp-core
│   ├── cas-server-support-saml-idp-discovery
│   ├── cas-server-support-saml-idp-metadata
│   ├── cas-server-support-saml-idp-metadata-aws-s3
│   ├── cas-server-support-saml-idp-metadata-couchdb
│   ├── cas-server-support-saml-idp-metadata-git
│   ├── cas-server-support-saml-idp-metadata-jpa
│   ├── cas-server-support-saml-idp-metadata-mongo
│   ├── cas-server-support-saml-idp-metadata-redis
│   ├── cas-server-support-saml-idp-metadata-rest
│   ├── cas-server-support-saml-idp-ticket
│   ├── cas-server-support-saml-idp-web
│   ├── cas-server-support-saml-mdui
│   ├── cas-server-support-saml-mdui-core
│   ├── cas-server-support-saml-sp-integrations
│   ├── cas-server-support-scim
│   ├── cas-server-support-script-engines
│   ├── cas-server-support-sentry
│   ├── cas-server-support-service-registry-stream
│   ├── cas-server-support-service-registry-stream-hazelcast
│   ├── cas-server-support-service-registry-stream-kafka
│   ├── cas-server-support-session-hazelcast
│   ├── cas-server-support-session-jdbc
│   ├── cas-server-support-session-mongo
│   ├── cas-server-support-session-redis
│   ├── cas-server-support-shell
│   ├── cas-server-support-shibboleth
│   ├── cas-server-support-shiro-authentication
│   ├── cas-server-support-simple-mfa
│   ├── cas-server-support-simple-mfa-core
│   ├── cas-server-support-sleuth
│   ├── cas-server-support-sms-aws-sns
│   ├── cas-server-support-sms-clickatell
│   ├── cas-server-support-sms-nexmo
│   ├── cas-server-support-sms-smsmode
│   ├── cas-server-support-sms-textmagic
│   ├── cas-server-support-sms-twilio
│   ├── cas-server-support-soap-authentication
│   ├── cas-server-support-spnego
│   ├── cas-server-support-spnego-webflow
│   ├── cas-server-support-surrogate-api
│   ├── cas-server-support-surrogate-authentication
│   ├── cas-server-support-surrogate-authentication-couchdb
│   ├── cas-server-support-surrogate-authentication-jdbc
│   ├── cas-server-support-surrogate-authentication-ldap
│   ├── cas-server-support-surrogate-authentication-rest
│   ├── cas-server-support-surrogate-webflow
│   ├── cas-server-support-swagger
│   ├── cas-server-support-swivel
│   ├── cas-server-support-swivel-core
│   ├── cas-server-support-syncope-authentication
│   ├── cas-server-support-themes
│   ├── cas-server-support-themes-collection
│   ├── cas-server-support-themes-core
│   ├── cas-server-support-throttle
│   ├── cas-server-support-throttle-bucket4j
│   ├── cas-server-support-throttle-core
│   ├── cas-server-support-throttle-couchdb
│   ├── cas-server-support-throttle-hazelcast
│   ├── cas-server-support-throttle-jdbc
│   ├── cas-server-support-throttle-mongo
│   ├── cas-server-support-throttle-redis
│   ├── cas-server-support-thymeleaf
│   ├── cas-server-support-token-authentication
│   ├── cas-server-support-token-core
│   ├── cas-server-support-token-core-api
│   ├── cas-server-support-token-tickets
│   ├── cas-server-support-token-webflow
│   ├── cas-server-support-trusted
│   ├── cas-server-support-trusted-mfa
│   ├── cas-server-support-trusted-mfa-core
│   ├── cas-server-support-trusted-mfa-couchdb
│   ├── cas-server-support-trusted-mfa-dynamodb
│   ├── cas-server-support-trusted-mfa-jdbc
│   ├── cas-server-support-trusted-mfa-mongo
│   ├── cas-server-support-trusted-mfa-redis
│   ├── cas-server-support-trusted-mfa-rest
│   ├── cas-server-support-trusted-webflow
│   ├── cas-server-support-u2f
│   ├── cas-server-support-u2f-core
│   ├── cas-server-support-u2f-couchdb
│   ├── cas-server-support-u2f-dynamodb
│   ├── cas-server-support-u2f-jpa
│   ├── cas-server-support-u2f-mongo
│   ├── cas-server-support-u2f-redis
│   ├── cas-server-support-validation
│   ├── cas-server-support-validation-core
│   ├── cas-server-support-webauthn
│   ├── cas-server-support-webauthn-core
│   ├── cas-server-support-webauthn-core-webflow
│   ├── cas-server-support-webauthn-dynamodb
│   ├── cas-server-support-webauthn-jpa
│   ├── cas-server-support-webauthn-ldap
│   ├── cas-server-support-webauthn-mongo
│   ├── cas-server-support-webauthn-redis
│   ├── cas-server-support-webauthn-rest
│   ├── cas-server-support-websockets
│   ├── cas-server-support-ws-idp
│   ├── cas-server-support-ws-idp-api
│   ├── cas-server-support-ws-sts
│   ├── cas-server-support-ws-sts-api
│   ├── cas-server-support-wsfederation
│   ├── cas-server-support-wsfederation-webflow
│   ├── cas-server-support-x509
│   ├── cas-server-support-x509-core
│   ├── cas-server-support-x509-webflow
│   ├── cas-server-support-yaml-service-registry
│   ├── cas-server-support-yubikey
│   ├── cas-server-support-yubikey-core
│   ├── cas-server-support-yubikey-core-mfa
│   ├── cas-server-support-yubikey-couchdb
│   ├── cas-server-support-yubikey-dynamodb
│   ├── cas-server-support-yubikey-jpa
│   ├── cas-server-support-yubikey-mongo
│   └── cas-server-support-yubikey-redis
└── webapp
    ├── build
    ├── build.gradle
    ├── cas-server-webapp
    ├── cas-server-webapp-bootadmin-server
    ├── cas-server-webapp-config
    ├── cas-server-webapp-config-server
    ├── cas-server-webapp-eureka-server
    ├── cas-server-webapp-init
    ├── cas-server-webapp-init-bootadmin-server
    ├── cas-server-webapp-init-config-server
    ├── cas-server-webapp-init-eureka-server
    ├── cas-server-webapp-init-tomcat
    ├── cas-server-webapp-jetty
    ├── cas-server-webapp-resources
    ├── cas-server-webapp-starter-tomcat
    ├── cas-server-webapp-tomcat
    └── cas-server-webapp-undertow
```
### (4). 总结
CAS在工程分层结构上,还是比较明细的,虽然,工程有点多(代表着生态比较强大的),会让你不知道该如何入手,在后面的章节,我会深入剖析应该如何入手.  