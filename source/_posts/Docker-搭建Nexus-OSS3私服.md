---
title: Docker 搭建Nexus OSS3私服
date: 2017-04-01 15:26:59
categories: 技术类
tags:
- Docker
- Nexus
---

### Docker安装

Docker 安装可以[参考这里](https://softgoto.github.io/2017/03/%E5%A6%82%E4%BD%95%E5%9C%A8CentOS-7%E4%B8%8A%E9%80%9A%E8%BF%87Docker%E7%9A%84%E6%96%B9%E5%BC%8F%E5%AE%89%E8%A3%85Gitlab%E5%92%8CJenkins/#more)

### Nexus介绍

Nexus 是个仓库管理器，目前主要分 2 大版本：2.X 和 3.X，其中 2.X 主要支持的格式是 Maven、P2、OBR、Yum。3.X 主要支持的是 Docker、NuGet、npm、Bower、Pypi、Ruby Gems，当然也支持构建工具 Maven 和 Gradle。Nexus 3 只支持 Oracle jdk8，不支持其它版本的 JDK，比如 OpenJDK。

### Nexus安装

本次安装的 Nexus OSS 版本是 3.2.1

镜像地址：[docker-nexus3](https://github.com/sonatype/docker-nexus3)

使用 Docker 安装就很简单了，三步搞定

<!--more-->

#### 下载镜像

```shell
$ docker pull sonatype/nexus3:3.2.1
$ docker images
```

#### 创建映射目录

因为在 Docker 容器中运行，所以需要将 Nexus 数据目录映射到实际磁盘上，方便后期备份和迁移

```shell
$ cd /mnt/docker
$ mkdir nexus-data && chown -R 200 nexus-data
```

#### 启动容器

```shell
$ docker run -d -p 18081:8081 -v /mnt/docker/nexus-data:/nexus-data --name nexus sonatype/nexus3
```

#### 使用

默认的账号密码

> `admin`
> 
> `admin123`



Done. ☕️