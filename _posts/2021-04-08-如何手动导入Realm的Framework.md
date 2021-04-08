---
title: "如何手动导入Realm的Framework"
description: "如何手动导入Realm的Framework"
category: programming
tags: iOS,Swift,Realm
---

有些特殊情况下，你可能不想使用SwiftPM, CocoaPods或者Carthage的方式安装Realm，那么这篇我将介绍如何手动导入；我以最新的5.X系列版本为例说明；

# 下载
从[Realm的Github Release](https://github.com/realm/realm-cocoa/releases/tag/v5.5.1)的找下载链接，有Swift和OC两个版本
* realm-objc-5.5.1.zip
* realm-swift-5.5.1.zip

这里我选择的Swift版本，下载解压后，进入到ios目录下

![2d777cef8166f6993a0f112fb791f1fc](/assets/images/截屏2021-04-08 下午3.27.09.png)

你会看到有三个版本，这里需要注意，需要选择和你Xcode版本匹配的版本，我在使用Xcode12.2版本，所以我只能选`swift-12.4`版本，而不能选低于的`swift-11.7`，否则会提示Swift的版本是4.x而无法编译通过。

# 导入

在你的项目中的
![2cf328cda9de2e81194aa6e7e642f5f4](/assets/images/截屏2021-04-08 下午3.31.30.png)
导入
![9f67fcdedf27e765cede32f5feb54f02](/assets/images/截屏2021-04-08 下午3.31.14.png)
**需要特别注意，右侧的配置选择“Embed & Sign”，否则启动运行时，会提示找不到动态库**

对于一个xxx.framework，它本身是一个文件夹，内部包含一个可执行文件，我们可以通过file命令来查看是动态库还是静态库；

```
% file Realm
Realm: Mach-O universal binary with 4 architectures: [x86_64:Mach-O 64-bit dynamically linked shared library x86_64] [arm_v7] [i386] [arm64]
Realm (for architecture x86_64):	Mach-O 64-bit dynamically linked shared library x86_64
Realm (for architecture armv7):	Mach-O dynamically linked shared library arm_v7
Realm (for architecture i386):	Mach-O dynamically linked shared library i386
Realm (for architecture arm64):	Mach-O 64-bit dynamically linked shared library arm64
```
你会看到多个`dynamically linked shared library`表明是**动态库**，如果是`current ar archive`则是**静态库**



# 添加脚本

到此为止，如果在Debug环境下，运行是正常的；但是如果打Release包，会出出现问题`IPA processing failed`

![bda4ed30cc672aa90ab7388ed6bea8b4](/assets/images/1683753-3ba89c5450837219.png)

查看Show logs中，第二个标准log

![1a02ec074236f3285bed230fa2edbae0](/assets/images/log.png)

可以找到出错原因是：
```swift
Assertion failed: Expected 4 archs in otool output:
/var/folders/db/t072jzzd7673f6v_2qdw80540000gn/T/IDEDistributionOptionThinning.~~~u6iAcS/Payload/sohuhy.app/Frameworks/Realm.framework/Realm:
```

我们之前在使用file查看Realm时就已经发现，其中包含了 [x86_64] [arm_v7] [i386] [arm64] 四种架构，但是Release上架时，我们是不需要其中的某些架构的，所以需要移除。

如果你在网络上搜索这个问题的解决方案，很多建议使用lipo移除不需要的架构。但是这样修改之后，模拟器就不能正常运行了，所以如果需要运行模拟器，就需要替换回原来的SDK，并不方便；

如果你仔细查看Realm.framework中默认已经提供了一个脚本`strip-frameworks.sh`，供你解决这个问题；

你需要在`Build Phases`中`Embed Frameworks`的下一步，添加执行脚本Run Script，为了方便区分我把它改名为Realm Strip Script；

常规思路，添加
```
"${PROJECT_DIR}/Vendors/Realm.framework/strip-frameworks.sh"
```
这样修改之后，运行，会执行出错
```
Permission Denied
```
如果你是在本机运行代码打Release，可以找到`strip-frameworks.sh`所在路径，执行
```
chmod 777 strip-frameworks.sh
```
是可以解决上面问题的，但是如果提交到Git，在远端Jenkins这类CI继承工具中执行，仍然会出现`Permission Denied`问题；

最后的解决方式：需要使用环境变量在编译后的目录执行
```
chmod +x "${BUILT_PRODUCTS_DIR}/${FRAMEWORKS_FOLDER_PATH}/Realm.framework/strip-frameworks.sh"
bash "${BUILT_PRODUCTS_DIR}/${FRAMEWORKS_FOLDER_PATH}/Realm.framework/strip-frameworks.sh"
```
最后，这段shell脚本只需在Release下执行
```
if [ "${CONFIGURATION}" = "Release" ]; then
chmod +x "${BUILT_PRODUCTS_DIR}/${FRAMEWORKS_FOLDER_PATH}/Realm.framework/strip-frameworks.sh";
bash "${BUILT_PRODUCTS_DIR}/${FRAMEWORKS_FOLDER_PATH}/Realm.framework/strip-frameworks.sh"
fi

```

# 测试

如果你的Test工程也许要引用`Realm`，需要在测试工程的Link Binary With Libraries中添加`Realm.framework`和`RealmSwift.framework`


# 参考

* https://stackoverflow.com/questions/21115430/jenkins-building-xcode-getting-build-error-permission-denied
* https://github.com/realm/realm-cocoa/issues/2070
* https://stackoverflow.com/questions/9850936/what-permissions-are-required-for-run-script-during-a-build-phase
* https://stackoverflow.com/questions/3614017/how-can-i-limit-a-run-script-build-phase-to-my-release-configuration


