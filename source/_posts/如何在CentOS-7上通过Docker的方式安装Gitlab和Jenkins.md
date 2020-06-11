---
title: 如何在CentOS 7上通过Docker的方式安装Gitlab和Jenkins
date: 2017-03-30 15:35:17
categories: 技术类
tags:
- Docker
- Gitlab
- Jenkins
---

## 系统要求

示例中使用的是：CentOS 7 64-bit

检查：Red Hat 会装旧版的 Docker，包名 `docker` 会和现在的 `docker-engine` 冲突，如果装了旧版的Docker，请用下面的命令删除：

```shell
$ sudo yum -y remove docker docker-common container-selinux
```

还需要删除`docker-selinux`

```shell
$ sudo yum -y remove docker-selinux
```

`/var/lib/docker` 下的内容不会被删除，意思是说如果用旧版本的 Docker 创建的 `images`、`containers`、`volumes` 都会保留。

<!--more-->

## Docker

参考官方教程 [Get Docker for CentOS](https://docs.docker.com/engine/installation/linux/centos/)，官方推荐从 `Docker’s repositories` 来安装 Docker，方便升级和管理。

### 配置yum存储库

1、安装 `yum-utils`，它提供了 `yum-config-manager`

```shell
$ sudo yum install -y yum-utils
```

2、设置固定的存储库

```shell
$ sudo yum-config-manager \
--add-repo \
https://docs.docker.com/engine/installation/linux/repo_files/centos/docker.repo
```

3、启用测试 repository（注意：这一步可选，并且不要在生产环境或者非测试环境上用不稳定的 repository）

**Do not use unstable repositories on on production systems or for non-testing workloads.**

> **Warning**: If you have both stable and unstable repositories enabled, installing or updating without specifying a version in the yum install or yum update command will always install the highest possible version, which will almost certainly be an unstable one.

4、开启测试或禁用测试（这里开启测试之后，后面安装或者更新时可能就会更新到最新的不稳定版本，需要注意）

```shell
$ sudo yum-config-manager --enable docker-testing
$ sudo yum-config-manager --disable docker-testing
```

### 安装 Docker

1、更新 `yum` 包索引

```shell
$ sudo yum makecache fast
```

2、安装最新版的 Docker，或者是安装指定版本的。前面如果开了 `docker-testing`，那这里安装的时候没有指定版本号，有可能会安装最新的不稳定版本，如果在生产环境，建议指定最新的稳定版本的版本号来安装。

```shell
$ sudo yum -y install docker-engine
```

3、查看 list 并且选择要安装的版本

```shell
$ yum list docker-engine.x86_64 --showduplicates |sort -r
```

4、指定版本号安装

```shell
$ sudo yum -y install docker-engine-1.13.1
```

5、启动 Docker

```shell
$ sudo systemctl start docker
```

6、启动 `hello-world` 镜像验证 Docker 是否安装成功。这条命令会下载一个测试镜像并运行在容器中，当容器启动时会打印一条消息然后退出。

```shell
$ sudo docker run hello-world
```

如果报这个错：`net/http: TLS handshake timeout`，原因是因为国内无法直接连上 Docker Hub，我们可以换成国内 DaoCloud 的 Docker 镜像仓库，或者自己想办法科学上网。

### Docker 另外一种安装方式

除了通过上面的方式安装 Docker 之外，如果你使用了阿里云，还可以使用阿里云的安装脚本，一键更方便。

```shell
$ curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
```

除了安装脚本，阿里云还提供了镜像加速器 [https://cr.console.aliyun.com/#/accelerator](https://cr.console.aliyun.com/#/accelerator)。针对 Docker 客户端版本大于 `1.10` 的用户，可以通过修改 docker daemon配置文件 `/etc/docker/daemon.json` 来使用加速器。

```shell
$ sudo mkdir -p /etc/docker
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://qi2pd4ym.mirror.aliyuncs.com"]
}
EOF
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

针对 Docker 客户的版本小于等于 `1.10` 的用户或者想配置启动参数，可以使用下面的命令将配置添加到 docker daemon 的启动参数中

```shell
# 系统要求 CentOS 7 以上，Docker 1.9 以上。
$ sudo cp -n /lib/systemd/system/docker.service /etc/systemd/system/docker.service
```

```shell
# Docker 1.12 以下版本使用 docker daemon 命令
$ sudo sed -i "s|ExecStart=/usr/bin/docker daemon|ExecStart=/usr/bin/docker daemon --registry-mirror=https://qxx96o44.mirror.aliyuncs.com|g" /etc/systemd/system/docker.service
```

```shell
#Docker 1.12 及以上版本使用 dockerd 命令
$ sudo sed -i "s|ExecStart=/usr/bin/dockerd|ExecStart=/usr/bin/dockerd --registry-mirror=https://qxx96o44.mirror.aliyuncs.com|g" /etc/systemd/system/docker.service
$ sudo systemctl daemon-reload
$ sudo service docker restart
```

### 安装 Docker Compose

Docker Compose 是用于定义和运行多容器 Docker 应用程序的工具。通过 Compose，可以使用 YML 文件来配置应用程序需要的所有服务。然后使用一个命令，就可以从 YML 文件配置中创建并启动所有服务，非常方便快捷。

在安装 Docker Compose 之前，需要确保系统已经安装 pip，如果没有请参考下面的步骤：

> 如果 `yum python-pip` 报错，是因为 CentOS 这种衍生发行版在更新源的时候比较滞后，或者一些扩展源根本就没有，所以这里需要先安装扩展源 EPEL。

```shell
$ sudo yum -y install epel-release
```

安装好扩展源 EPEL 之后，就可以安装 pip 了。

```shell
$ sudo yum -y install python-pip
```

然后清一下源缓存

```shell
$ sudo yum clean all
```

接着就可以通过 pip 安装 Docker Compose 了（pip 版本必须是 6.0 或更高）

```shell
$ pip install docker-compose
```

如果要卸载 Docker Compose

```shell
$ pip uninstall docker-compose
```

## Gitlab

### 安装 Gitlab

我们直接使用大神写好的 YML 文件来安装 Gitlab，这里已经将启动 Gitlab 需要的 Redis、PostgreSQL、Gitlab 模块全部都添加进来了。我们先将文件下载到服务器

大神开源的 [docker gitlab 镜像](https://github.com/sameersbn/docker-gitlab) 在这里，记得点个 star

```shell
$ wget https://raw.githubusercontent.com/sameersbn/docker-gitlab/master/docker-compose.yml
```

文件下载后就可以直接启动了，记得要在文件所在目录执行这条命令。

```shell
$ docker-compose up -d
```

> 在执行上面命令时 docker 服务必须启动，并且在当前目录下必须有 `docker-compose.yml` 文件。`docker-compose.yml` 文件**很重要**，后期如果修改了配置，删除镜像后需要根据此文件重新生成镜像。

等几分钟后，打开浏览器访问 `http://localhost:10080` 并输入初始化用户名密码

username: **root**
password: **5iveL!fe**

### 配置 Docker Gitlab 镜像

这里讲下 Docker Gitlab 镜像的配置，直接编辑前面下载下来的 `docker-compose.yml` 文件

```shell
$ vim docker-compose.yml
```

主要说下目录映射的配置，因为一般情况下我们都会把产生的数据文件存放到宿主机目录下，因为在删除镜像时所有内容都会被删除，为了防止手误或后期的备份迁移方便，所以我们会把目录映射到宿主机上。

```yaml
# 1、redis 模块
volumes:
    - /mnt/docker/gitlab/redis:/var/lib/redis:Z # 左边 /mnt/docker/gitlab/redis 为宿主机目录，右边 /var/lib/redis 镜像目录

# 2、postgresql 模块
volumes:
    - /mnt/docker/gitlab/postgresql:/var/lib/postgresql:Z # 同上，左边是宿主机目录，右边是镜像目录

# 3、gitlab 模块
volumes:
    - /mnt/docker/gitlab/gitlab:/home/git/data:Z # 同上，左边是宿主机目录，右边是镜像目录

    - GITLAB_HOST=本机IP # 默认的配置是localhost，如果不改，在 Gitlab 创建仓库后 SSH 的地址就会显示 localhost ，导致其他小伙伴无法访问仓库。
```

更多配置参数请参考官方文档 [https://github.com/sameersbn/docker-gitlab#available-configuration-parameters](https://github.com/sameersbn/docker-gitlab#available-configuration-parameters)

## Jenkins

### 安装 Jenkins

我们直接从 Docker Hub 镜像仓库获取 Jenkins 最新镜像

```shell
$ docker pull jenkins:latest
#或者
$ docker pull jenkins:2.7.4
```

下载完之后运行 Docker Jenkins 容器

```shell
$ sudo docker run -d --name jenkins -p 10090:8080 -v /mnt/docker-jenkins:/var/jenkins_home jenkins:latest
```

解释一下命令的含义：在后台运行一个基于 `jenkins:latest` 镜像的容器，容器的名字叫 `jenkins`，把容器的 `8080` 端口映射为 `10090` 端口，并且把容器的 `/var/jenkins_home` 目录映射到宿主机的 `/mnt/docker-jenkins` 目录上

`-d` 后台运行 docker 容器，如果不加 `-d` 容器运行会占用此终端，终端关闭则容器也相应关闭，jenkins 就无法访问了。

`--name` 为容器起个别名，如果不起别名系统会默认分配一个随机别名，类似 gklasd_sdfwe 。起了别名后，后续会通过该别名管理该容器，也就是管理 jenkins 容器。

`-p` 容器端口映射，jenkins 服务是运行在 docker 容器里的，docker 容器默认不对外暴露端口

`-v` 目录映射，如果不映射，则 jenkins 所有 job、用户配置文件都会在容器内，如果容器销毁 jenkins 得重新配置一遍并且之前创建的 job 全丢了，映射出来是方便备份迁移和后续管理。


当然，启动的时候有可能会因为权限的问题报错，比如

```
touch: cannot touch ‘/var/jenkins_home/copy_reference_file.log’: Permission denied
Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions?
```

解决方案：
1、先查看容器的用户

```shell
$ docker run -ti --rm --entrypoint="/bin/bash" jenkins -c "whoami && id"

jenkins
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)
```

2、查看容器 `/var/jenkins_home` 目录权限

```shell
$ docker run -ti --rm --entrypoint="/bin/bash" jenkins -c "ls -la /var/jenkins_home"

total 20
drwxr-xr-x  2 jenkins jenkins 4096 Nov 22 07:43 .
drwxr-xr-x 26 root    root    4096 Nov  8 21:55 ..
-rw-r--r--  1 jenkins jenkins  220 Nov 12  2014 .bash_logout
-rw-r--r--  1 jenkins jenkins 3515 Nov 12  2014 .bashrc
-rw-r--r--  1 jenkins jenkins  675 Nov 12  2014 .profile
```

3、修改本地映射目录 `/mnt/docker-jenkins` 的用户组

```shell
$ sudo chown -R 1000 docker-jenkins/
```

4、重新运行上面的 run 命令

### 重装 Jenkins

最后讲下如果想重装怎么办，先查看 Docker 中运行的容器的情况，查看正在运行和已经停止的容器

```shell
$ docker ps -a
```

找到 Jenkins 的容器先停止再删除，这里需要用到上一步查到的容器 ID（这里删的只是容器，不是镜像，注意两者的区别）

```shell
$ docker stop 容器ID
$ docker rm 容器ID
```

删掉容器之后，再创建新的容器并加上运行参数

```shell
$ sudo docker run -d --name jenkins -p 10090:8080 -v /mnt/docker-jenkins:/var/jenkins_home jenkins:latest
```

如果需要删除镜像怎么办，先查一下镜像 ID

```shell
$ docker images
$ docker rmi 镜像ID
```

> 参考资料：[Docker — 从入门到实践](https://yeasy.gitbook.io/docker_practice/)


Done. :coffee: