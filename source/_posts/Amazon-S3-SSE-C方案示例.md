---
title: Amazon S3 SSE-C方案示例
date: 2018-05-03 17:36:06
categories: 技术类
tags:
- AWS
- S3
---

## 开发环境

Mac OS
JDK1.7
Eclipse

## 前言

亚马逊云分国内亚马逊云存储([AWS中国](https://www.amazonaws.cn/))和国外亚马逊云存储([AWS全球](https://aws.amazon.com))，这两套账号是独立的，在注册账号时需要留意。国内的亚马逊云存储：基本下载、分段下载、基本上传、分段上传这4个功能，整个过程网络速度没有优化，按照实际的网络速度上传。国外的亚马逊云存储：基本下载、分段下载、基本上传、分段上传这4个功能，还有额外的收费一个功能，加速下载和加速上传；整个过程会充分利用网络资源提高数据传输速度。

国内的亚马逊云存储加密功能是没有阉割的，国内上传文件到国内亚马逊云存储，上传速度平均是 540K/s ，高峰时段速度会降低。国内上传文件到国外亚马逊云存储，上传速度是 62k/s ，没有使用加速功能。

<!--more-->

## 账号配置

### 创建 Bucket

Bucket 就是桶，也叫存储桶，用来存放资源的。直接在 AWS 管理页面上操作就不介绍了

### 创建 IAM 用户

1、在管理平台添加一个用户，访问类型选编程访问，权限先不用给

2、创建完成后记录下 `访问私钥ID` 和 `私有访问密钥`，很重要，后面会用到

3、点开创建的用户，给用户添加权限

4、点击添加内联策略，选择策略生成器

5、AWS 服务选择 `Amazon S3`

6、操作先选读写 `PutObject` 和 `GetObject`，不够后面再改

7、ARN 填*，这里其实可以对资源进行权限控制，很强大

8、添加声明，下一步，验证策略，没问题就应用策略

9、当前创建的 IAM 用户就可以上传和下载 Object 了

## AWS 环境搭建

### 下载 SDK

1、下载最新版的 [AWS SDK](https://sdk-for-java.amazonwebservices.com/latest/aws-java-sdk.zip)

2、解压 `aws-java-sdk.zip` 并将 `lib` 和 `third-party/lib` 下的 `jar` 添加到工程中

### 环境变量配置（Credentials and Region）

[参考这里](http://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/setup-credentials.html)

#### 设置 Credentials

在 Linux、OS X 或 Unix 环境下，like this `~/.aws/credentials`

在 `~/` 目录下创建 `.aws` 文件夹和 `credentials` 文件

在 Windows 环境下

同上创建 `.aws` 文件夹和 `credentials` 文件，like this `C:\Users\USERNAME\.aws\credentials`

然后在创建的文件中写入一下内容

```
[default]
aws_access_key_id = your_access_key_id
aws_secret_access_key = your_secret_access_key
```

把 `your_access_key_id` 和 `your_secret_access_key` 替换成自己对应的信息，这些信息就是上面创建 IAM 用户时生成的 `访问私钥ID` 和 `私有访问密钥`

#### 设置 Region

在上面创建的 `.aws` 目录下再创建一个 `config` 文件，Windows 或者 Linux 都一样

`~/.aws/config` 在 Linux, OS X, or Unix 系统这样

`C:\Users\USERNAME\.aws\config` 在 Windows 系统这样

然后在创建的 config 文件中写入一下内容

```
[default]
region = your_aws_region
```

把 `your_aws_region` 替换成自己对应的信息，这些信息可以在管理平台找到，不填默认是 `us-east-1` (If you don't select a region, then us-east-1 will be used by default.)

## 写代码

下面代码中验证了上传没加密文件、下载没加密文件、上传文件且服务端加密、下载加密文件、下载加密文件不给密钥、获取加密文件的元数据、获取没加密文件的元数据

```java
package com.xxx.demo;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.nio.file.Paths;
import java.security.SecureRandom;

import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;

import com.amazonaws.auth.profile.ProfileCredentialsProvider;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3Client;
import com.amazonaws.services.s3.model.GetObjectMetadataRequest;
import com.amazonaws.services.s3.model.GetObjectRequest;
import com.amazonaws.services.s3.model.ObjectMetadata;
import com.amazonaws.services.s3.model.PutObjectRequest;
import com.amazonaws.services.s3.model.S3Object;
import com.amazonaws.services.s3.model.S3ObjectInputStream;
import com.amazonaws.services.s3.model.SSECustomerKey;
import com.amazonaws.util.Base64;

public class SSECDemo {

	private static String bucketName = "softgoto";// 后台已经创建名为 softgoto 的 Bucket 用来测试
    private static String keyName = "";
    
    private static String workspace = "/Users/softgoto/Desktop/";// 读取文件和输出文件的路径

    private static AmazonS3 s3client;
    private static SecretKey secretKey;
    private static SSECustomerKey sseKey;

    public static void main(String[] args) {
    	s3client = new AmazonS3Client(new ProfileCredentialsProvider());
    	System.out.println("----------------创建密钥----------------");
    	// 创建加密key
        secretKey = generateSecretKey();
        sseKey = new SSECustomerKey(secretKey);
    	
        System.out.println("\n\n");
        System.out.println("----------------上传加密----------------");
    	//上传文件服务端加密
    	serverSideEncryptionPutObject();
    	
    	System.out.println("\n\n");
        System.out.println("----------------上传不加密----------------");
    	//上传文件服务端不加密
    	serverSideDonotEncryptionPutObject();
    	
    	System.out.println("\n\n");
        System.out.println("----------------获取加密文件不给密钥----------------");
    	//获取加密文件不给密钥
    	getObject("加密文件.jpg", null);
    	
    	System.out.println("\n\n");
        System.out.println("----------------获取加密文件给密钥----------------");
    	//获取加密文件给密钥
    	getObject("加密文件.jpg", sseKey);
    	
    	System.out.println("\n\n");
        System.out.println("----------------获取没有加密的文件----------------");
    	//获取没有加密的文件
    	getObject("不加密文件.png", null);
    	
    	System.out.println("\n\n");
        System.out.println("----------------获取加密文件元数据----------------");
        //获取加密文件元数据
        retrieveObjectMetadata("加密文件.jpg", sseKey);
        
        System.out.println("\n\n");
        System.out.println("----------------获取未加密文件元数据----------------");
        //获取未加密文件元数据
        retrieveObjectMetadata("不加密文件.png", null);
    	
    	System.out.println("\n\n");
        System.out.println("----------------Done!----------------");
    }
    
    /**
     * 服务端加密上传
     */
    private static void serverSideEncryptionPutObject() {
    	String uploadFileName = "加密文件.jpg";
    	keyName = Paths.get(workspace + uploadFileName).getFileName().toString();
        try {
            File file = new File(workspace + uploadFileName);
            
            System.out.println("上传文件，并且让服务端加密");
            System.out.println("AES256密钥: "+Base64.encodeAsString(secretKey.getEncoded()));
            
            // 上传文件
            uploadObject(file, sseKey);

         } catch (Exception e) {
            System.out.println(e);
         }
    }
    
    /**
     * 获取文件-传或者不传密钥
     * @param sseKey
     */
    private static void getObject(String fileName, SSECustomerKey sseKey) {
    	try {
    		keyName = Paths.get(workspace + fileName).getFileName().toString();
    		
    		if(sseKey != null){
    			System.out.println("AES256密钥: "+Base64.encodeAsString(secretKey.getEncoded()));
    		}
    		
			// 下载文件
			downloadObject(sseKey);
		} catch (Exception e) {
			System.out.println(e);
		}
    }
    
    /**
     * 服务端不加密上传
     */
    private static void serverSideDonotEncryptionPutObject() {
    	String uploadFileName = "不加密文件.png";
    	keyName = Paths.get(workspace + uploadFileName).getFileName().toString();
        try {
            File file = new File(workspace + uploadFileName);
            System.out.println("上传文件，服务端不加密");
            
            // 上传文件
            uploadObject(file, null);

         } catch (Exception e) {
            System.out.println(e);
        }
    }
    

    private static PutObjectRequest uploadObject(File file, SSECustomerKey sseKey) {
        // 1. Upload Object.
        PutObjectRequest putObjectRequest = null;
        
        if(sseKey == null){
        	putObjectRequest = new PutObjectRequest(bucketName, keyName, file);
        }else{
        	putObjectRequest = new PutObjectRequest(bucketName, keyName, file).withSSECustomerKey(sseKey);
        }

        s3client.putObject(putObjectRequest);
        System.out.println("文件上传完成!");
        return putObjectRequest;
    }

    private static void downloadObject(SSECustomerKey sseKey) {
        // Get a range of bytes from an object.
    	System.out.println("开始下载保存文件");
        GetObjectRequest getObjectRequest = null;

        if(sseKey == null){
        	getObjectRequest = new GetObjectRequest(bucketName, keyName);
        }else{
        	getObjectRequest = new GetObjectRequest(bucketName, keyName).withSSECustomerKey(sseKey);
        }
        
        S3Object s3Object = s3client.getObject(getObjectRequest);

        saveInputStream(s3Object.getObjectContent(), "new_"+s3Object.getKey());
    }
    
    private static void retrieveObjectMetadata(String fileName, SSECustomerKey sseKey) {
        GetObjectMetadataRequest getMetadataRequest = null;

        if(sseKey == null){
        	getMetadataRequest = new GetObjectMetadataRequest(bucketName, fileName);
        }else{
        	getMetadataRequest = new GetObjectMetadataRequest(bucketName, fileName).withSSECustomerKey(sseKey);
        }
        
        ObjectMetadata objectMetadata = s3client.getObjectMetadata(getMetadataRequest);
        System.out.println("对象大小: " + objectMetadata.getContentLength());
        System.out.println("元数据校验完成!");
    }

    private static void saveInputStream(S3ObjectInputStream inputStream, String fileName) {
        OutputStream os = null;
        try {
            String path = workspace;
            // 2、保存到临时文件
            // 1K的数据缓冲
            byte[] bs = new byte[1024];
            // 读取到的数据长度
            int len;
            // 输出的文件流保存到本地文件

            File tempFile = new File(path);
            if (!tempFile.exists()) {
            	System.out.println("Path is not exists!");
            }
            os = new FileOutputStream(tempFile.getPath() + File.separator + fileName);
            // 开始读取
            while ((len = inputStream.read(bs)) != -1) {
                os.write(bs, 0, len);
            }
            
            System.out.println("文件已保存到本地!");
            
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 完毕，关闭所有链接
            try {
                os.close();
                inputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
	 * 创建AES-256密钥
	 * @return SecretKey Object
	 */
	private static SecretKey generateSecretKey() {
        try {
        	SecretKey key = null;
        	System.out.println("生成AES256密钥");
            KeyGenerator generator = KeyGenerator.getInstance("AES");
            generator.init(256, new SecureRandom());
            key = generator.generateKey();
            System.out.println("AES256密钥已生成");
            System.out.println("AES256密钥："+Base64.encodeAsString(key.getEncoded()));
            return key;
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(-1);
            return null;
        }
    }
    
}

```


Done. :coffee: