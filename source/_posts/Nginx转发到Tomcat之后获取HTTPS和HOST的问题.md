---
title: Nginx转发到Tomcat之后获取HTTPS和HOST的问题
date: 2017-06-26 14:32:00
categories: 技术类
tags:
- Tomcat
- HTTPS
---

问题是在 Nginx 配置完 SSL 证书之后，跑在 Tomcat 中的应用不能获取前端用户真实的 IP 和 HTTPS 头，我们需要在 Nginx 和 Tomcat 都加上以下配置。

### Nginx配置

```
proxy_set_header            X-Real-IP           $remote_addr;
proxy_set_header            REMOTE-HOST         $remote_addr;
proxy_set_header            Host                $host;
proxy_set_header            X-Forwarded-For     $proxy_add_x_forwarded_for;
proxy_set_header            X-Forwarded-Proto   $scheme;
proxy_pass_request_headers                      on;
```

<!--more-->

### Tomcat配置

在 `conf/server.xml` 文件中 `Engine` 节点下增加如下配置：

```xml
<Valve
   className="org.apache.catalina.valves.RemoteIpValve"
   internalProxies="10\.0\.1\.137"
   remoteIpHeader="X-Forwarded-For"
   proxiesHeader="X-Forwarded-By"
   protocolHeader="X-Forwarded-Proto"
   />
```

> Tomcat 7.0 之后的版本默认在规则匹配上只校验内网 IP ，如果 Nginx 转发到 Tomcat 是走外网方式，则需要增加 `internalProxies=".*"` 属性，具体参考：[Class RemoteIpValve](http://tomcat.apache.org/tomcat-7.0-doc/api/org/apache/catalina/valves/RemoteIpValve.html)



Done. :coffee: