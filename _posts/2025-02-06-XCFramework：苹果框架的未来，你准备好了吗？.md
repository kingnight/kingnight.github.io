---
title: XCFramework：苹果框架的未来，你准备好了吗？
tags:
  - ios
  - swift
  - swift package manager
  - xcframework
---

在这篇文章中，我们将深入探讨XCFramework的核心特点、集成方法、层级结构、创建过程以及最佳实践。无论您是经验丰富的开发者，还是刚刚踏入苹果开发世界的新手，这篇文章都将为您提供宝贵的指导和洞见。让我们一起揭开XCFramework的神秘面纱，探索它如何改变我们的开发方式，以及如何利用这一强大的工具来构建更加健壮和高效的应用程序。

# 简介

XCFramework是苹果推出的新的Framework格式，它实际上是另一种类型的Bundle，与原有的Framework很相似，但是可以包含多种体系机构和平台，这意味着你可以为苹果旗下所有操作系统（iOS，macOS，iPadOS等），不同体系结构（arm64，armv7 ，i386）包含模拟器，甚至UIKit和AppKit App 提供统一的分发方式。它在Xcode 11及更高版本中引入，旨在解决多平台、多架构环境下库文件分发和管理的挑战。

在这篇文章中，我们将深入探讨XCFramework的核心特点、集成方法、层级结构、创建过程以及最佳实践。无论您是经验丰富的开发者，还是刚刚踏入苹果开发世界的新手，这篇文章都将为您提供宝贵的指导和洞见。让我们一起揭开XCFramework的神秘面纱，探索它如何改变我们的开发方式，以及如何利用这一强大的工具来构建更加健壮和高效的应用程序。

## 特点

1. **跨平台支持**：XCFramework支持所有苹果平台和架构，包括iOS、macOS、tvOS和watchOS等。这意味着开发者只需要一个`.xcframework`文件，就可以同时支持多个平台的不同架构。

2. **架构管理**：XCFramework允许开发者按照平台划分文件，每个平台只包含该平台所需的架构文件。这简化了库文件的管理和分发过程，提高了编译和链接的效率。

3. **易于使用**：XCFramework的使用非常简单，只需要在Xcode项目中添加相应的`.xcframework`文件，然后按照需要进行配置即可。开发者无需编写额外的脚本或进行复杂的配置步骤。同时，它还支持在Swift Package Manager中接入，方便开发者进行依赖管理和版本控制。

##  集成

有两种方式可以集成XCFramework到项目中：

* 如果是一个简单项目，只需将生成的`xxx.xcframework`文件添加到Xcode项目中，具体来说，需要将XCFramework拖拽到Xcode中`Genernal` - `Frameworks, Libraries, and Embedded Content` 中即可，就可以像使用传统的`.framework`文件一样使用XCFramework了。

* 如果是复杂的大型项目，采用Swift Package方式接入是更加推荐的方式；具体来说，创建一个Package目录，包含Framework文件夹和Package.swifte配置文件，其中Framework文件夹存放具体的`xxx.xcframework`文件

![663f24e4af0949630e810a45dfab2448.png](/assets/images/663f24e4af0949630e810a45dfab2448-1.png)

Package.swifte配置文件模版如下，采用binaryTarget形式接入
```swift
// swift-tools-version:5.5
// The swift-tools-version declares the minimum version of Swift required to build this package.
 
import PackageDescription
 
let targets: [Target] = [
    .binaryTarget(name: "YYTextExt", path: "Framework/YYTextExt.xcframework")
]
 
let products: [Product] = [
    .library(name: "YYTextExt", targets: ["YYTextExt"])
]
 
let package = Package(
    name: "YYTextExt",
    products: products,
    dependencies: [],
    targets: targets
)
```
在Swift Package中，binaryTarget是一个用于定义二进制目标的功能，允许开发者将预编译的二进制文件（如XCFramework）作为包的一部分进行分发。这一特性从Xcode 12和Swift 5.3开始引入，旨在简化闭源库的使用和管理。

# 层级结构

XCFramework是一种容器格式，它包含了针对特定平台和架构编译的多个二进制文件，因此按平台和架构组合模式组织目录结构。我们以知名的第三方移动端数据库Realm为例，展示一个完整的XCFramework结构与层级；

<img src="/assets/images/截屏2024-10-29%2015.19.02-1.png" alt="截屏2024-10-29 15.19.02.png" style="zoom:50%;" />

以下是每个部分的作用：
* _CodeSignature：这个目录包含代码签名相关的文件，用于确保XCFramework的安全性和完整性，以及验证其来源。
* Info.plist：这是一个属性列表文件，包含了XCFramework的元数据，其中最重要的信息包括不同体系结构（arm64，armv7 ，i386）的包文件的位置路径、二进制文件路径、标识符。

![截屏2024-10-29 14.55.36.png](/assets/images/截屏2024-10-29%2014.55.36-1.png)
* 按不同系统和体系结构组合的目录，以ios-arm64为例，这个目录包含了为iOS平台的arm64架构编译的二进制文件和其他必要的资源。
	* RealmSwift.framework：这是一个框架目录，包含了RealmSwift框架的二进制文件。这是XCFramework的核心部分，包含了库的代码和资源。
	* Headers：这个目录包含了框架的头文件，这些头文件是框架对外提供的接口声明，开发者在集成框架时需要包含这些头文件。
	* RealmSwift-Swift.h：这是一个桥接头文件，用于Objective-C和Swift混合编程时，让Objective-C代码能够访问Swift定义的类和方法。
	* Modules：这个目录包含了模块映射文件（module.modulemap），它定义了模块的组织结构和对外暴露的接口。
	* RealmSwift.swiftmodule：这个文件包含了Swift模块的元数据，用于支持Swift语言的特性，如类型检查和代码补全。
	* PrivacyInfo.xcprivacy：这个文件包含了应用程序的隐私信息，用于描述应用程序如何使用用户数据。
	* Info.plist：属性列表文件，这个文件与外层的Info.plist都是必须存在文件，其中`Bundle name`, `Executable file`, `Bundle identifier` ,` Bundle version string (short)`,`MinimumOSVersion`，这些属性都是必须存在的字段，且其中前三项不能与项目中其他framework重复，在Xcode16之后版本中编译与提交App Store时都会被被检查，缺失会产生错误，稍后我们会提到。

![截屏2024-10-29 15.08.26.png](/assets/images/截屏2024-10-29%2015.08.26-1.png)

# 创建

创建XCFramework主要以下两种方式：

## 通过源码或静态库创建

> 注意：
>
> * 静态库指现有的static library（xxx.a）
>
> * 以下流程应该在Intel类型CPU设备上完成，如果在Apple 芯片设备上生成的缺少x86体系结构binary，如果不需要x86体系结构则可忽略此问题。

1. 创建target，选择类型Framework
![24866f22caae5defdcafce30068ee6d8.png](/assets/images/24866f22caae5defdcafce30068ee6d8-1.png)
2. 在项目中导入源码或static library文件
3. 修改Build Settings
	* Build Libraries for Disturibition 改为 yes
	* Skip Install 改为 no
	* Mach-O Type 用于指定编译完成的库，是动态链接还是静态连接，优先考虑设置为Static Library
4. 如果有Objc代码需要暴露出来，则需要考虑修改项目头文件
Build Phrases - Headers 根据需要决定哪些头文件需要移动到public ，暴露出来
![d96740bb7eee9a48cf4a48d35c3b19c1.png](/assets/images/d96740bb7eee9a48cf4a48d35c3b19c1-1.png)
5. 使用“xcode archive"为每个平台构建归档
	* 分别选择模拟器和真机单独编译，生成对应的framework文件
	* 生成路径位置位于/Users/xxx/Library/Developer/Xcode/DerivedData下
```swift
//项目xxxx
-Build
--Products
---Release-iphonesimulator
----xxxx.framework
---Release-iphoneos
----xxxx.framework
```

* 拷贝不同目录下xxxx.framework文件到同一个新建目录下，并重新命名为xxxx1.framework，xxxx2.framework

6. 生成xcframework

运行新的xcodebuild命令“xcodebuild -create-xcframework”把上一步生成的各个归档文件合并成一个

```
xcodebuild -create-xcframework -framework xxxx1.framework -framework xxxx2.framework -output xxxx.xcframework
```



显然，上面流程可以优化通过脚本生成

create-xcframework.sh

```shell
#!/bin/sh
PROJECT_NAME= XXXX  # 项目名称，需要修改

# 默认framework的名字是和项目名一样的，如果不一样的的话，需要此处单独设置。
framework_name=$PROJECT_NAME

# 导出xcframework的路径
output_path=XCFramework

# 模拟器的存档位置
simulator_archive_path=$output_path/simulator.xcarchive
# 真机的存档位置
iOS_device_archive_path=$output_path/iOS.xcarchive

# 删除旧版，然后创建新版
# 检查并创建输出目录
if [ -d "$output_path" ]; then
    echo "Removing existing directory: $output_path"
    rm -r $output_path
fi

mkdir $output_path || { echo "Error: Unable to create directory $output_path"; exit 1; }

# 打包模拟器
xcodebuild archive \
-scheme $framework_name \
-destination "generic/platform=iOS Simulator" \
-archivePath $simulator_archive_path \
BUILD_LIBRARY_FOR_DISTRIBUTION=YES \
SKIP_INSTALL=NO

echo "Simulator framework generated at: $simulator_archive_path"
# 检查模拟器框架是否存在
simulator_framework_path="$simulator_archive_path/Products/Library/Frameworks/$framework_name.framework"
if [ ! -d "$simulator_framework_path" ]; then
    echo "Error: Simulator framework not found!"
    exit 1
else
    echo "Simulator framework generated at: $simulator_archive_path"
fi

# 打包真机
xcodebuild archive \
-scheme $framework_name \
-destination "generic/platform=iOS" \
-archivePath $iOS_device_archive_path \
BUILD_LIBRARY_FOR_DISTRIBUTION=YES \
SKIP_INSTALL=NO


# 检查真机框架是否存在
ios_device_framework_path="$iOS_device_archive_path/Products/Library/Frameworks/$framework_name.framework"
if [ ! -d "$ios_device_framework_path" ]; then
    echo "Error: iOS device framework not found!"
    exit 1
else
    echo "iOS device framework generated at: $iOS_device_archive_path"
fi

# 创建 xcframework
xcodebuild -create-xcframework \
-framework $simulator_archive_path/Products/Library/Frameworks/$framework_name.framework \
-framework $iOS_device_archive_path/Products/Library/Frameworks/$framework_name.framework \
-output $output_path/$framework_name.xcframework

# 打包完成后，存档就失去作用，只作为中间打包过程使用。
rm -r $simulator_archive_path $iOS_device_archive_path

# 打开 XCFramework 所在的目录
open $output_path

```



## 通过Fat Framework创建

"Fat Framework"（也称为"universal framework"）是指一个包含了多个架构二进制文件的framework；

通常来说，通过Framework形式创建的二进制文件，都是默认采用动态库形式（Dynamic Library）;

构建一个fat framework通常涉及以下步骤：

1. **编译二进制文件**：为每个目标架构编译framework的代码。
2. **使用lipo合并**：使用`lipo`工具将不同架构的二进制文件合并成一个单一的二进制文件。
3. **创建framework结构**：将合并后的二进制文件放入framework的`Frameworks`目录中，并确保`Headers`和`Resources`等目录也包含在内。

本质上，Fat Framework是包含多个架构（如arm64和x86_64）的单一framework，因此通过转换可以成为XCFramework。

[xcframework-maker](https://github.com/darrarski/xcframework-maker) 工具可以将 Fat Framework 作为原料来转化为 XCFramework 文件。

使用步骤如下：

1. 安装xcframework-maker命令行

2. 生成Xcframework

```
xcframework-maker-main/.build/release/make-xcframework -ios ./xxx.framework -output ./ 
```



## 注意事项

下面列出各种可能出现的问题，并给出修改方案；

1. 导入xcframework，项目编译失败

module.modulemap位置应该位于framework内Modules文件夹下， 某些fat framework内部结构不标准，导致导入项目编译失败，所以需要把module.modulemap移动到正确位置

2. 导入xcframework，编译错误：`xxxx.framework/Versions/A: bundle format unrecognized, invalid, or unsuitable Command CodeSign failed with a nonzero exit code`

原因：错误格式，文件夹带软链接，包含Resources目录，但无文件，这种形式在Xcode15.0.1及以前版本可以编译通过，之后版本就会失败；

例如，以下图片所示结构就是一种错误形式：
<img src="/assets/images/d5e5b789c4b361f748a2bf45a5a65f4e-1.png" alt="d5e5b789c4b361f748a2bf45a5a65f4e.png" style="zoom:50%;" />

正确格式：
<img src="/assets/images/c19d0c3d429c9ce70149305435d62bf6-1.png" alt="c19d0c3d429c9ce70149305435d62bf6.png" style="zoom:50%;" />

3. 安装App时失败

以下错误情况在Xcode16之后版本会可能出现

```sh
无法安装“xxxx”
Domain: IXUserPresentableErrorDomain
Code: 1
Recovery Suggestion: Failed to load Info.plist from bundle at path /var/installd/Library/Caches/com.apple.mobile.installd.staging/temp.Zf08iP/extracted/xxxx.app/Frameworks/AMapSearchKit.framework; Extra info about "/var/installd/Library/Caches/com.apple.mobile.installd.staging/temp.Zf08iP/extracted/xxxx.app/Frameworks/AMapSearchKit.framework/Info.plist": Couldn't stat /var/installd/Library/Caches/com.apple.mobile.installd.staging/temp.Zf08iP/extracted/xxxx.app/Frameworks/AMapSearchKit.framework/Info.plist: No such file or directory
User Info: {
    DVTErrorCreationDateKey = "2024-10-25 02:21:38 +0000";
    IDERunOperationFailingWorker = IDEInstallCoreDeviceWorker;
}
```

此示例中提到的MapSearchKit是通过Fat Fraemwork转成XCFramework的，由于MapSearchKit内部结构不标准或者是使用较老Xcode编译，其中的Info.plist缺少`Bundle name`,`Executable file`,`Bundle identifier`等必须存在的字段，Xcode编译可以通过，但是安装App时，问题出现。

4.  xxxx framework does not support the minimum OS Version specified in the Info.plist

在提交App Store时，如果出现如下错误：
```
NSUnderlyingError = "Error Domain=IrisAPI Code=-19241 \"Asset validation failed\" UserInfo={status=409, detail=Invalid Bundle. The bundle xxxx.app/Frameworks/xxxx.framework does not support the minimum OS Version specified in the Info.plist., id=f6e7b58e-6edd-49dc-ba55-a9524c53a7d6, code=STATE_ERROR.VALIDATION_ERROR.90208, title=Asset validation failed, NSLocalizedFailureReason=Invalid Bundle. The bundle xxxx.app/Frameworks/xxxx.framework does not support the minimum OS Version specified in the Info.plist., NSLocalizedDescription=Asset validation failed}";
```

首先framework内层的 Info.plist属性列表文件不能缺少`MinimumOSVersion`字段，另外参考
>https://forums.developer.apple.com/forums/thread/748173
> * The similar issue we facing is connected with static frameworks. The SPM in XCode 15.3 embedding static frameworks and causing validation issue.
> * The workaround suggested by Apple is to set minimum OS version for that frameworks higher than/ or equal to application minimum OS version.
>* Another workaround is to add run script build phase in application target to delete the static frameworks from .app/Frameworks like following:
> * `rm -rf "${TARGET_BUILD_DIR}/${TARGET_NAME}.app/Frameworks/XXXX.framework`

这段话描述了这个问题与静态框架（static frameworks）相关。Swift Package Manager（SPM）嵌入静态框架时会导致验证问题。

1. **苹果建议的解决方案**：

   - 苹果建议将这些框架的最低操作系统版本设置为高于或等于应用程序的最低操作系统版本。这意味着，如果你的应用程序支持iOS 14，那么这些静态框架也应该支持iOS 14或更高版本。

2. **另一种解决方案**：

   - 另一种解决方法是在应用程序的目标中添加一个运行脚本构建阶段（run script build phase），以从.app/Frameworks目录中删除静态框架。具体操作如下：
     ```
     rm -rf "${TARGET_BUILD_DIR}/${TARGET_NAME}.app/Frameworks/XXXX.framework"
     ```
     这里的`XXXX.framework`应该替换为你想要删除的静态框架的实际名称。`TARGET_BUILD_DIR`和`TARGET_NAME`是Xcode构建过程中使用的变量，分别代表构建目录和目标应用程序的名称。


5. CFBundleShortVersionString 格式不正确
```
 Asset validation failed (90060) This bundle is invalid. The value for key CFBundleShortVersionString 'SA_6.2.0' in the Info.plist file at 'Payload/xxxx.app/Frameworks/OAuth.framework' must be a period-separated list of at most three non-negative integers. Please find more information about CFBundleShortVersionString at https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleshortversionstring (ID: 79a102f6-0f11-4b68-9e0b-fb15e755b56f)
```

参考[官方说明](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleshortversionstring)
这个键是一个用户可见的字符串，表示bundle的版本。所需的格式是三个句点分隔的整数，例如10.14.1。字符串只能包含数字字符（0 ~ 9）和句点。而错误示例中，SA_6.2.0 包含了字符，这在Xcode16之后都会被认为错误，无法提交App Store

6.CFBundleIdentifier  存在不合法字符

```
Asset validation failed (90049) This bundle is invalid. The bundle at path Payload/sohuhy.app/Frameworks/Weibo_SDK.framework has an invalid CFBundleIdentifier 'org.cocoapods.Weibo_SDK' There are invalid characters(characters that are not dots, hyphen and alphanumerics) that have been replaced with their code point 'org.cocoapods.Weibo\u005fSDK' CFBundleIdentifier must be present, must contain only alphanumerics, dots, hyphens and must not end with a dot. [see the Core Foundation Keys at https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/TP40009249-102070-TPXREF105] (ID: fa03fa9a-7246-4bf6-afb7-66e95614582d)
```

CFBundleIdentifier只能包含字母，点（`.`），连字符（`-`）

# 依赖关系处理

在一些复杂情况下，不同的Frameowork之间存在依赖关系，需要配合协同使用，下面介绍各种情况及处理策略；

## 采用依赖项传递解决依赖关系

假设存在两个XCFrameworkA，XCFrameworkB，如果XCFrameworkA中源码需要依赖另一个XCFrameworkB中功能，**将 XCFrameworkB 作为依赖项传递**解决这个问题；

这种方法将 xcframeworkB 的依赖关系传递给最终使用 xcframeworkA 的应用程序。

1. **创建 xcframeworkA 时，不要嵌入 xcframeworkB:** 在构建 xcframeworkA 时，不要将 xcframeworkB 嵌入其中。只需在 xcframeworkA 的 target 的 "Frameworks and Libraries" 中链接 xcframeworkB，并在代码中 `import FrameworkB`。
2. **在使用 xcframeworkA 的 App 项目中，同时添加 xcframeworkA 和 xcframeworkB:** 将 xcframeworkA 和 xcframeworkB 都拖拽到 App 项目的项目导航器中。
3. **链接 xcframeworkA 和 xcframeworkB:** 确保 App target 的 "Frameworks and Libraries" 中同时链接了 xcframeworkA 和 xcframeworkB。
4. **在 App 代码中导入 xcframeworkA:** 在需要使用 xcframeworkA 功能的 Swift 文件中，使用 `import FrameworkA` 导入 xcframeworkA。 无需显式导入 xcframeworkB，因为 xcframeworkA 已经隐式地依赖于它。

## 使用Swift Package Manager (SPM)管理依赖
Swift Package Manager是苹果官方提供的用于管理Swift项目依赖关系和构建过程的工具。通过SPM，你可以定义一个Package.swift文件，其中列出了项目的依赖项，包括XCFramework。例如，你可以在Package.swift中添加如下依赖配置：
```swift
import PackageDescription

let package = Package(
    name: "MyLibrary",
    dependencies: [
        .package(name: "MySDK", url: "https://example.com/MySDK.xcframework.zip", checksum: "sha256 checksum here")
    ],
    targets: [
        .target(
            name: "MyLibraryTarget",
            dependencies: ["MySDK"]),
    ]
)
```
这种方式可以确保XCFramework及其依赖项被正确解析和集成到项目中。

此外，**使用二进制目标接入依赖项也是推荐方式**： 在应用程序的`Package.swift`文件中，可以将XCFramework导入为二进制目标，这样应用程序直接使用XCFramework的二进制代码，而不会导入框架的依赖项。这可以通过如下方式实现：

```swift
import PackageDescription

let package = Package(
    name: "ExampleApp",
    dependencies: [
        .binaryTarget(name: "ExampleSDK", path: "ExampleSDK.xcframework")
    ]
)
```

还有一些特殊情况需要考虑，我们以Swift Package依赖CryptoSwift为例说明：


* 如果项目中有且仅有`A Package`依赖CryptoSwift，推荐直接在Package.swift中通过binaryTarget添加xcframework

```swift
.binaryTarget(name: "CryptoSwift", path: "../../ibrary/CryptoSwift.xcframework"),
```

* 如果`A` Package依赖CryptoSwift，且`B` Package也依赖CryptoSwift，但是A与B之间没有任何依赖关系，如果在A与B package中都各自加入CryptoSwift，各自Package独立编译正确，但是两个Package集成到同一个项目中，就会遇到错误
```swift
multiple packages ('A', 'B') declare targets with a conflicting name: 'CryptoSwift’; target names need to be unique across the package graph
```
此时，解决方案是创建一个新的CryptoSwift Package包裹binaryTarget
```swift
import PackageDescription

let targets: [Target] = [
	.binaryTarget(name: "CryptoSwift", path: "Framework/CryptoSwift.xcframework")
]

let products: [Product] = [
    .library(name: "CryptoSwift", targets: ["CryptoSwift"])
]

let package = Package(
    name: "CryptoSwift",
    products: products,
    dependencies: [],
    targets: targets
)
```
然后在`A`和`B` Package，都添加dependencies
```swift
    dependencies: [
        // Dependencies declare other packages that this package depends on.
        .package(name: "CryptoSwift", path: "../Product/CryptoSwift"),
    ],
```


* 如果`A `Package依赖CryptoSwift，且`B` Package也依赖CryptoSwift，同时`B`也依赖`A` Package，则只需要在`A` Package中添加binaryTarget，`B`中无需添加
```swift
.binaryTarget(name: "CryptoSwift", path: "../../ibrary/CryptoSwift.xcframework"),
```




## **隐藏依赖项**

 如果你的XCFramework依赖于其他框架，可以使用`@_implementationOnly`属性在XCFramework的根模块中隐藏对依赖项的导入。这将使依赖项仅在框架内部使用，而不会暴露给使用框架的应用程序，从而避免符号重复定义的问题。

```swift
import Foundation
@_implementationOnly import cmark_gfm
@_implementationOnly import cmark_gfm_extensions
```

Package.swift配置中指明依赖

```swift
// swift-tools-version:5.6

import PackageDescription

let package = Package(
  name: "swift-markdown-ui",
  platforms: [
    .macOS(.v12),
    .iOS(.v15),
    .tvOS(.v15),
    .macCatalyst(.v15),
    .watchOS(.v8),
  ],
  products: [
    .library(
      name: "MarkdownUI",
      targets: ["MarkdownUI"]
    )
  ],
  dependencies: [
    .package(url: "https://github.com/gonzalezreal/NetworkImage", from: "6.0.0"),
    .package(url: "https://github.com/pointfreeco/swift-snapshot-testing", from: "1.10.0"),
    .package(url: "https://github.com/swiftlang/swift-cmark", from: "0.4.0"),
  ],
  targets: [
    .target(
      name: "MarkdownUI",
      dependencies: [
        .product(name: "cmark-gfm", package: "swift-cmark"),
        .product(name: "cmark-gfm-extensions", package: "swift-cmark"),
        .product(name: "NetworkImage", package: "NetworkImage"),
      ]
    ),
    .testTarget(
      name: "MarkdownUITests",
      dependencies: [
        "MarkdownUI",
        .product(name: "SnapshotTesting", package: "swift-snapshot-testing"),
      ],
      exclude: ["__Snapshots__"]
    ),
  ]
)
```

## **使用Cocoapods和SPM分发XCFramework**

 如果项目中同时使用了Cocoapods和SPM，可以创建一个XCFramework，并将其通过Cocoapods和SPM进行分发。这需要在Cocoapods的`podspec`文件中设置依赖项，并在SPM的`Package.swift`中添加相应的二进制目标。

但是我们强烈不推荐使用这种方案，在依赖关系复杂情况下，可能会出现很多异常现象。

# 资源

# 最佳实践

使用XCFramework时，遵循以下最佳实践可以提高开发效率并确保框架的稳定性和兼容性：

1. **版本控制**：确保每个XCFramework都有明确的版本号，这有助于管理和更新框架。

2. **自动化构建**：利用CI/CD工具自动化XCFramework的构建和发布流程，这样可以确保每次代码更改后都能自动生成新的框架版本。

3. **文档完善**：为每个XCFramework提供详细的文档，包括使用方法和依赖关系，这有助于其他开发者理解和使用你的框架。

4. **兼容性测试**：在不同的平台和架构上测试XCFramework，确保它在所有目标环境中都能正常工作。

5. **调试和符号支持**：在构建XCFramework时包含调试符号，这有助于在出现问题时进行调试和错误分析。

在代码层面，也有一些推荐:

1. 使用条件编译指令导入：`#if canImport()`

`#if canImport()`它是一个条件编译指令，允许您根据特定模块是否可以导入到当前编译上下文中来有条件地包含或排除代码。

```swift
#if canImport(FrameworkA)
import FrameworkA
  // Your code that depends on Foundation framework
#else
  // Fallback or alternative code for platforms without Foundation framework
  throw ModuleFrameworkError
#endif
```

2. 使用objc_getClass

在Swift中，objc_getClass是一个由Objective-C运行时提供的函数，它允许你通过类名获得对类的引用。
这个功能对于在运行时验证框架中特定类的存在性非常方便。

```swift
#if canImport(FrameworkA)
    if objc_getClass("FrameworkA.FrameworkAFunc") != nil {
        FrameworkAFunc.manager.configure(deeplink: deeplink)
    }
#else
    throw ModuleConfigError
#endif
```

通过遵循这些最佳实践，你可以确保XCFramework的质量和可用性，同时提高开发效率和项目的可维护性。

# 总结

XCFramework作为一种新型的跨平台二进制库分发解决方案，为iOS开发者带来了很多便利。它支持多平台、多架构的库文件分发和管理，提供了更好的架构管理功能，并简化了库文件的使用过程。随着苹果对XCFramework的支持不断完善和普及，相信它将在未来的iOS开发中发挥越来越重要的作用。

# 参考

* https://developer.apple.com/documentation/xcode/creating-a-multi-platform-binary-framework-bundle
* https://developer.apple.com/documentation/xcode/distributing-binary-frameworks-as-swift-packages
* https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md
