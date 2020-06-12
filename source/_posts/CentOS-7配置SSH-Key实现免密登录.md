---
title: CentOS 7配置SSH Key实现免密登录
date: 2018-04-10 15:39:57
categories: 技术类
tags:
- SSH-Key
---

### SSH

简单的说，SSH 是一种网络协议，用于计算机之间的加密登录。整个过程是这样的。

* 远程主机收到用户的登录请求，把自己的公钥发给用户。
* 用户使用这个公钥，将登录密码加密之后，发送回来。
* 远程主机用自己的私钥，解密登录密码。如果密码正确，就同意用户登录。

### SSH Key 公钥登录

使用密码登录，每次都必须输入密码，非常麻烦。好在 SSH 还提供了公钥登录，可以省去输入密码的步骤。

所谓 "公钥登录"，原理很简单，就是用户将自己的公钥存在远程主机上。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先存储的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录 shell，不再要求密码。这种方法要求用户必须提供自己的公钥。

远程主机将用户的公钥，保存在登录后的用户主目录的 `$HOME/.ssh/authorized_keys` 文件中。公钥是一段字符串，只要把它追加在 `authorized_keys` 文件的末尾就可以了。

<!--more-->

### 如何配置

下面介绍 SSK Key 的使用配置

#### 生成密钥

打开终端，进入用户目录下的 `.ssh` 文件夹中，如果没有则创建 `.ssh` 目录，使用 `ssh-keygen` 命令生成 `rsa 4096` 位的密钥。如果是配置较低的 Windows 电脑在生成 4096 位秘钥时可能会非常慢，这样可以根据情况调低秘钥位数比如 2048，当然安全性也随之降低。

```shell
$ ssh-keygen -t rsa -C "your_email@example.com" -b 4096
```

-t 指定密钥类型，默认是 rsa ，可以省略。
-C 设置注释文字，比如邮箱。
-f 指定密钥文件存储文件名，可以忽略，默认是存在用户目录 .ssh 下面的

```
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/your/.ssh/id_rsa):在这里输入文件名
Enter passphrase (empty for no passphrase):输入ssh key的管理密码，可以不设置密码直接回车
Enter same passphrase again:重复密码
Your identification has been saved in id_rsa_test.
Your public key has been saved in id_rsa_test.pub.
The key fingerprint is:
SHA256:nLMdP1s5OHmQQezSjCXRQscEVXjaAd94Iu5PTEw7ujg test
The key's randomart image is:
+---[RSA 4096]----+
|         .+O*+.  |
|          oo*.oo |
|           Oo+=.o|
|       . .o.=*.+ |
|        S ..+ =  |
|         + + B o |
|        . . O B  |
|          E. X . |
|          ..o .  |
+----[SHA256]-----+
```

新生成的密钥默认写入 `/Users/your/.ssh/id_rsa` 文件，需要注意的是，如果之前生成过 ssh key，则会覆盖掉之前的 key，建议输入新的文件名存储新的 ssh key。在 .ssh 目录下会看到生成的两个文件 `id_rsa_test` 和 `id_rsa_test.pub`，前者为私钥文件，后者为公钥文件。

到此，本地的配置已完成。

题外话：可以使用 `config` 文件管理多个 Host，同时可以指定不同的 `ssh key`，当然这个 config 文件也是存在 .ssh 目录下的。格式如下
```
Host test #别名
HostName 1.2.3.4 #主机名
Port 22 #端口
User root #用户
IdentityFile ~/.ssh/id_rsa_test #密钥文件路径
PreferredAuthentications publickey #首选认证方式 公钥验证
```

#### 将公钥添加到服务器

先远程连上服务器，进入当前用户目录下的.ssh文件夹，创建 `authorized_keys` 文件用来存储公钥，如果之前有就不用创建了。然后将我们前面在本地生成的公钥文件内容 copy 到 `authorized_keys` 中，如果有多个客户端公钥，换行添加就可以。

注意一下 `.ssh` 和 `authorized_keys` 的权限，需要设置 `.ssh` 目录的权限为 `700`，需要设置 `authorized_keys` 文件的权限为 `600`。这里要特别注意，往往添加了密码但还是不能正常连接上服务器的一版都是权限问题。

```shell
$ chmod 700 ~/.ssh/
$ chmod 600 authorized_keys
```

#### 配置 sshd

编辑 `/etc/ssh/sshd_config` 文件将 `RSAAuthentication` 和 `PubkeyAuthentication` 的值改成 `yes`，为了安全建议禁止密码验证，将 `PasswordAuthentication` 的值改为 `no`，保存重启sshd

```shell
$ systemctl restart sshd
```

#### 客户端测试

```shell
$ ssh root@1.2.3.4
```


### Windows 生成 SSH KEY

在 Windows 下生成 ssh key 一般有两种方式，第一种就是在 Windows 下安装 `git`，然后鼠标右键，选择 `Git Bush here`，打开 `git bush`，然后就可以使用 `ssh-keygen` 命令生成 ssh key 了。这里不介绍如何安装 git，内事不决找baidu，外事不决找Google。

这里主要介绍另外一种方式：`PuTTY` 下载地址：[Download PuTTY: latest release (0.69)](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

打开 PuTTY 的安装目录，有一个 `puttygen.exe`，这是专门用来生成 key 的，支持多种加密方式。最底下 `Type of key to generate` 选择 `RSA`，`Number of bits in a generated key` 建议使用 `4096`，如果电脑配置很低就使用 `2048`，点击 `Generate` 开始生成 key，电脑配置越低，密钥位数越高，生成速度越慢。

![](https://i.loli.net/2020/06/12/ryfLWdMslZ4b6K3.png)

生成完之后，上面方框内的内容就是 RSA 的 Public key 公钥，公钥需要放到服务器。点击 `Save public key` 和 `Save private key` 将公钥和私钥保存下来。中间会提示你没使用密码来保存密钥，点击 `是` 不设置密码。

到这里 ssh key 就已经生成完成了，剩下的就是使用远程工具连接服务器，使用工具时需要手动配置私钥地址。不同工具之间稍微有点区别自己研究一下就能配好。


Done. :coffee: