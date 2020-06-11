---
title: 使用yum安装Mysql5.7
date: 2017-04-08 18:28:39
categories: 技术类
tags:
- MySQL
---

记一下使用 yum 安装 mysql 的过程，环境：CentOS 7.2、MySQL5.7

### 配置 yum 源

在 MySQL 官网中下载 yum 源 RPM 安装包：[https://dev.mysql.com/downloads/repo/yum/](https://dev.mysql.com/downloads/repo/yum/)

<!--more-->

![006tKfTcgy1fee6zaqdyjj317z0b3tbh.jpg](https://i.loli.net/2020/06/11/qxSvHGLRf48hr1Y.jpg)

将 MySQL RPM 源的安装包下载到服务器

```shell
$ wget http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
```

安装 MySQL 源

```shell
$ yum localinstall mysql57-community-release-el7-8.noarch.rpm
```

检查一下 MySQL 源有没有安装成功

```shell
yum repolist enabled | grep "mysql.*-community.*"
```

看到图这样就表示安装成功了

![006tKfTcgy1fee6zwdrbwj30ir02n3z6.jpg](https://i.loli.net/2020/06/11/ctJ3iH5Cfu71ROF.jpg)

当然也可以选择 MySQL 版本来安装，编辑一下 MySQL RPM 源文件

```shell
$ vim /etc/yum.repos.d/mysql-community.repo
```

比如要安装 5.6 版本，将 5.7 源的 `enabled=1` 改成 `enabled=0`，然后再将 5.6 源的 `enabled=0` 改成 `enabled=1` 就可以了，改完之后的效果如下所示： 

![006tKfTcgy1fee709xjjtj31kw0qt45k.jpg](https://i.loli.net/2020/06/11/Y8v3hlPFGe2zXuK.jpg)

### 安装 MySQL

```shell
$ yum install mysql-community-server
```

### 启动 MySQL 服务

```shell
$ systemctl start mysqld
```

查看 MySQL 的启动状态

```shell
$ systemctl status mysqld
```

### 配置开机启动

```shell
$ systemctl enable mysqld
$ systemctl daemon-reload
```

### MySQL 配置

安装完成之后，在 `/var/log/mysqld.log` 文件中给 MySQL root 用户生成了一个默认密码，通过下面的方式找到 root 默认密码，然后登录 MySQL 就能修改 root 密码了。

```shell
$ grep 'temporary password' /var/log/mysqld.log
```

![006tKfTcgy1fee70irrwvj30n701k0t4.jpg](https://i.loli.net/2020/06/11/Cx6udDtUyn8Eiao.jpg)

登录 MySQL

```shell
$ mysql -uroot -p
```

修改 root 用户密码

```shell
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '密码';
# 或者
mysql> set password for 'root'@'localhost'=password('密码');
```

> MySQL 5.7 默认安装了密码安全检查插件（validate_password），默认密码检查策略要求密码必须包含：大小写字母、数字和特殊符号，并且长度不能少于 8 位。否则会提示 ERROR 1819 (HY000): Your password does not satisfy the current policy requirements 错误，如下图所示： 

![006tKfTcgy1fee711wtn4j30ig01h3yp.jpg](https://i.loli.net/2020/06/11/tynpgrMWEFechQ8.jpg)

我们可以通过 MySQL 环境变量查看密码策略的相关信息：

```shell
mysql> show variables like '%password%';
```

![006tKfTcgy1fee71avgi3j30dg0a8ta6.jpg](https://i.loli.net/2020/06/11/SEeUYlRjTc1wxZo.jpg)

validate_password_policy：密码策略，默认为MEDIUM策略 
validate_password_dictionary_file：密码策略文件，策略为STRONG才需要 
validate_password_length：密码最少长度 
validate_password_mixed_case_count：大小写字符长度，至少1个 
validate_password_number_count ：数字至少1个 
validate_password_special_char_count：特殊字符至少1个 

上述参数是默认策略 MEDIUM 的密码检查规则，共有以下几种密码策略：

|     策略     |     检查规则     |
|:-----------:|:---------------:|
| 0 or LOW    | Length          |
| 1 or MEDIUM | Length; numeric, lowercase/uppercase, and special characters|
| 2 or STRONG | Length; numeric, lowercase/uppercase, and special characters; dictionary file|

这里是 MySQL 官方对密码策略的详细说明：[点我](http://dev.mysql.com/doc/refman/5.7/en/validate-password-options-variables.html#sysvar_validate_password_policy)

如果需要修改密码安全策略，需要在 `/etc/my.cnf` 文件添加 `validate_password_policy` 配置，配置分三种，0（LOW）、1（MEDIUM）、2（STRONG），如果选择 2 需要提供密码字典文件。如果不需要密码策略可以把 `validate_password` 配置成 `off`，当然我不建议这样做 :underage:。配置好后需要重启 MySQL 服务使配置生效。

```shell
systemctl restart mysqld
```

### 添加远程登录用户

默认只允许 root 帐户在本地登录，如果要在其它机器上连接 MySQL，必须修改 root 允许远程连接，或者添加一个允许远程连接的帐户，为了安全起见，我添加一个新的帐户：

```shell
mysql> GRANT ALL PRIVILEGES ON *.* TO 'newuser'@'%' IDENTIFIED BY 'newuserpassword' WITH GRANT OPTION;
```

### 修改字符集

将 MySQL 字符集改为 `utf8` 或者 `utf8mb4`，需要修改 `/etc/my.cnf` 配置文件，在 `[mysqld]` 下添加编码配置

```
[mysqld]
character_set_server=utf8
init_connect='SET NAMES utf8'
```

重新启动 MySQL 服务，检查一下数据库默认编码有没有修改成功

```shell
mysql> show variables like '%character%';
```

![006tKfTcgy1fee71fclrzj30dr07it9n.jpg](https://i.loli.net/2020/06/11/ucBKRAdpUGmen5N.jpg)

### 修改用户密码过期规则

默认情况下密码超过 90 天就会提示要更换，当然我们可以设置密码过期规则

```shell
mysql> ALTER USER 'root'@'localhost' PASSWORD EXPIRE NEVER;

mysql> ALTER USER 'root'@'localhost' PASSWORD EXPIRE INTERVAL 30 DAY;
```


Done. :coffee: