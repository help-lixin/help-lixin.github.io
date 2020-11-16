---
layout: post
title: 'Solr Tomcat配置安全管理'
date: 2018-10-01
author: 李新
tags: Solr
---

### (1). 配置安全账号(tomcat-users.xml)
```
<tomcat-users>
	<role rolename="tomcat"/>
	<user username="tomcat" password="tomcat" roles="tomcat"/>
</tomcat-users>
```
### (2). 配置web.xml
```
<security-constraint>
    <web-resource-collection>
        <web-resource-name>   Restrict access to Solr admin </web-resource-name>
        <url-pattern>/</url-pattern>
        <http-method>DELETE</http-method>
        <http-method>GET</http-method>
        <http-method>POST</http-method>
        <http-method>PUT</http-method>
    </web-resource-collection>

    <auth-constraint>
        <role-name>tomcat</role-name>
    </auth-constraint>
    <user-data-constraint>
        <transport-guarantee>NONE</transport-guarantee>
    </user-data-constraint>
</security-constraint>

<login-config>
    <auth-method>BASIC</auth-method>
    <realm-name>default</realm-name>
</login-config>

```
