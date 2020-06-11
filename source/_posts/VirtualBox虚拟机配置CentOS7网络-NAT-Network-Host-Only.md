---
title: VirtualBox虚拟机配置CentOS7网络(NAT Network + Host-Only)
date: 2018-09-21 19:32:57
updated: 2020-06-10 17:21:12
categories: 技术类
tags:
- VirtualBox
- 虚拟机
---

本篇介绍一下如何配置虚拟机网络，实现多台虚拟机可以正常访问外网同时内部可以互通，并且可以通过宿主机控制三台虚拟机。

先介绍一下环境
宿主机：`macOS High Sierra 10.13.6` 
虚拟机工具：`Oracle VM VirtualBox 5.2.18` 
虚拟机系统：`CentOS 7`

## 创建虚拟机

VirtualBox 的安装这里就不介绍，具体可以 Google。下面介绍如何使用 VirtualBox 创建虚拟机。

选择 `新建`，在弹出框中输入虚拟机名称并选择类型和版本，本例中安装的是 CentOS 7 所以类型选择 Linux 版本选择 Red Hat(64-bit)，点击继续直到创建成功。

<!--more-->

![1.png](https://i.loli.net/2020/06/11/7nDRSFrNgqsoLcz.png)

选中新创建的虚拟机，点击 `设置`，在弹出框中选中 `存储`，点击左侧的 `没有盘片`，在右侧点击光盘的 icon，选择本地事先准备好的 CentOS 7 的镜像文件（镜像在官网下载 [http://www.centos.org](http://www.centos.org)），然后点 OK。

![2.png](https://i.loli.net/2020/06/11/VYwIcZOeiUWKqTN.png)

配置好系统镜像之后，启动虚拟机。

![3.png](https://i.loli.net/2020/06/11/ZDlOmA5zX3hdFUS.png)

选择 `Install CentOS 7`。

常规检查完成之后进入系统安装界面，language 可以默认 English 或者选择中文，然后点击 `Continue`。

![4.png](https://i.loli.net/2020/06/11/nO62zUSCY3uLsRK.png)

这里会自动检查时区语言和键盘等设置，都是自动完成的，但系统盘的盘符设置需要手动确认一下。

![5.png](https://i.loli.net/2020/06/11/vztqn8mxrDUH6ky.png)

点击 `INSTALLATION DESTINATION` 然后点击 `Done`。

![6.png](https://i.loli.net/2020/06/11/gYUJ62fWDN4Qlk5.png)

然后等待系统检测没问题之后，`Begin Installation` 按钮就可以点击了。

![7.png](https://i.loli.net/2020/06/11/19XzJcGgNlyKe2A.png)

然后等待系统安装完成，这里可以设置一下ROOT用户的密码，点击 `ROOT PASSWORD`，设置完密码后点击 `Done`。

![8.png](https://i.loli.net/2020/06/11/p78Tdat5Z4RufMv.png)

安装完成，重启点击 `Reboot`。

![9.png](https://i.loli.net/2020/06/11/YtsNTiG76anvQzc.png)

到这里 CentOS 7 安装完成，输入账号和密码进入系统。

![10.png](https://i.loli.net/2020/06/11/zjZKhGTCoLyvDOa.png)

## 网络配置

本例演示的方式是 `NAT Network` 加上 `Host-Only` 的模式，所以这里我们需要准备两张网卡，先停掉虚拟机 `shutdown now`。

1、NAT Network 网卡配置

选择 VirtualBox 的 `偏好设置`，点击 `网络`，点击右侧的加号新增 NAT 网络后，点击 `OK`。

![11.png](https://i.loli.net/2020/06/11/Vo5ibHexXM1wnTm.png)

2、Host-Only 网卡配置

选择 VirtualBox 的 `全局工具`，选择 `主机网络管理器`。

![12.png](https://i.loli.net/2020/06/11/pjnGFCsq5DScvtM.png)

点击 `创建`

![13.png](https://i.loli.net/2020/06/11/Uh3SvMmkRgPwf2T.png)

到这里网卡就添加好了，现在我们需要将添加的网卡配置到虚拟机系统上。

选中上面创建的虚拟机，点击 `设置`，在弹出框中选择 `网络`，选择 `网卡1` 并启用网络连接，将连接方式改成 `NAT网络`，界面名称会默认选中 `NatNetwork` 这就是上面我们准备的第一张网卡。

![14.png](https://i.loli.net/2020/06/11/dUazKSeETX2W4Jl.png)

接着我们配置第二张网卡，还是上面的界面，选择 `网卡2`，勾选 `启用网络连接`，连接方式选择 `仅主机(Host-Only)网络`，界面名称会默认选中 `vboxnet0` 这就是上面我们准备的第二张网卡。

![15.png](https://i.loli.net/2020/06/11/DbhgJi9ZsWjL5Up.png)

点击 OK，这里就将上面我们准备的两张网卡都配置到虚拟机系统中了。

> 这里有个细节需要注意一下，在网卡1、网卡2高级设置中，两张网卡都会有 MAC 地址，这里的地址我们下面在配置网络的时候需要使用到

![16.png](https://i.loli.net/2020/06/11/naJsmzoR7dNkqQL.png)

## 虚拟机访问外网

因为我安装的系统镜像是 `Minimal ISO` 最小镜像，网络配置需要手动修改，以及很多其他依赖包也需要手动安装，这里我们先配置虚拟机让它能正常访问外网。

```shell
$ cd /etc/sysconfig/network-scripts/
```

![17.png](https://i.loli.net/2020/06/11/3stIY81HAlFbBf7.png)

编辑网卡配置文件 `ifcfg-enp0s3`。

```shell
$ vi ifcfg-enp0s3
```

![18.png](https://i.loli.net/2020/06/11/ZNxuUA3n6OFMR2p.png)

在配置中增加 `HWADDR` 参数，并修改 `ONBOOT` 参数的值为 `yes`。

![19.png](https://i.loli.net/2020/06/11/YNP1UGa8msryzek.png)

> 这里 `HWADDR` 的值就是上面配置的 NAT 网卡的 MAC 地址
> 这里改的 `enp0s3` 文件是 `网卡1` `NAT网卡` 的配置文件

修改完配置文件保存退出，然后重启网卡。

```shell
$ systemctl restart network
```

![20.png](https://i.loli.net/2020/06/11/cqTdsxJmpvR6a9W.png)

测试外网是否可以正常访问。

```shell
$ ping baidu.com
```

![21.png](https://i.loli.net/2020/06/11/y3SudZ6Re1vLEiJ.png)

不出意外，到这里虚拟机访问外网就完成了，下面我们继续配置，让宿主机可以访问虚拟机。

## 宿主机访问虚拟机

上面我们添加的 `网卡2`，连接方式为 `仅主机(Host-Only)网络` 就是专门用来让宿主机连接的。回到 CentOS 系统内，我们先查看一下网卡的情况

```shell
$ ip addr
```

![22.png](https://i.loli.net/2020/06/11/YTL8rphqxeAtMNV.png)

这里我们发现有两张网卡，`enp0s3` 和 `enp0s8`，第一张 `enp0s3` 就是上面一步配置虚拟机访问外网的网卡，可以看到 `dhcp` 默认分配的ip地址是 `10.0.2.7`。但第二张网卡没有正常启用，现在我们需要对第二张网卡进行配置，还是进入到 `/etc/sysconfig/network-scripts/` 目录下

```shell
$ cd /etc/sysconfig/network-scripts/
```

进入后会发现根本就没有第二张网卡对应的配置文件 `ifcfg-enp0s8`，不要方，我们把 `ifcfg-enp0s3` 文件复制一份改个名字就可以了。

```shell
$ cp ifcfg-enp0s3 ifcfg-enp0s8
```

![23.png](https://i.loli.net/2020/06/11/W7k4eCqR32zPmxF.png)

接下来修改 `ifcfg-enp0s8` 文件

把 `HWADDR` 的值改为网卡2的 MAC 地址

将 `BOOTPROTO` 的值改为 `static`

将 `NAME` 和 `DEVICE` 的值改为 `enp0s8`

将 `UUID` 的值改为和 `ifcfg-enp0s3` 中的不一样，可以随便填

增加 `IPADDR`，手动设置一个静态IP

增加 `NETMASK=255.255.255.0`

![24.png](https://i.loli.net/2020/06/11/hezrgJHqdL6MNFP.png)

配置完成后重启网卡，然后查看网卡情况，可以看到 `enp0s8` 正常启用了。

![25.png](https://i.loli.net/2020/06/11/3PaHRJVMBTkpb2L.png)

我们在宿主机上尝试ping一下虚拟机。

![26.png](https://i.loli.net/2020/06/11/9vywAF2tsCapDSI.png)

完全正常，接着尝试 ssh 到虚拟机，也是可以正常连接的。

![27.png](https://i.loli.net/2020/06/11/I5v9w7groV1aUMW.png)

> 上面 `IPADDR` 的静态IP配置需要参考 `Host-Only网卡配置` 中的 `IPv4网络掩码`
> 比如是 `192.168.56.1/24`，这里我们在设置静态IP的时候最后一位就可以在2-254中间随便设置

## 多台虚拟机内网互通

完成上面的配置后，一台机器的网络配置就完成了，剩下的两台按照上面的步骤虚拟机访问外网、宿主机访问虚拟机就可以了，这里要注意，全局添加的 `NAT Network网卡` 和 `Host-Only网卡` 不用重复添加，多台虚拟机可以公用

多台机器配置完成后，内网就是互通的，一般默认情况下，`NAT Network网卡` 使用的是 `10.0.2` 网段


Dnoe. :coffee: