---
title: NFS安装配置和使用
date: 2018-03-22 17:11:41
updated: 2019-05-10 10:20:11
categories: 技术类
tags:
- NFS
---

## NFS 服务介绍

NFS 是 Network File System 的缩写，即网络文件系统。一种使用于分散式文件系统的协定，由 Sun 公司开发，于1984年向外公布。功能是通过网络让不同的机器、不同的操作系统能够彼此分享个别的数据，让应用程序在客户端通过网络访问位于服务器磁盘中的数据，是在类 Unix 系统间实现磁盘文件共享的一种方法。

NFS 的基本原则是“容许不同的客户端及服务端通过一组 RPC 分享相同的文件系统”，它是独立于操作系统，容许不同硬件及操作系统的系统共同进行文件的分享。

NFS 在文件传送或信息传送过程中依赖于 RPC 协议。RPC 远程过程调用 (Remote Procedure Call) 是能使客户端执行其他系统中程序的一种机制。NFS 本身是没有提供信息传输的协议和功能的，但 NFS 却能让我们通过网络进行资料的分享，这是因为 NFS 使用了一些其它的传输协议。而这些传输协议用到这个 RPC 功能的。可以说 NFS 本身就是使用RPC的一个程序。或者说 NFS 也是一个 RPC SERVER。所以只要用到 NFS 的地方都要启动 RPC 服务，不论是 NFS SERVER 或者 NFS CLIENT。这样 SERVER 和 CLIENT 才能通过 RPC 来实现 PROGRAM PORT 的对应。可以这么理解 RPC 和 NFS 的关系：NFS 是一个文件系统，而 RPC 是负责负责信息的传输。

<!--more-->

## 系统约定

服务端：120.24.79.96

客户端：112.74.96.60

## 安装

服务端和客户端都要安装 nfs 和 rpcbind

```shell
$ yum -y install nfs-utils rpcbind
```

服务器端配置开机启动，客户端不需要

```shell
$ systemctl enable rpcbind
$ systemctl enable nfs-server
```

启动服务

```shell
# rpcbind 必须在 nfs 服务前面启动
$ systemctl start rpcbind
$ systemctl start nfs-server
```

服务器端设置 NFS 卷输出，即编辑 `/etc/exports` 添加：

```shell
$ /opt/nfsshare/upload 112.74.96.60(rw,sync,no_root_squash)
```

`/opt/nfsshare/upload` 共享目录
`112.74.96.60` 允许访问 NFS 的客户端 IP 地址段
`rw` 允许对共享目录进行读写
`sync` 实时同步共享目录
`no_root_squash` 允许root访问
`no_all_squash` 允许用户授权
`no_subtree_check` 如果卷的一部分被输出，从客户端发出请求文件的一个常规的调用子目录检查验证卷的相应部分。如果是整个卷输出，禁止这个检查可以加速传输。


重新加载 NFS 配置文件

```shell
$ exportfs -r
```

## NFS 客户端挂载

先查看是否有挂载点

```shell
$ showmount -e 120.24.79.96
```

Linux 挂载 NFS 的客户端非常简单的命令，先创建挂载目录，然后用 `-t nfs` 参数挂载就可以了，记得客户端的 `/mnt/nfsfrom96` 目录要存在。

```shell
$ mount -t nfs 120.24.79.96:/opt/nfsshare/upload /mnt/nfsfrom96
```

如果要设置客户端启动时候就挂载 NFS，可以配置 `/etc/fstab` 添加以下内容

```shell
120.24.79.96:/opt/nfsshare/upload /mnt/nfsfrom96 nfs auto,rw,vers=3,hard,intr,tcp,rsize=32768,wsize=32768 0 0
```

检查是否挂载成功

```shell
$ df -h
```

如果要解除挂载

```shell
$ umount 120.24.79.96:/opt/nfsshare/upload /mnt/nfsfrom96
```

## 常见错误

mount.nfs: an incorrect mount option was specifiedt

这是由于 Linux 和 solaris10 使用了不同版本的 NFS 导致，solaris10 默认使用的是 NFS4，这导致了 Solaris nfs 和 Linux nfs 的兼容问题，可以加上参数 `-o vers=3` 或者 `nfsvers=3,vers=3` 来解决。

```shell
$ mount -t nfs 120.24.79.96:/opt/nfsshare/upload /mnt/nfsfrom96 -o vers=3
```

另外还有因为 IPV6 原因导致挂载失败的需要关闭 IPV6 监听

先找到 rpcbind 的 socket 文件

```shell
$ find /etc -name 'rpcbind.socket'
```

编辑注释掉 `ListenStream=[::]:111` 之后重新加载配置并重启服务

```shell
$ systemctl daemon-reload
$ systemctl restart rpcbind.socket
```

> daemon-reload: 重新加载某个服务的配置文件，如果新安装了一个服务，归属于 systemctl 管理，要是新服务的服务程序配置文件生效，需重新加载。

之后就可以按照上面的客户端挂载流程继续执行了。


Done. :coffee: