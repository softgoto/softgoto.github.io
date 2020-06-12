---
title: MySQL数据定时备份
date: 2017-03-30 14:40:08
categories: 技术类
tags:
- MySQL
---

这里用到的 MySQL 服务是我们在云服务器上自己搭建的，在跑备份脚本之前我们先定好数据备份目录和脚本日志目录

```shell
$ cd /mnt
$ mkdir backup
$ cd backup
$ mkdir db
$ mkdir log
```

下面是备份脚本

<!--more-->

```shell
#!/bin/bash

# 数据库的账号密码
dbuser='username'
dbpasswd='********'

#可以定义多个数据库,中间以空格隔开,如:test test1 test2 
dbnames='test test1 test2'
backtime=`date +%Y%m%d%H%M`

#记录日志路径
logpath='/mnt/backup/log'

#数据备份路径
datapath='/mnt/backup/db'

#备份数据库
for dbname in $dbnames; do
source=`mysqldump -u ${dbuser} -p${dbpasswd} ${dbname} --skip-lock-tables | gzip > ${datapath}/${dbname}-${backtime}.sql.gz` 2>> ${logpath}/db_bak.log;

#备份成功
if [ "$?" == 0 ];then
echo "Database ${dbname} backup success:${dbname}-${backtime}.sql.gz" >> ${logpath}/db_bak.log
else
#备份失败
echo "Database ${dbname} backup failure:${dbname}-${backtime}.sql.gz" >> ${logpath}/db_bak.log
fi

done
```

给脚本文件赋权

```shell
$ chmod u+x dbbackup.sh

# 测试脚本是否能正常运行
$ ./bkDatabaseName.sh
```

添加到系统计划任务

```shell
$ sudo echo "00 0,6,12,18 * * * root /opt/task/dbbackup.sh" >> /etc/crontab
```

重启 crontab

```shell
$ sudo service crond restart
```

关于计划任务参数说明

```
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
```


Done. :coffee: