---
title: "由App Extension无法编译看Swift API可用性"
description: "从Xcode12.5开始，苹果要求所有的Extension Target必须设置APPLICATION_EXTENSION_API_ONLY为true，否则将会导致编译错误“Application extensions and any libraries they link to must be built with the `APPLICATION_EXTENSION_API_ONLY` build setting set to YES”；但是我们通常会在主工程和Extension之间使用Framework或其他方式共享代码，这些代码中使用了非extension-only API，所以导致问题出现，本篇文章将探讨如何解决这个问题"
category: programming
tags: iOS,Swift,Extension
---


从Xcode12.5开始，苹果要求所有的Extension Target必须设置APPLICATION_EXTENSION_API_ONLY为true，否则将会导致编译错误“Application extensions and any libraries they link to must be built with the `APPLICATION_EXTENSION_API_ONLY` build setting set to YES”；但是我们通常会在主工程和Extension之间使用Framework或其他方式共享代码，这些代码中使用了非extension-only API，所以导致问题出现，本篇文章将探讨如何解决这个问题。

# 探索

我们以一个具体的工程结构为例，如下图所示：

![30f5f7fba999fba5f30b04a099f35abb](/assets/images/AppExtension.png)

我们的主工程Host App中，创建了一个`Share Extension`的扩展Target做分享相关的操作；另外，为了模块化，我们有一个`Library`工程包含所有的基础组件和Fundation扩展方法，`NetworkService`工程包含网络请求相关的功能封装和处理，他们都被编译为Framework供主工程和`Share Extension`共同使用；

我们首先需要把`Share Extension`、`Library`、`NetworkService`这三个工程的Build Setting中APPLICATION_EXTENSION_API_ONLY设置为true；由于我们在`Library`和`NetworkService`中都使用了`UIApplication.shared.open`，`UIApplication.shared.keyWindow`这类非extension-only的API，所以编译这两个子工程是无法通过的。


首先想到的解决办法是代码拆分，我们可以把`Library`按是否使用extension-only API进行拆分，拆分成两个工程`Libray`和`LibraryExtension`，`LibrayExtension`中包含符合extension-only的API，提供给`Share Extesnion`使用；`Library`中包含其他不设限API，提供给主工程或其他非Extesnion工程使用；

然后`NetworkService`也采用相同方法进行改造，这种方式是可以解决问题的，但是除了拆分代码创建新工程的代价，也会带来很多额外的工作量；比如主工程中Host App，原来都是引用`import Library`，现在就需要逐个修改确认，是使用`Libray`还是`LibraryExtension`，或者添加同时引用两个，这对于一个已有的大工程来说，是一个不小的工作量；

![8acab95c90ed5aa452353df9ef8caeb1](/assets/images/AppExtension2.png)

另外，笔者在搜索这个问题的解决方式时，发现[swift-cast](https://swift-cast.com/2021/04/38/)网站作者提供了一个另外的解决方案：使用`ACTION_EXTENSION`；如果你的App恰好使用的是`Action Extension`可以采用这个方式去尝试：
```swift
#if !ACTION_EXTENSION
    //codes that don't obey extension-only API requests
#else
    //normal codes 
#endif 
```

显然，上面的两种方式都有各自的局限性，所以我们需要寻找更加广泛使用的解决方案，改动更小的解决方案；Swift语言提供了API可用性的标识，这个功能能够解决我们所遇到的问题，下面我们先来了解一下Swift的API可用性。

# Swift的API可用性（API availability）

在Swift中使用@available可以标记API的可用性信息，比如是否API在某个版本被废弃，这个API需要的Swift版本大于5.4才能使用，等等。我们来看几个具体方面：

## 平台可用性

```swift
@available(iOS 13.0, OSX 10.15, *)
@available(tvOS, unavailable)
@available(watchOS, unavailable)
public struct SearchField: View{
    ...
}
```
在SwiftUI中我们定义一个SearchField组件，就需要对其适用的平台和系统版本进行限制，上面这段代码表明，SearchField适用于大于等于`iOS13`或`OSX 10.15`版本，同时针对`tvOS`和`watchOS`都是不可用的。

具体注释定义如下：
```swift
@available(platform version , platform version ..., *)
```

* platform：指定具体适用的平台，比如`iOS`, `macCatalyst`, `macOS/OSX`, `tvOS` 或者 `watchOS`，**还可以指定适用的扩展Extension Target，比如`ApplicationExtension`或`macOSApplicationExtension`，这是解决文章开头提出的问题的重要工具，这部分稍后将详细介绍**；
* version：具体的数字，可以由一位、两位或三位正整数通过点号（.）分割组成，分别代表主版本号，次版本号，补丁版本号；
* 可以有多个platform+version组成，之间用逗号分隔（,），比如`@available(iOS 13.0, OSX 10.15, *)`；
* 星号(*)，表示该API可用于所有其他平台。为了处理潜在的未来平台，平台可用性注释总是需要一个星号；

## API可用性

我们在软件开发过程中会不断改进，引入新的API，废弃旧的API，与此对应的@available可以进行这部分工作API的标记。

```swift
// With introduced, deprecated, and/or obsoleted
@available(platform | *
          , introduced: version , deprecated: version , obsoleted: version
          , renamed: "..."
          , message: "...")

// With unavailable
@available(platform | *, unavailable , renamed: "..." , message: "...")
```

* platform：与前面介绍一致；
* introduced, deprecated, obsoleted：API在指定版本开始可以使用，标记introduced；API即将在指定版本被废弃，标记deprecated，使用此API时编译产生警告；API在指定版本开始被淘汰，标记obsoleted，使用此API时编译产生错误。
* unavailable：通过与platform配合使用，表示指定平台版本API不能使用，如果在此情况下使用此API将会产生编译错误。
* renamed：当使用此API时提供另一个API用来替换当前API，当有此标识时Xcode提供了一个自动修复选项。
* message：在编译警告或错误发生时，提供一个字符串说明给使用者。

与平台可用性不同，这种形式只允许指定一个平台。因此，如果你想注释多个平台的可用性，你需要使用多个@available属性。例如，下面是简单示例如何表示多个平台:

```swift
@available(macOS, introduced: 10.15)
@available(iOS, introduced: 13)
@available(watchOS, introduced: 6)
@available(tvOS, introduced: 13)
```

##  #available

在Swift中，你可以使用可用性条件#available来断言if、guard和while语句，以确定运行时API的可用性。与@available属性不同，#available条件不能用于Swift语言版本检查。

#available表达式的语法类似于@available属性:

```swift
if | guard | while #available(platform version , platform version ..., *) …
```

不能使用&&和||等逻辑操作符组合多个#available表达式，但可以使用逗号，它们等价于&&。在实践中，这只对调整Swift语言版本和单个平台的可用性有用(因为对多个平台的检查要么是多余的，要么是不可能的)。
```swift
// 要求Swift 5 和 iOS 13
guard #available(swift 5.0), #available(iOS 13.0) else { return }
```

**注意，#available没有对应的unavailable标识，只能判断符合条件的；**

# 解决问题

有了前面介绍的基础知识，我们来讨论本文开头提出的“App Extension无法编译通过”的**具体解决方案：使用@available对API做函数级别的平台API标记；**

以下面gotoAppSystemSetting函数为例：
```swift
    @available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    @available(watchOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    @available(tvOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    @available(iOSMacApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    @available(OSXApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    static func gotoAppSystemSetting() {
        if let url = URL(string: UIApplication.openSettingsURLString) {
            if UIApplication.shared.canOpenURL(url) {
                UIApplication.shared.open(url, options: [:], completionHandler: nil)
            }
            
        }
        
    }
```
由于函数中使用了`UIApplication.shared.open`这类Extension不能使用的API，所以需要增加Extension不可用@available标记。

这样修改之后，由于明确的标记了API的可用性范围，开启APPLICATION_EXTENSION_API_ONLY为true之后，编译也可以正常通过。


## 函数调用链

单个API函数问题解决后，你还需要考虑函数调用的链，比如在函数A中使用了`UIApplication.shared.keyWindow`，那么A函数就需要标记为不能Extension使用的API；

```swift
@available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
func A() {
    ...
    //调用UIApplication.shared.keyWindow
    ...
}
```

接下来，如果有函数B调用了函数A，函数C都调用了函数B，那么函数B和C也都需要上述相同的`@available`标记；

```swift
@available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
func B() {
    ...
    A（） //函数B中调用A函数
    ...
}

@available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
func C() {
    ...
    B（） //函数C中调用B函数
    ...
}

```


## 协议函数

接下来，我们继续看协议函数相关的一个问题。有一个协议DialogueViewProtocol，声明了一个show方法如下面代码所示：

```swift
public protocol DialogueViewProtocol{
    func show(cancelHander acancelHander:(() -> Void)? ,comfirmHander acomfirmHander:(() -> Void)?)
}
```
我们有两种样式DialogueView组件需要实现，分别是DialogueView_1和DialogueView_2，为了抽取共同代码，它们都继承自DialogueView；
```swift
//DialogueView_1继承DialogueView
public class DialogueView_1:DialogueView{
   
}
//DialogueView_2继承DialogueView
public class DialogueView_2:DialogueView{

}
//DialogueView_1遵守DialogueViewProtocol实现show函数
extension DialogueView_1:DialogueViewProtocol{
    public func show(cancelHander acancelHander: (() -> Void)?,comfirmHander acomfirmHander:(() -> Void)?){
     ...
     //不同的内部实现
    }
    
//DialogueView_2遵守DialogueViewProtocol实现show函数    
extension DialogueView_2:DialogueViewProtocol{
    public func show(cancelHander acancelHander: (() -> Void)?,comfirmHander acomfirmHander:(() -> Void)?){
     ...
     //不同的内部实现
    }
```
在具体实现两个不同样式的DialogueView子类时，它们都遵守DialogueViewProtocol，实现了不同的show方法；

在show函数中两个方法都用到了`UIApplication.shared.keyWindow`，显然都无法被Extension使用，所以需要对show方法进行前面相同的`@available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")`标注，这样修改之后，API不被Extension使用的问题解决了，但是又引出了新的问题，导致show方法不可见，两个子View没有实现遵守DialogueViewProtocol协议；

那么怎么来解决呢？我们需要继续改造代码，在DialogueView基类中实现协议，来解决这个问题；

```swift
//基类DialogueView遵守DialogueViewProtocol
extension DialogueView:DialogueViewProtocol{
    public func show(cancelHander acancelHander:(() -> Void)? ,comfirmHander acomfirmHander:(() -> Void)?){
        //空实现
    }
}

extension DialogueView_1{
//标记，复写show函数
    @available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    public override func show(cancelHander acancelHander: (() -> Void)?,comfirmHander acomfirmHander:(() -> Void)?){
        ... 
    }
}

extension DialogueView_2{
//标记，复写show函数
    @available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    public override func show(cancelHander acancelHander: (() -> Void)?,comfirmHander acomfirmHander:(() -> Void)?){
        ...
    }
}

```
DialogueView遵循DialogueView，实现了show函数的空实现，然后在DialogueView_1和DialogueView_2中重写（override）show函数，需要注意DialogueView中show函数因为不使用Extension限制API，所以无需@available标记，而DialogueView_1和DialogueView_2是需要的。


## Objective-C函数API

使用Objective-C编写的函数也可能是用了Extension不可用的API，Objective-C语言也提供了`NS_EXTENSION_UNAVAILABLE_IOS`对其进行标记，系统函数中有很多这样的例子：

```objc
./EventKitUI.framework/Headers/EKEventViewController.h:NS_EXTENSION_UNAVAILABLE_IOS("EventKitUI is not supported in extensions")
./Foundation.framework/Headers/NSObjCRuntime.h:#define NS_EXTENSION_UNAVAILABLE_IOS(_msg)  __IOS_EXTENSION_UNAVAILABLE(_msg)
./UIKit.framework/Headers/UIAlertView.h:- (instancetype)initWithTitle:(NSString *)title message:(NSString *)message delegate:(id /*<UIAlertViewDelegate>*/)delegate cancelButtonTitle:(NSString *)cancelButtonTitle otherButtonTitles:(NSString *)otherButtonTitles, ... NS_REQUIRES_NIL_TERMINATION NS_EXTENSION_UNAVAILABLE_IOS("Use UIAlertController instead.");
./UIKit.framework/Headers/UIApplication.h:- (void)beginIgnoringInteractionEvents NS_EXTENSION_UNAVAILABLE_IOS("");               // nested. set should be set during animations & transitions to ignore touch and other events
```

具体针对iOS来说：
```
NS_EXTENSION_UNAVAILABLE_IOS（"..."）
```
标记Extension不能使用这些API,后面有一个参数，可以作为提示，用什么API替换

### 普通API

在头文件中对函数添加标记

```objc
+ (UIImage *)launchImage NS_EXTENSION_UNAVAILABLE_IOS("");
```

### 重写系统方法的API

如果是重写了系统方法，比如继承UIViewController，重写了statusBarStyle，那么对其进行标记就需要在实现函数中

```objc
- (UIStatusBarStyle)statusBarStyle NS_EXTENSION_UNAVAILABLE_IOS("") {
    return [UIApplication sharedApplication].statusBarStyle;
}
```


# 结语


由于Xcode12.5新版本带来的变化，导致我们需要利用Swift语言API可用性的标记，对不合符extension-only的API进行
`@available(iOSApplicationExtension, unavailable)`标记，Objective-C语言也有对应的`NS_EXTENSION_UNAVAILABLE_IOS（"..."）`标记可以使用；你也需要对当前标记函数调用链上的上游函数增加标记，另外，你也可能需要针对协议函数，实现默认实现，以解决标记@available不可见问题。


# 参考

* https://stackoverflow.com/questions/33308196/checking-for-protocol-availability-in-swift
* https://forums.swift.org/t/availability-checking-for-protocol-conformances/42066/8
* https://www.cnblogs.com/lxlx1798/articles/13060713.html
* https://apollozhu.github.io/2017/06/20/swift-and-ns-extension-unavailable/
* https://swift-cast.com/2021/04/38/
* https://nshipster.com/available/
* https://davedelong.com/blog/2019/04/09/conditional-compilation-part-3/