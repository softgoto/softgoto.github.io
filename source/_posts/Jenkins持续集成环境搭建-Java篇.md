---
title: Jenkins持续集成环境搭建-Java篇
date: 2020-05-12 17:58:31
categories: 技术类
tags:
- Jenkins
- CI
---

## 前言

持续集成（Continuous integration）是一种软件开发实践，即团队开发成员经常集成它们的工作，通过每个成员每天至少集成一次，也就意味着每天可能会发生多次集成。每次集成都通过自动化的构建（包括编译，发布 ，自动化测试）来验证，从而尽早地发现集成错误。

这一篇介绍一下如何使用 Jenkins 来完成 Java 项目的持续集成。

macOS Sierra 10.12.1
版本控制：SVN
Jenkins 2.34
Apache Maven 3.3.9
JDK 1.7
Apache Tomcat 7

在开始之前先说一下项目情况：
Java 后台代码通过 Maven 管理库，其中需要依赖另外一个通过 Maven 管理的 Java 工程， Web 前端使用 Vue 框架和后台代码独立存放，在部署时需要将 Web 前端文件复制到后台代码的 WAR 包中。

<!--more-->


## 第一步 创建项目

因为项目依赖Maven管理，所以这里需要创建一个Maven项目

![](https://i.loli.net/2020/06/15/146YamhsEI7bdok.png)

## 第二步 Global Tool Configuration

配置 Jenkins 的 Maven 和 JDK 环境：Jenkins -> 系统管理 -> Global Tool Configuration

因为用的是系统自带的 JDK ，所以地址为 `/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home`

![](https://i.loli.net/2020/06/15/TWz7RsOCSHI8p9f.png)

如果是另外安装的 JDK ，可以打开设置，点击左下角的 Java

![](https://i.loli.net/2020/06/15/rYt9h2MUvRC3pq8.png)

![](https://i.loli.net/2020/06/15/sZ1RYXhlVLCaW72.png)

![](https://i.loli.net/2020/06/15/LGfOb46kd9nIPey.png)

找到 Java 的路径，`MAVEN_HOME` 为 Maven 安装路径

![](https://i.loli.net/2020/06/15/imbPHDUqFat7MKA.png)

## 第三步 源码管理

![](https://i.loli.net/2020/06/15/97mDtBPvX4oWNy6.png)

## 第四步 Build

`Root POM` 这里需要指定 `pom.xml` 文件的路径，需要根据实际情况来调整，`${WORKSPACE}/xxxx/pom.xml`。以当前 Demo 为例，`/Users/You name/.jenkins/workspace` 目录结构如下

```
Demo-Server
    |---A                    //后台工程
        |---src              //后台代码
        |---pom.xml          //Maven pom文件
        |---... ...
    |---A-View               //Web前端代码
        |---... ...
```

所以我这里 `Root POM` 就填 `A/pom.xml`

![](https://i.loli.net/2020/06/15/lbBxsweiuvEjWn7.png)

到这里就完了，点击 `Apply` `保存`，然后点击 `立即构建`

![](https://i.loli.net/2020/06/15/BK75EgXeFHfajlS.png)

如果只是单纯的通过 Maven 管理的项目，不和其他项目依赖，理论上到这里就结束了，Jenkins 自动编译打包完成，拿到 WAR 包手动部署到服务器。Good job! :coffee:

## 问题

但是，我这里却编译失败，错误是找不到xxx依赖工程，前面已经交代过项目情况了，对于我这个项目，还有三件事没有完成：

1、有通过 Maven 依赖另外一个项目
2、Web 前端代码需要手动编译，并且需要复制到后台代码的 WAR 包中
3、自动部署

OK，那一个一个来解决。

### 1、添加依赖项目
首先依赖的问题，我的方案是通过 Jenkins 来管理依赖项目，先 Build 依赖项目，再 Build Demo-Server。操作和上面一样，先创建项目，然后管理源码，配置 POM 文件，最后 Build ，搞定。依赖项目 Build 成功后，会将项目打包成 jar 包放到 Maven 本地库中。在依赖项目编译成功后，再去编译 Demo-Server 项目，就 OK 了。

### 2、Web 前端代码编译，移动
前端使用 Vue 框架，编译需要使用 npm 命令。关于 Node.js 和 npm 请自行 Google。前面编译 Demo-Server 已经生成了一个 WAR 包，现在需要先解压 WAR，然后将编译好的前端代码 copy 进去，最后压缩生成新的 WAR。我们打开 Demo-Server 的配置页，在 Build 下面 `Add post-build step` 选择 `Execute shell`。记得上面需要勾选 `Run only if build succeeds`，因为只有当项目Build成功后才能去执行下一步操作

![](https://i.loli.net/2020/06/15/45hl86aGNEWAgRj.png)

`Command` 中填写 shell 脚本代码

```shell
#!/bin/bash
echo "**************************************"
echo "softgoto"
echo "----------------"
echo "描述："
echo "1、解压war包"
echo "2、编译前端Vue代码"
echo "3、将前端编译成功的代码复制到后台代码目录"
echo "4、将后台和前端代码整合打包war"
echo "**************************************"

WAR_PATH="${WORKSPACE}/A/target"
A_VIEW_PATH="${WORKSPACE}/A-view"

#解压
cd ${WAR_PATH}
unzip A.war -d A_jenkins_temp/
rm -rf A.war

#编译前端代码
cd ${A_VIEW_PATH}

#第一次需要执行npm i安装相关依赖
#npm i
npm run build

#移动前端代码
cp -r ${A_VIEW_PATH}/dist/* ${WAR_PATH}/A_jenkins_temp/
cd ${WAR_PATH}/A_jenkins_temp/
zip -r A.war ./
mv A.war ../
cd ${WAR_PATH}
rm -rf A_jenkins_temp/

```

> 脚本经供参考，实际业务需要根据项目实际情况来修改脚本


### 3、自动部署
自动部署需要安装插件Deploy

> 插件安装：Jenkins -> 系统管理 -> 管理插件 -> 可选插件，右上角过滤，输入Deploy，勾选 Deploy to Websphere container Plugin 安装并重启

插件安装成功后，选择 Demo-Server -> 配置 -> 构建后操作 -> 增加构建后操作步骤 选择 `Deploy war/ear to a container`

![](https://i.loli.net/2020/06/15/VudqUpehlxa2wY8.png)

保存，构建，完成

注意事项：
1、上图 WAR 包文件地址请填写正确
2、Tomcat 下 webapps 目录下的 manager、host-manager、ROOT 文件夹不要删除，如果删除会导致 Deploy 插件不能上传 WAR 到 Tomcat 中
3、Tomcat 管理员账号配置在 conf 目录下的 `tomcat-users.xml` 文件，配置方式如下
```xml
<role rolename="tomcat"/>
<role rolename="role1"/>
<role rolename="manager-script"/>
<role rolename="manager-gui"/>
<role rolename="manager-status"/>
<role rolename="admin-gui"/>
<role rolename="admin-script"/>
<user username="tomcat" password="tomcat" roles="manager-gui,manager-script,tomcat,admin-gui,admin-script"/>
```
4、Deploy 报内存问题，需要修改 JVM 内存设置，Tomcat 的 bin 下 `catalina.sh`，在 `cygwin=false` 上一行添加下面的代码
```shell
export JAVA_OPTS="-Xms512m -Xmx1024m -XX:PermSize=128m -XX:MaxPermSize=256m"
```


Done. :coffee: