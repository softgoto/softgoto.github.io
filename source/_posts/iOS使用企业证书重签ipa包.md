---
title: iOS使用企业证书重签ipa包
date: 2016-04-12 15:05:09
categories: 技术类
tags:
- iOS
---

iOS9 升级后众多企业分发的 APP 已经出现了不能安装的情况，而 iOS 8 或更早的系统不受影响。那是因为从 iOS 9 以后，系统会在 ipa 包下载完之后，拿 ipa 包中的 `bundle ID` 与 `manifest.plist` 中的 `bundle ID` 进行比对，不一致不允许安装。

所以现在重签时需要修改 ID，重签需要用到一个重签工具，我们先下载开源重签工具 [iReSign](https://github.com/maciekish/iReSign)，打开 `iReSign.app` 点击 `重新签名!` 按钮，成功后就会在原 ipa 包同目录下生成一个重签后的 ipa 包。建议将需要用到的 ipa、plist、profile 文件放在同一个文件夹内

<!--more-->

### 创建 App ID

登录[苹果开发者中心](https://developer.apple.com/membercenter/index.action)，创建 `App ID`

![](https://i.loli.net/2020/06/12/x5rCZ1FtSOcLA64.jpg)

### 创建 Profile 文件

为上面创建的 `App ID` 创建一个 `iOS Provisioning Profiles(Distribution)` 并下载到本地（可以通过 Xcode 来创建）

![](https://i.loli.net/2020/06/12/yaOHgrJ6fu5hPdl.jpg)

### 新建 entitlements.plist 文件

创建 `entitlements.plist` 文件，内容如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>application-identifier</key>
	<string>PREFIX.yourappBundleID</string>
	<key>aps-environment</key>
	<string>production</string>
	<key>get-task-allow</key>
	<false/>
	<key>keychain-access-groups</key>
	<array>
		<string>PREFIX.yourappBundleID</string>
	</array>
</dict>
</plist>
```

将其中的 `PREFIX.yourappBundleID` 替换为第一步截图中的 `Prefix` 和 `ID`。该文件中内容配置可以参考[代码签名探析](https://objccn.io/issue-17-2/)，比如没有推送功能的话，就去掉如下两行：

```xml
<key>aps-environment</key>
<string>production</string>
```

### 重签

打开`iReSign`

![](https://i.loli.net/2020/06/12/juRE7sTUXt8I2QG.jpg)


* 需要重签 ipa 的存放路径
* 在第 2 步中下载的 `Provisioning Profiles` 文件存放路径
* 在第 3 步中创建的 `entitlements.plist` 文件存放路径
* 勾选 `修改ID`，然后填写第 1 步中的 `ID`
* 选择重签使用的证书

点击 `重新签名!`，如果没有报错，恭喜你，重签成功！如果失败请仔细检查 `App ID`、`Profile`、`plist` 文件、证书信息是否正确。

### 验证

找个 HTTPS 环境通过苹果的 `itms-services` 协议来安装，`manifest.plist` 文件的参考配置

```xml
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- array of downloads. -->
  <key>items</key>
  <array>
      <dict>
          <!-- an array of assets to download -->
          <key>assets</key>
          <array>
              <!-- software-package: the ipa to install. -->
              <dict>
                  <!-- required.  the asset kind. -->
                  <key>kind</key>
                  <string>software-package</string>
                  <!-- optional.  md5 every n bytes.  -->
                  <!-- will restart a chunk if md5 fails. -->
                  <key>md5-size</key>
                  <integer>10485760</integer>
                  <!-- optional.  array of md5 hashes -->
                  <key>md5s</key>
                  <array>
                      <string>41fa64bb7a7cae5a46bfb45821ac8bba</string>
                      <string>51fa64bb7a7cae5a46bfb45821ac8bba</string>
                  </array>
                  <!-- required.  the URL of the file to download. -->
                  <key>url</key>
                  <string>http://www.example.com/apps/foo.ipa</string>
              </dict>
              <!-- display-image: the icon to display during download. -->
              <dict>
                  <key>kind</key>
                  <string>display-image</string>
                  <!-- optional. icon needs shine effect applied. -->
                  <key>needs-shine</key>
                  <true/>
                  <key>url</key>
                  <string>http://www.example.com/image.57×57.png</string>
              </dict>
              <!-- full-size-image: the large 512×512 icon used by iTunes. -->
              <dict>
                  <key>kind</key>
                  <string>full-size-image</string>
                  <!-- optional.  one md5 hash for the entire file. -->
                  <key>md5</key>
                  <string>61fa64bb7a7cae5a46bfb45821ac8bba</string>
                  <key>needs-shine</key>
                  <true/>
                  <key>url</key>
                  <string>http://www.example.com/image.512×512.jpg</string>
              </dict>
          </array><key>metadata</key>
          <dict>
              <!-- required -->
              <key>bundle-identifier</key>
              <string>com.example.fooapp</string>
              <!-- optional (software only) -->
              <key>bundle-version</key>
              <string>1.0</string>
              <!-- required.  the download kind. -->
              <key>kind</key>
              <string>software</string>
              <!-- optional. displayed during download; -->
              <!-- typically company name -->
              <key>subtitle</key>
              <string>Apple</string>
              <!-- required.  the title to display during the download. -->
              <key>title</key>
              <string>Example Corporate App</string>
          </dict>
      </dict>
  </array>
</dict>
</plist>
```


Done. :coffee: