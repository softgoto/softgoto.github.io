---
title: iOS 如何封装 SDK 提供给第三方开发者
date: 2016-03-17 12:19:27
categories: 技术类
tags:
- iOS
- Xcode
- SDK
---

需求：我们需要将一些功能封装并提供给第三方开发者使用，也就是我们常说的第三方 SDK 

思路：`.a` 和 `Framework`

先介绍一下静态库和动态库，静态库和动态库是相对编译期和运行期的，静态库以 `.a` 和 `.framework` 形式存在，链接时静态库会被完整地复制到可执行文件中，被多次使用就有多份冗余拷贝；动态库以 `.dylib` 和 `.framework` 形式存在，链接时不复制，程序运行时由系统动态加载到内存供程序调用，系统只加载一次，多个程序共用节省内存，一般只能由系统创建。framework 为什么既是静态库又是动态库呢？这是因为系统的 `.framework` 是动态库，我们自己建立的 `.framework` 是静态库。

<!--more-->

Framework 文件创建步骤：（iOS8 以上才能使用）这里我使用的是 Xcode7.2.1，代码我放在这里，有兴趣的可以参考一下。

[https://github.com/softgoto/XHFramework](https://github.com/softgoto/XHFramework)
[https://github.com/softgoto/XHFrameworkUserDemo](https://github.com/softgoto/XHFrameworkUserDemo)

1、新建Cocoa Touch Framework Project 

![](https://i.loli.net/2020/06/12/RDsJcPq7WAlKrGh.png)

2、命名 XHFramework

![](https://i.loli.net/2020/06/12/V5inMmD7ZbwSagd.png)

3、导入组件代码 [https://github.com/softgoto/RWKnobControl](https://github.com/softgoto/RWKnobControl)

![](https://i.loli.net/2020/06/12/6J7EBie8qzrY1ka.png)

4、选择 Build Phases

![](https://i.loli.net/2020/06/12/MvBwdmCGIZqYaLy.png)

5、在 Headers 中有 Public、Private、Projec 三块，Public 里的头文件是暴露给第三方的，Project 中的则只有当前项目可看，对第三方是隐藏的。

6、这里我们把需要暴露给第三方调用的头文件拖到 Public 中

![](https://i.loli.net/2020/06/12/WxBALZsYGveJkh9.png)

7、打包

先选择 `iOS Device` 再按快捷键 `Command + B`

![](https://i.loli.net/2020/06/12/DLpawFsAb62HSJy.png)

然后选择模拟器按快捷键 `Command + B`

![](https://i.loli.net/2020/06/12/FzxvnroqlALDweg.png)

8、可以看到 Products 下的 `.framework` 文件已经由红色变成了黑色，说明 `.framework` 文件已经创建成功。选中文件点击右键 `Show in Finder`

![](https://i.loli.net/2020/06/12/MHCipmLwN1D98JI.png)

9、发现 Debug-iphoneos 和 Debug-iphonesimulator 目录下都有 `XHFramework.framework` 文件，这里我们需要将两个文件合并来兼容真机和模拟器调试使用。

10、在合并之前我们先检查生成的文件所支持的框架类型

11、先选中真机下的 framework 文件，可以看到里面有一个和 `XHFramework.framework` 同名但是没有后缀的文件（应该就是 `.a` 文件），打开终端输入：

```shell
$ lipo -info 文件路径
```

![](https://i.loli.net/2020/06/12/aMCLKnIY9ARED8h.png)

然后选择模拟器下的 framework 文件，在终端中输入同上命令

![](https://i.loli.net/2020/06/12/C1DuJBGvH46EyZS.png)

12、合并（命令：lipo -create A文件 B文件 -output C输出路径）

![](https://i.loli.net/2020/06/12/ZXdz2wYHgI5RWBr.png)

13、选择一个 framework 复制出来，然后打开将新生成的文件替换

![](https://i.loli.net/2020/06/12/CjhXRqGKvnOW1QL.png)

14、大功告成

15、Framework 的使用，创建一个 `Single View Application` ，命名 XHFrameworkUserDemo

16、将 `XHFramework.framework` 拖入到新创建的项目中

![](https://i.loli.net/2020/06/12/yzOXYcMBALab5xD.png)

17、在 `ViewController.m` 中引入头文件

![](https://i.loli.net/2020/06/12/WFLXn2CrB7f4pmd.png)

18、Run，发现报错

![](https://i.loli.net/2020/06/12/WeAZp8GNEMOvUIz.png)

19、解决方案：选择 `Build Phases`，点开 `Copy Files` ，如果没有需要添加；点击加号将 `XHFramework.framework` 文件添加到 `Copy Files` 中，并将 `Destination` 改为 `Frameworks`

![](https://i.loli.net/2020/06/12/rZjH8toeJYOhDPw.png)

20、然后一顿快捷操作 `Command + B` `Command + R` 真机测试成功

![](https://i.loli.net/2020/06/12/hLF2XM9oIvngj3N.png)

常见问题：

**Q、只有一种模拟器设备能运行，其他模拟器报错**
A、兼容全部设备解决方法：改下静态库的兼容属性。`Target：-> Build Settings -> Architectures -> Build Active Architecture Only` 全改成 `NO`；`Build Active Architecture Only` 这个属性设置为 `yes`，是为了 DEBUG 的时候编译速度更快，它只编译当前的 architecture 版本，所以会报错编译不到文件，出错 `"_OBJC_CLASS_$_xxxxxx", referenced from:` 而设置为 `no` 时，会编译所有的版本，这样就解决编译出错的问题了。
下面是设备对应的 architecture：
armv6：iPhone 2G/3G，iPod 1G/2G
armv7：iPhone 3GS/4/4s，iPod 3G/4G，iPad 1G/2G/3G
armv7s：iPhone5, iPod5
arm64：iPhone5s
编译出的版本是向下兼容的，比如你设置此值为 `yes`，用 iPhone 4 编译出来的是 `armv7` 版本的，iPhone 5 也可以运行，但是 `armv6` 的设备就不能运行。

**Q、warning: embedded dylibs/frameworks only run on iOS 8 or later**
A、如 framework 设置支持 iOS 7 ，Build 时会发现有 `ld: warning: embedded dylibs/frameworks only run on iOS 8 or later` 警告，这是因为工程默认编译设置的是 `Dynamic Framework`。这种编译只有在 iOS 8 以后才能使用。动态库可以分开发布，在运行时查找并存入内存，但苹果只允许他自己用，到 iOS 8 以后才开放给开发者。因此，我需要将 `Dynamic Framework` 更换为 `Static Library` 静态模式。设置路径为 `Build Settings -> Linking -> Mach-O Type -> Static Library`

**Q、framework 中使用 Category**
A、如果你用了 Category，别人在用你的 framework 时会发生崩溃。这时别人在引用时需要在工程中 `other linker flags` 中添加 `-objC` 如果依然有问题，再添加 `-all_load`。

**Q、编译成功，但发现很多关于符号表的警告**
A、需要将 `Generate Debug Symbols` 设置为 NO 即可关闭符号表警告。

**Q、支持 bitcode**
A、在 TAGETS 的 Build setting 中搜索 `Other C Flags`，添加命令 `-fembed-bitcode`。同样的设置在 PROJECT 中。如果不进行以上操作。别人在集成你的 framework 时可以编译，亦可以真机测试。唯独在打包时会发出警告并打包失败。警告为 framework 不支持 bitcode！


Done :coffee: