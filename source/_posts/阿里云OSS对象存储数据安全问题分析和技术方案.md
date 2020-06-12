---
title: 阿里云 OSS 对象存储数据安全问题分析和技术方案
date: 2017-03-29 11:00:28
categories: 技术类
tags:
- OSS
---

## 前言

阿里云对象存储服务（Object Storage Service，简称 OSS），是阿里云提供的海量、安全、低成本、高可靠的云存储服务。您可以通过调用 API，在任何应用、任何时间、任何地点上传和下载数据，也可以通过 Web 控制台对数据进行简单的管理。OSS 适合存放任意类型的文件，适合各种网站、开发企业及开发者使用。

在使用过 OSS 服务之后我们发现，存储在 OSS 服务器的文件是未加密的，通过 OSS 管理后台可以很方便的预览存储的文件内容。但如果不想让第三方或者未经授权的人随意访问查看文件，这里就需要做一些处理，比如数据加密上传、授权临时访问权限等。基于此需求，下面就来讲讲如何来保护数据。

<!--more-->

## 方案一

OSS 支持在服务器端对用户上传的数据进行加密编码（Server-Side Encryption）：用户上传数据时，OSS 对收到的用户数据进行加密编码，然后再将编码得到的数据永久保存下来；用户下载数据时，OSS 自动对保存的编码数据进行解码并把原始数据返回给用户，并在返回的 HTTP 请求 Header 中声明该数据进行了服务器端加密编码。换句话说，下载一个进行服务器端加密编码的 Object 和下载一个普通的Object 没有多少区别，因为 OSS 会为用户管理整个编解码过程。

OSS 的服务器端加密编码是 Object 的一个属性。用户创建一个 Object 的时候，只需要在 Put Object 的请求中携带 `x-oss-server-side-encryption` 的 HTTP Header，并指定其值为 `AES256`，即可以实现该 Object 的服务器端加密编码存储。目前支持服务器端加密编码的操作包括：

1、Put Object
2、Copy Object
3、Initiate Multipart Upload

除了 `Put Object`、`Copy Object` 和 `Initiate Multipart Upload` 以外，其他 OSS 收到的请求中如果出现 `x-oss-server-side-encryption` 头，OSS 会直接返回 HTTP 状态码：400；并在消息体内注明错误码是：`InvalidArgument`。

## 方案二

用户数据在发送给 OSS 服务器之前就完成加密，而加密所用的密钥的明文只保留在本地，从而可以保证用户数据安全，即使数据泄漏别人也无法解密得到原始数据。

![](https://i.loli.net/2020/06/12/Kt4VkCHJv7dOSD2.jpg)

### 流程详解

Put Object
1、Client 端上传文件到 Client Server
2、Client Server 得到文件后对文件内容进行加密
3、加密完成后将密文上传到 OSS 服务器

Get Object
1、Client发送获取文件请求到 Client Server
2、Client Server 向 OSS 服务器获取文件内容
3、Client Server 获取到的内容为密文，得到内容后解密
4、将解密后的文件返回给 Client

### 技术方案介绍

1、用户本地维护一对 RSA 密钥( `rsa_private_key` 和 `rsa_public_key` )。
2、每次上传 Object 时，随机生成一个 AES256 类型的对称密钥 `data_key`，然后用 `data_key` 加密原始 content 得到 encrypt_content 。
3、用 `rsa_public_key` 加密 `data_key`，得到 `encrypt_data_key`，作为用户的自定义 meta 放入请求头部，和 encrypt_content 一起发送到 OSS。
4、Get Object 时，首先得到 encrypt_content 以及用户自定义 meta 中的 `encrypt_data_key`。
5、用户使用 `rsa_private_key` 解密 `encrypt_data_key` 得到 `data_key`，然后用 `data_key` 解密 encrypt_content 得到原始 content。


> 本文用户的密钥为非对称的 RSA 密钥, 加密 object content 时用的 `AES256-CTR` 算法, 详情可参考 [PyCrypto Document] 。本文旨在介绍如何通过 object 的自定义 meta 来实现客户端加密，至于加密密匙类型及加密算法，用户可以根据自己的需要进行选择。

### 架构图

![](https://i.loli.net/2020/06/12/VvRoSjhxFcnmbMI.jpg)

### 代码示例

#### Client Server 加密文件

```Python
# -*- coding: utf-8 -*-

import os
import shutil
import base64
import random
import oss2
from Crypto.Cipher import PKCS1_OAEP
from Crypto.PublicKey import RSA
from Crypto.Cipher import AES
from Crypto import Random
from Crypto.Util import Counter

from os import path

# 需要上传的文件名称
obj_name = 'GCD.pdf'

# aes 256, key always is 32 bytes
_AES_256_KEY_SIZE = 32
_AES_CTR_COUNTER_BITS_LEN = 8 * 16
class AESCipher:
    def __init__(self, key=None, start=None):
        self.key = key
        self.start = start
        if not self.key:
            self.key = Random.new().read(_AES_256_KEY_SIZE)
        if not self.start:
            self.start = random.randint(1, 10)
        ctr = Counter.new(_AES_CTR_COUNTER_BITS_LEN, initial_value=self.start)
        self.cipher = AES.new(self.key, AES.MODE_CTR, counter=ctr)
    def encrypt(self, raw):
        return self.cipher.encrypt(raw)
    def decrypt(self, enc):
        return self.cipher.decrypt(enc)
# 首先初始化AccessKeyId、AccessKeySecret、Endpoint等信息。
# 通过环境变量获取，或者把诸如“<您的AccessKeyId>”替换成真实的AccessKeyId等。
#
# 以杭州区域为例，Endpoint可以是：
#   http://oss-cn-hangzhou.aliyuncs.com
#   https://oss-cn-hangzhou.aliyuncs.com
# 分别以HTTP、HTTPS协议访问。
access_key_id = os.getenv('OSS_TEST_ACCESS_KEY_ID', 'wtiUZd71Qvb0zIsQ')
access_key_secret = os.getenv('OSS_TEST_ACCESS_KEY_SECRET', 'aOywWxrs9kc33CrhABv4f10gDqEttY')
bucket_name = os.getenv('OSS_TEST_BUCKET', 'elsdocman')
endpoint = os.getenv('OSS_TEST_ENDPOINT', 'http://oss-cn-shenzhen.aliyuncs.com')
# 确认上面的参数都填写正确了
for param in (access_key_id, access_key_secret, bucket_name, endpoint):
    assert '<' not in param, '请设置参数：' + param
##### 0 prepare ########

# 先判断本地是否已经存在秘钥
if path.exists("private_key.pem") and path.exists("public_key.pem"):
    # 从本地读取公钥
    print "秘钥已存在，读取本地秘钥"
    with open("public_key.pem", 'rb') as f:
        data = f.read()
    public_key = RSA.importKey(data)
    encrypt_obj = PKCS1_OAEP.new(public_key)
else:
    print "生成新的秘钥文件"
    # 0.1 生成rsa key文件并保存到disk 
    rsa_private_key_obj = RSA.generate(2048)
    rsa_public_key_obj = rsa_private_key_obj.publickey()

    encrypt_obj = PKCS1_OAEP.new(rsa_public_key_obj)
    # decrypt_obj = PKCS1_OAEP.new(rsa_private_key_obj)
    # save to local disk 
    file_out = open("private_key.pem", "w")
    file_out.write(rsa_private_key_obj.exportKey())
    file_out.close()
    file_out = open("public_key.pem", "w")
    file_out.write(rsa_public_key_obj.exportKey())
    file_out.close()

# 0.2 创建Bucket对象，所有Object相关的接口都可以通过Bucket对象来进行
bucket = oss2.Bucket(oss2.Auth(access_key_id, access_key_secret), endpoint, bucket_name)
content = open(obj_name, "rb")
print "需要上传的文件名: %s" % obj_name

#### 1 Put Object  ####
# 1.1 生成加密这个object所用的一次性的对称密钥 encrypt_cipher, 其中的key 和 start为随机生成的value
encrypt_cipher = AESCipher()
print "key: %s start: %s" % (encrypt_cipher.key, encrypt_cipher.start)
# 1.2 将辅助解密的信息用公钥加密后存到object的自定义meta中. 后续当我们get object时，就可以根据自定义meta，用私钥解密得到原始content
headers = {}
headers['x-oss-meta-x-oss-key'] = base64.b64encode(encrypt_obj.encrypt(encrypt_cipher.key))
headers['x-oss-meta-x-oss-start'] = base64.b64encode(encrypt_obj.encrypt(str(encrypt_cipher.start)))
print "封装请求头信息: %s" % headers
# 1.3. 用 encrypt_cipher 对原始content加密得到encrypt_content
print "开始加密文件"
encryt_content = encrypt_cipher.encrypt(content.read())
print "文件加密完成"
# 1.4 上传object
print "开始上传文件"
result = bucket.put_object(obj_name, encryt_content, headers)
if result.status / 100 != 2:
    print "文件上传失败: %s" % result.status
    exit(1)
else:
    print "文件上传成功"
```


#### Client Server 解密文件

```Python
# -*- coding: utf-8 -*-

import os
import shutil
import base64
import random
import oss2
from Crypto.Cipher import PKCS1_OAEP
from Crypto.PublicKey import RSA
from Crypto.Cipher import AES
from Crypto import Random
from Crypto.Util import Counter

from os import path

# 需要下载的文件名称
obj_name = 'GCD.pdf'

# aes 256, key always is 32 bytes
_AES_256_KEY_SIZE = 32
_AES_CTR_COUNTER_BITS_LEN = 8 * 16
class AESCipher:
    def __init__(self, key=None, start=None):
        self.key = key
        self.start = start
        if not self.key:
            self.key = Random.new().read(_AES_256_KEY_SIZE)
        if not self.start:
            self.start = random.randint(1, 10)
        ctr = Counter.new(_AES_CTR_COUNTER_BITS_LEN, initial_value=self.start)
        self.cipher = AES.new(self.key, AES.MODE_CTR, counter=ctr)
    def encrypt(self, raw):
        return self.cipher.encrypt(raw)
    def decrypt(self, enc):
        return self.cipher.decrypt(enc)
# 首先初始化AccessKeyId、AccessKeySecret、Endpoint等信息。
# 通过环境变量获取，或者把诸如“<您的AccessKeyId>”替换成真实的AccessKeyId等。
#
# 以杭州区域为例，Endpoint可以是：
#   http://oss-cn-hangzhou.aliyuncs.com
#   https://oss-cn-hangzhou.aliyuncs.com
# 分别以HTTP、HTTPS协议访问。
access_key_id = os.getenv('OSS_TEST_ACCESS_KEY_ID', 'wtiUZd71Qvb0zIsQ')
access_key_secret = os.getenv('OSS_TEST_ACCESS_KEY_SECRET', 'aOywWxrs9kc33CrhABv4f10gDqEttY')
bucket_name = os.getenv('OSS_TEST_BUCKET', 'elsdocman')
endpoint = os.getenv('OSS_TEST_ENDPOINT', 'http://oss-cn-shenzhen.aliyuncs.com')
# 确认上面的参数都填写正确了
for param in (access_key_id, access_key_secret, bucket_name, endpoint):
    assert '<' not in param, '请设置参数：' + param
##### 0 prepare ########
# 0.1 生成rsa key文件并保存到disk 
# rsa_private_key_obj = RSA.generate(2048)
# rsa_public_key_obj = rsa_private_key_obj.publickey()
# encrypt_obj = PKCS1_OAEP.new(rsa_public_key_obj)
# decrypt_obj = PKCS1_OAEP.new(rsa_private_key_obj)

# 读取本地的私钥
def read_private_key():
    with open("private_key.pem", 'rb') as f:
        data = f.read()
    key = RSA.importKey(data)
    return key

# 判断文件是否存在
if path.exists("private_key.pem"):
    private_key = read_private_key()
    decrypt_obj = PKCS1_OAEP.new(private_key)
else:
    print("没有找到私钥文件")
    exit(1)


# save to local disk 
# file_out = open("private_key.pem", "w")
# file_out.write(rsa_private_key_obj.exportKey())
# file_out.close()
# file_out = open("public_key.pem", "w")
# file_out.write(rsa_public_key_obj.exportKey())
# file_out.close()
# 0.2 创建Bucket对象，所有Object相关的接口都可以通过Bucket对象来进行
bucket = oss2.Bucket(oss2.Auth(access_key_id, access_key_secret), endpoint, bucket_name)
print "需要下载的文件名: %s" % obj_name



#### 2 Get Object ####
# 2.1 下载得到加密后的object
print "开始下载文件"
result = bucket.get_object(obj_name)
if result.status / 100 != 2:
    print "文件下载失败: %s" % result.status
    exit(1)
else:
    print "文件下载成功"

resp = result.resp
download_encrypt_content = resp.read()
print "开始准备解密"
# print download_encrypt_content
# 2.2 从自定义meta中解析出之前加密这个object所用的key 和 start 
download_encrypt_key = base64.b64decode(resp.headers.get('x-oss-meta-x-oss-key', ''))
key = decrypt_obj.decrypt(download_encrypt_key)
download_encrypt_start = base64.b64decode(resp.headers.get('x-oss-meta-x-oss-start', ''))
start = int(decrypt_obj.decrypt(download_encrypt_start))
print (key, start)
# 2.3 生成解密用的cipher, 并解密得到原始content
decrypt_cipher = AESCipher(key, start)
download_content = decrypt_cipher.decrypt(download_encrypt_content)
# if download_content != content:
    # print "Error!"
# else:
# print "Decrypt ok. Content is: %s" % download_content
print "文件解密完成，准备将文件写到本地"
# 将下载下来的文件写到本地
output = open("new_"+obj_name, "wb")
output.writelines(download_content)
output.close()
print "文件写入完成，文件名: new_%s" % obj_name

```

## AES256-CTR 加解密效率测试

测试环境：

* MacBook Pro macOS Sierra 10.12.2 
* 2.6 GHz Intel Core i5
* 8 GB 1600 MHz DDR3

| 文件大小     | 加密耗时    |  解密耗时  |
| :--------:   | :-----:     | :----:     |
| 704.9 MB     | 13s         |   9s       |
| 202.8 MB     |    3s       |  3s        |
| 30.1 MB      |   1s        |   <1s      |
| 16.5 MB      |   <1s       |   <1s      |


[PyCrypto Document]:https://www.dlitz.net/software/pycrypto/api/2.6/?spm=5176.doc32259.2.1.v3K4Oo


Done. :coffee: