---
title: "Swift Package Manager工程实践"
description: "Swift Package Manager工程实践"
category: programming
tags: Apple,iOS,Swift Package Manager,SwiftPM,SPM
---

`Swift Package Manager`（下文简称 `SwiftPM` ）是苹果官方提供的一个用于管理源代码分发的工具，它与Swift构建系统集成在一起，可以自动执行依赖项的下载，编译和链接过程。该工具可以帮助我们编译和链接 Swift packages（包），管理依赖关系、版本控制，以及支持灵活分发和协作（公开、私有、团队共享）等。支持Swift、Objective-C、Objective-C ++、C或C ++。 `SwiftPM`包管理器支持 macOS 和 Linux，与CocoaPods和Carthage功能类似，不过比这两个更简洁，代码的侵入性更小，也不需要额外安装工具。

本文将详细介绍团队在引入`SwiftPM`进行工程实践中，探索和累积的相关知识和实践经验，我们将从结构设计、资源处理、链接方式的选择、编译与链接参数设置、异常处理，这五个方面展开详细介绍，每个小部分结尾都提供了最佳实践的总结；希望能够帮助其他想要尝试`SwiftPM`的开发者顺利过渡；

本文面向了解`SwiftPM`基本知识，但是没有深度使用的开发者；如果你还不了解`SwiftPM`，建议首先阅读WWDC相关视频；

# 结构设计

梳理清楚代码之间的依赖关系，对于设计`SwiftPM`中，模块如何组成和进行合理拆分，非常重要！

## 代码组织方式

* 建议从两个维度考虑组件的组织方式，组件的性质：基础组件，业务组件；代码的依赖关系：通用组件，专用组件；
* 优先统一代码语言，把Objc代码全部转成Swift后，会很方便整合到`SwiftPM`中；
* 如果Objc代码过多，或者这部分OBjc代码是一个完整的功能模块，即不存在Swift/Objc互相依赖混编的情况，也可以拆分到一个`SwiftPM`中；
* 如果一个模块依赖另一个模块中的极少数代码，可以考虑复制所需代码到本模块，并标记为非`public`，解除模块之间耦合关系；

## 目录结构

推荐使用命令行方式创建`SwiftPM`；注意命名时尽可能与模块功能相符，不要包含Lib、Framework等不能体现功能的描述；

### 创建SwiftPM
```swift
 $ mkdir MyPackage
 $ cd MyPackage
 $ swift package init # or swift package init --type library
```
默认创建的`SwiftPM`项目的名字就是文件夹的名字；这将创建带有目标target和相应测试target的库，包含所需的目录结构和单元测试，如下面所示：

```swift
Tests           //单元测试
--MyPackageTests
-----MyPackageTests.swift
Sources
--MyPackage     //同顶层目录名的target目录
-----MyPackage.swift
README.md       //文档  
Package.swift   //配置文件，类似Cocoapods自定义pod库的podspec文件
```

### 组织结构与路径设置

生成的`Package.swift`如下面所示：
```swift
let package = Package(
     name: "MyPackage",
     products: [
         // Products define the executables and libraries a package produces, and make them visible to other packages.
         .library(
             name: "MyPackage",
             targets: ["MyPackage"]),
     ],
     dependencies: [
         // Dependencies declare other packages that this package depends on.
         // .package(url: /* package url */, from: "1.0.0"),
     ],
     targets: [
         // Targets are the basic building blocks of a package. A target can define a module or a test suite.
         // Targets can depend on other targets in this package, and on products in packages this package depends on.
         .target(             //单独一个Target
             name: "MyPackage",
             dependencies: []),  //此处没有设置path，默认查找Sources下MyPackage目录
         .testTarget(
             name: "MyPackageTests",
             dependencies: ["MyPackage"]),
     ]
 )
```

由于一个`SwiftPM`库包可以包含多个`target`，所以`Sources/Tests`目录下存放源代码的目录结构组织方式，有不同的选择；有两种方式可供参考：

1. **单Target模式**，默认情况下，SwiftPM自动创建生成对应的target目录，如当前目录结构所示，查找Sources下MyPackage目录，**`Sources`目录下仅有一个目录`MyPackage`，且该目录名与target的name参数保持一致**；当需要通过多个目录结构拆分代码时，只能在`MyPackage`下创建子目录，不能与`MyPackage`同级；
2. **多Target模式**，在`Sources`目录下，创建多个平级目录，如targetA，targetB，同时在`Package.swift`中targets数组中对应进行配置，必要时指明target的path路径；


接下来我们通过一个示例，展示多Target模式，目录结构如下所示，
```swift
//...
Sources
--MyPackage     //MyPackage目录
-----MyPackage.swift
--TargetA       //TargetA目录
-----TargetA.swift
--TargetB       //TargetB目录
-----TargetB.swift
//...
```

多Target模式`Package.swift`示例代码如下所示，

```swift
let package = Package(
    name: "MyPackage",
    products: [
        .library(
            name: "MyPackage",
            targets: ["MyPackage","TargetA","TargetB"]), //1，需要指明对外暴露的target
    ],
    //......省略部分代码
    targets: [
        .target(                   //2，MyPackage定义
            name: "MyPackage",    
            dependencies: []),
        .target(                   //3，TargetA定义
            name: "TargetA",
            dependencies: []),
        .target(                   //4，TargetB定义
            name: "TargetB",
            dependencies: []),
        //......省略部分代码
    ]
)
```

其中需要特别提到target对象中path参数的设置

```swift
 static func target(
     name: String,
     dependencies: [Target.Dependency] = [],
     path: String? = nil,
     exclude: [String] = [],
     sources: [String]? = nil,
     resources: [Resource]? = nil,
     publicHeadersPath: String? = nil,
     cSettings: [CSetting]? = nil,
     cxxSettings: [CXXSetting]? = nil,
     swiftSettings: [SwiftSetting]? = nil,
     linkerSettings: [LinkerSetting]? = nil
 ) -> Target
```
当target的name参数指定的字符串与对应Sources下的目录名称完全一致时，则无需设置path参数，`SwiftPM`默认查找name参数同名目录；

**不符合上述情况时，则必须指定path参数；**

* 如果name参数名与Sources下目录名不一致，则需要指定path参数路径；
* 如果所有文件直接放在Sources下，没有新建文件夹，则需要指定path参数为`path: "Sources"`；
* path参数设置时，支持相对路径模式，即可以使用“.”或“..”路径匹配。

## 依赖处理

`SwiftPM`类似`CocoaPods`，可以添加其他依赖的Package，这里以添加本地依赖项为例说明：

```swift
 let package = Package(
     name: "NetService",
     platforms: [.iOS(.v11)], //指定Package支持的平台
     products: [
         .library(
             name: "NetService",
             targets: ["NetService"]),
     ],
     dependencies: [
         // 当前Package依赖的外部依赖项，以local package相对路径为例
         .package(name: "UserAndSetting", path: "../UserAndSetting"),
         .package(name: "Alamofire", path: "../Product/Alamofire"),
         .package(name: "CryptoSwift", path: "../Product/CryptoSwift")
     ],
     targets: [
         .target(
             name: "NetService",
             dependencies: ["UserAndSetting","Alamofire","CryptoSwift"],//当前target依赖的外部依赖项需要在此处指定，字符串或使用.target参数指定
             path: "Sources"), //此处指定path是Sources目录
         .testTarget(
             name: "NetServiceTests",
             dependencies: ["NetService"]),
     ]
 )
```

* Package顶层的dependencies，添加的是外部依赖；外部依赖是指当前Package以外其它`SwiftPM`；
* 每个target中的dependencies添加的是当前target需要的依赖项；可以是外部依赖，在数组中增加外部依赖的名称；可以是其他的target，使用`.target`方式引入；

```swift
.target(name: "TargetA",
                dependencies: [.target(name: "MyPackage")]),
```

这样TargetA中可以调用MyPackage中对外提供的API接口；

```swift
import MyPackage

struct TargetA{
    func testfun() {
        print("TargetA")
        let p = MyPackage()
        p.tttdebug()
    }
}
```
## Objc与Swift混编

在`SwiftPM`中，一个target中，只能存在一种语言，不可以混编；

假设现在有一个完整的业务功能的代码，是混编的，即Swift/Objc代码都有；我们的目标是使用`SwiftPM`进行模块化处理；一种简单方案是将混编代码中的Objc代码，先转写成Swift代码，这样就不再存在混编问题，下一步处理成`SwiftPM`是最简单的；

但是，可能由于一些限制或者原因，Objc不能变成Swift；必须混编。

两种改成`SwiftPM`的解决方案：

* 多Package模式：将代码按语言不同分开，拆分成两个Package，但是需要满足不存在**循环依赖关系**，只是单向依赖没有问题；
* 单Package模式：在一个Package中，`Sources`文件夹下按语言建立两个单独的目录，分别存放Swift代码和Objc代码，在Package配置文件中建立两个Target，每个Target设置明确的path路径，如`path: "Sources/A"`，同时设置Target的依赖关系；

**注意：任何一种方式都需要满足模块代码之间不能循环依赖，如果有这种情况发生，需要先重构代码！**

## 最佳实践

* 梳理清楚代码之间的依赖关系，对于设计`SwiftPM`目录结构至关重要；
* 根据实际情况，选择“单target模式”或“多target模式”组织代码目录结构；
* 当target的name参数指定的字符串与对应Sources下的目录名称完全一致时，无需设置path参数，其他情况下需要设置；
* 处理Objc与Swift混编时，需要满足模块代码之间不存在循环依赖关系；


# 资源处理

在开发过程中，图片、文本、JSON、XML等资源文件，是我们必须使用的；接下来通过一个具体的实例，展示不同类型的资源文件`SwiftPM`如何处理；这里创建了一个实例工程`SpmResourceTest`，列举了各种资源文件的存放方式和目录设置；

![cd47201357baf22236ea4e3c6c218b6e](/assets/images/资源文件处理.png)

## Package内部几种资源处理的方式

* 直接把资源文件拖到项目中：放在根目录下，如`ic_linkfailed_mid_normal@2x`图片，或者放在创建的子目录下，如images目录和json目录，下面都有不同的资源文件；
* 使用`Asset Catalog`，在`SwiftPM`中会默认命名为Media
* 使用`Bundle`，如HYContentShare.bundle

从`DerivedData`找到对应的项目编译后的成果

![0348ba5b0f87fa088de22bb9d8bb82a4](/assets/images/截屏2022-07-01 10.27.41.png)

注意到出现了一个`SpmResourceTest_SpmResourceTest.bundle`的生成文件，显示包内容查看内部结构：

![3c43a83d3001efc3807b700ca4ec3d36](/assets/images/截屏2022-07-01 10.28.15.png)

当前target的resources设置
```swift
    .target(
            name: "SpmResourceTest",
            dependencies: [],
            resources: [.copy("HYContentShare.bundle"),
                        .process("images"),
                        .copy("json")]),
```

发现一些不同的规律：

* 没有在配置中明确指定的文件，如`ic_linkfailed_mid_normal@2x`是不会被处理的，即不被编译进当前bundle中；在编译时，编译器也会出现提醒，例如`found 3 file(s) which are unhandled; explicitly declare them as resources or exclude from the target`；
* 使用`procees`处理的文件目录，其目录结构下的内容会被 **平铺（减少目录结构层级的一种方式）** 放到bundle的根层级下，没有子目录出现，如images目录；
* 使用`copy`处理的文件目录，会保留目录层级结构，放在bundle中，如json目录；
* 资源中如果包含bundle文件，如当前例子中HYContentShare.bundle，不同`swift-tools-version`版本使用有区别；
    * `swift-tools-version:5.5`及以下，使用`copy`与`process`没有区别，都会保持`HYContentShare.bundle`结构；
    * `swift-tools-version:5.6`及以上，如果使用`copy`处理，保持HYContentShare.bundle结构；如果改为`process`处理，则也会将资源平铺放到根目录下，不再有HYContentShare.bundle；如下图所示；

![2fbe0e6414b476cc3ad2fcf1e772bc05](/assets/images/截屏2022-07-01 10.54.53.png)

>注意：由于`Swift Package Manager`随着时间推移也在不断迭代，所以需要注意`swift-tools-version`中指定的版本号，有些功能在高版本与低版本内部实现有差别，某些新功能在高版本才能使用；
>`// swift-tools-version:5.6`
>`// The swift-tools-version declares the minimum version of Swift required to build this package.`

## 读取资源

下面介绍读取资源的代码实现；

### 读取Asset Catalog中图片

```swift
//从Media直接获得图片，不需要区分2x 3x
UIImage(named: "ic_lianjie_grey_normal", in: .module, compatibleWith: UITraitCollection())!
```

### 获取prcoess处理的资源

* 图片
```swift
//方式1，通过路径path
    public func getImageUrl() -> UIImage{
        let url = Bundle.module.url(forResource: "ic_right@2x", withExtension: "png")
        let path = url?.path ?? ""
        let image = UIImage(contentsOfFile: path)
        return image!
    }
//方式2，使用UIImage的API
    public func getImageUrl2() -> UIImage{
        UIImage(named: "ic_right", in: .module, compatibleWith: UITraitCollection())!
    }
```

* 其他资源
仍然通过`Bundle.module.url`获得path
```swift
let url = Bundle.module.url(forResource: "xzloading_middle", withExtension: "json")
let path = url?.path ?? ""
```

### 获取copy处理的资源

```swift
//创建获取Bundle路径的扩展函数
extension Bundle{
    public static func inner(path:String) -> Bundle{
        let bundleURL = Bundle.module.bundleURL
        let subURL  = bundleURL.appendingPathComponent(path)
        return Bundle(url: subURL) ?? Bundle.module
    }
}

//获取copy处理的资源
public func getCopyResource() -> String {
    let url = Bundle.inner(path: "json").url(forResource: "xzloading_big", withExtension: "json")
    var string = ""
    do {
       string =  try String(contentsOfFile: url?.path ?? "")
    } catch {
       print("error=\(error)")
    }
    return string
}
```

## 最佳实践

* 优先使用`Asset Catalog`管理资源，使用起来最简便；
* 如果不能使用Asset，比如json文件或者plist，并且可能混合图片资源一起使用（如Lottie动画），优先使用`process`处理；
* 如果需要多套同名资源同时存在，如`dark/icon.png`,`light/icon.png`，则通过建立多个目录，使用`copy`处理是合适的,使用时读取不同的path；


# 链接方式的选择

### 静态链接与动态链接的区别

我们在编写代码的同时，也需要使用别人提供的库或者框架，就需要使用链接器；链接分为两种类型：
* 静态链接，它发生在编译构建 App 的时候，影响到构建的耗时以及 App 最终的二进制体积；
* 动态链接，它发生在 App 启动的时候，影响 App 的启动耗时；

由于上述两种链接方式的区别，一般来说，推荐更多的使用静态链接，减少动态链接，来降低App启动耗时；但是，如果是在开发/调试阶段，频繁修改的代码，建议采用动态链接方式，降低构建的耗时。

### SwiftPM编译链接选项

```swift
static func library(
    name: String,
    type: Product.Library.LibraryType? = nil,
    targets: [String]
) -> Product
```

其中type参数的类型是`Product.Library.LibraryType`，可以通过指定`.static`或`.dynamic`来决定生成的`SwiftPM`最终产物的形态，是动态链接库或是静态链接库；

通过下面这张图，解释一下不同的type参数设置，对生成的framework的影响：

![81cd5b412a84ba96f8413482882fbcd4](/assets/images/SwiftPM-framework.png)

* 我们假设有两个`SwiftPM`，分别为`consumer`和`producer`，其中`consumer`依赖`producer`；
* 每个`SwiftPM`在编译后都会生成对应的中间文件，可重定位对象文件（`relocatable object`），即`.o `文件；
* 当`consumer`和`producer`的type均设置为`nil`或`static`时，都会仅生成中间文件.o；
* 当`consumer`的type设置为`dynamic`，`producer`的type设置为`nil`或`static`，在生成对应的中间文件后，`consumer.o`与`producer.o`会进行合并，生成最终的动态链接库`consumer.framework`
* 当`consumer`的type设置为`dynamic`，`producer`的type设置为`synamic`，在生成对应的中间文件后，`consumer.o`与`producer.o`会分别生成对应的动态链接库`consumer.framework`与`producer.framework`

通过上面的举例描述，我们可以得到下面结论：

* `SwiftPM`的LibraryType设置为`dynamic`时，会生成动态链接库，同时当前库所依赖的库，如果不是设置为`dynamic`，则会被合并编译成一个动态链接库framework；

## 最佳实践

* 如果`SwiftPM`的产品可以静态链接，也可以动态链接。建议不要明确声明库的类型，这样`SwiftPM`可以根据包使用者的偏好，选择静态链接还是动态链接。


# 编译与链接参数设置

## 定制编译参数

在CocoaPods使用中可以通过配置指定在Debug模式下导入；
```swift
pod 'SourceModel', :configurations => ['Debug']
```
对应的，`SwiftPM`也可以通过配置实现相同的功能，并且功能更强大；我们通过一段代码示例来展示一下，这段代码源自[官方Swift提案](https://github.com/apple/swift-evolution/blob/master/proposals/0273-swiftpm-conditional-target-dependencies.md#proposed-solution)；
```swift
import PackageDescription

let package = Package(
    name: "BestPackage",
    dependencies: [
        .package(url: "https://github.com/pureswift/bluetooth", .branch("master")),
        .package(url: "https://github.com/pureswift/bluetoothlinux", .branch("master")),
    ],
    targets: [
        .target(
            name: "BestExecutable",
            dependencies: [
                .product(name: "Bluetooth", condition: .when(platforms: [.macOS])),
                //指定生效平台Mac
                .product(name: "BluetoothLinux", condition: .when(platforms: [.linux])),
                //指定生效平台Linux
                .target(name: "DebugHelpers", condition: .when(configuration: .debug)),
                //指定在Debug下生效
            ]
        ),
        .target(name: "DebugHelpers")
     ]
)
```

### 关闭ARC

我们在项目中一直使用Google版本的[protocolbuffer](https://github.com/protocolbuffers/protobuf)作为埋点数据序列化工具；到目前为止，它仍然仅支持Objective-C，不支持Swift；

生成的代码是非ARC的，需要特别在BuildSetting中标注；
![6776e7a968960c73a1ddd51e7f1a831c](/assets/images/NoArcBuildSetting.png)

如果对代码进行改造使用`SwiftPM`包管理器进行处理，需要设置参数关闭ARC；
```swift
.target(
     name: "ProtobufFiles",
     dependencies: ["Protobuf"],
     path: "Sources",
     publicHeadersPath: ".",
     cSettings: [.headerSearchPath("."), //objc代码指定头文件搜索路径
                .define("GPB_USE_PROTOBUF_FRAMEWORK_IMPORTS"),//设置预编译宏
                .unsafeFlags(["-fno-objc-arc"]) //关闭ARC
     ]),
```
**注意：这种方式针对当前target进行设置，与原有的BuildSetting中针对文件的设置不同，影响target中所有文件；**

### 预编译宏设置

我们继续通过Google版本的[protocolbuffer](https://github.com/protocolbuffers/protobuf)为例分析，通过proto文件生成的Objc头文件包含预编译宏，如下面所示：

```objc
#if !defined(GPB_USE_PROTOBUF_FRAMEWORK_IMPORTS)
 #define GPB_USE_PROTOBUF_FRAMEWORK_IMPORTS 0
#endif

#if GPB_USE_PROTOBUF_FRAMEWORK_IMPORTS
 #import <Protobuf/GPBProtocolBuffers.h>
#else
 #import "GPBProtocolBuffers.h"
#endif
```
参考CocoaPods配置
```swift
GCC_PREPROCESSOR_DEFINITIONS = $(inherited) COCOAPODS=1 $(inherited) GPB_USE_PROTOBUF_FRAMEWORK_IMPORTS=1
```
在taregt的`cSettings`中需要设置宏，才能编译通过；
```swift
.define("GPB_USE_PROTOBUF_FRAMEWORK_IMPORTS")
```
但是，问题并没有真的解决，当其他模块依赖当前target时，**公开暴露出去的 header 的 #if 判断不能使用任何自定义的宏变量**，所以均无法编译通过；

**因此，在`SwiftPM`中不能通过任何自定义的宏变量的方式，向外暴露头文件；**


## 链接参数设置

通过前面的分析可以发现，由于Swift Package Manager发展时间相对CocoaPods更短，很多三方组件或开源项目均没有适配SwiftPM，无对应的Package.swift配置，所以当我们项目中自行接入此类项目时，参考对应项目的CocoaPods配置是很好的借鉴方式；

还以高德地图为例，对应项目的CocoaPods配置如下所示：
```swift
OTHER_LDFLAGS = $(inherited) -l"c++" -l"z" -framework "CoreLocation" -framework "CoreTelephony" -framework "QuartzCore" -framework "Security" -framework "SystemConfiguration"
```
这里我们借鉴上面的配置，在当前target的linkerSettings中增加必要的链接参数；
```swift
linkerSettings: [.linkedLibrary("c++"),
                             .linkedLibrary("z"),
                             .linkedFramework("ExternalAccessory"),
                             .linkedFramework("ImageIO"),
                             .linkedFramework("MobileCoreServices"),])
```
一般来说，通过这种方式调整后，都可以顺利编译通过，正常使用。

# 异常处理

接下来介绍一些使用`SwiftPM`可能会遇到的特殊情况，如何处理。
## 非Clang Module生成的SwiftPM接入

`Clang Module`包含`module.modulemap`，某些三方SDK历史版本比较老，生成的XCFramework不含`module.modulemap`，在通过`SwiftPM`接入项目主工程时，例如MAMapKit（高德地图），这类模块无法直接暴露给Swift，必须通过桥接方式引入；

```swift
#import <MAMapKit/MAMapKit.h>  //高德地图SDK需要保留，MAMapKit没有被编译为Mudule，Swift引用不到，必须通过桥接方式使用
```

## 无法解决的包管理的问题

如果使用Xcode + SwiftPM 遇到奇怪问题，使用过了`Clean Build Folder`，删除`DerivedData`，重启Xcode等一系列方式后，仍然无法解决，请尝试下面方式；

```swift
rm -rf ~/Library/Caches/org.swift.swiftpm
rm -rf ~/Library/org.swift.swiftpm 
```

# 总结

`Swift Package Manager`作为苹果推出的包管理依赖工具，可以说是补足了苹果生态的短板；`SwfitPM`相比`Cocoapods`它配置更加简洁易用，相比`Carthage`更加轻量化，无入侵；因为Swift本身是跨平台语言，`SwfitPM`完全使用Swift编写，所以使用场景不仅仅局限于Mac平台；

随着今年WWDC22上`Swift Package Plugins`的发布，解决了`SwfitPM`目前不支持在构建期间执行任何自定义操作的问题，包括源代码生成以及特殊类型资源的自定义处理等问题；为提高研发流程效率，更加自动化提供了有力的支持；


# 参考

* https://github.com/apple/swift-evolution/blob/master/proposals/0273-swiftpm-conditional-target-dependencies.md#proposed-solution
* https://useyourloaf.com/blog/add-resources-to-swift-packages/
* https://developer.apple.com/documentation/xcode/bundling-resources-with-a-swift-package