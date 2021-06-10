---
title: "Using Swift API availability to solve App Extension Compiled Error"
description: "Starting from Xcode12.5, Apple requires that all Extension Targets must set APPLICATION_EXTENSION_API_ONLY to true, otherwise it will cause a compilation error "Application extensions and any libraries they link to must be built with the `APPLICATION_EXTENSION_API_ONLY` build setting set to YES"; but we Framework or other methods are usually used to share code between the main project and the extension. These codes use non-extension-only APIs, which leads to problems. This article will discuss how to solve this problem."
category: programming
tags: iOS,Swift,Extension,APPLICATION_EXTENSION_API_ONLY
---

Starting from Xcode12.5, Apple requires that all Extension Targets must set APPLICATION_EXTENSION_API_ONLY to true, otherwise it will cause a compilation error "Application extensions and any libraries they link to must be built with the `APPLICATION_EXTENSION_API_ONLY` build setting set to YES"; but we Framework or other methods are usually used to share code between the main project and the extension. These codes use non-extension-only APIs, which leads to problems. This article will discuss how to solve this problem.

# Explore
Let's take a specific  structure as an example, as shown in the following figure:

![30f5f7fba999fba5f30b04a099f35abb](/assets/images/AppExtension.png)

In our main project Host App, we created an extension Target of `Share Extension` to do share-related operations; in addition, for modularization, we have a `Library` project that contains all the basic components and Fundation extension methods, `NetworkService` The project contains functional packaging and processing related to network requests, and they are all compiled into Framework for common use by the main project and `Share Extension`;

We first need to set the APPLICATION_EXTENSION_API_ONLY in the Build Setting of the three projects `Share Extension`, `Library`, and `NetworkService` to true; because we use `UIApplication.shared.open` in both `Library` and `NetworkService` , `UIApplication.shared.keyWindow` is a non-extension-only API, so it is impossible to compile these two sub-projects.

The first solution that comes to mind is code splitting. We can split `Library` according to whether extension-only API is used or not, and split it into two projects `Libray` and `LibraryExtension`, and `LibrayExtension` contains extension-only APIs for `Share Extesnion`; `Library` contains other unrestricted APIs, which are provided to the main project or other non-Extesnion projects;

Then `NetworkService` also uses the same method for transformation. This method can solve the problem, but in addition to the cost of splitting the code to create a new project, it will also bring a lot of extra workload; for example, the Host App in the main project, the original It is to reference `import Library`, and now you need to modify and confirm one by one, whether to use `Libray` or `LibraryExtension`, or add two references at the same time, this is a lot of work for an existing large project;

![8acab95c90ed5aa452353df9ef8caeb1](/assets/images/AppExtension2.png)

Obviously, the above methods have their own limitations, so we need to find more widely used solutions, with smaller changes; Swift language provides API usability identification, this function can solve the problems we encounter, Let's first understand the availability of Swift API.

# Swift API Availability

In Swift, you use the `@available` attribute to annotate APIs with availability information,such as whether the API has been deprecated in a version. the API requires a Swift version greater than 5.4 to be used, and so on. Let's look at a few specific aspects:

## Platform Availability

```swift
@available(iOS 13.0, OSX 10.15, *)
@available(tvOS, unavailable)
@available(watchOS, unavailable)
public struct SearchField: View{
    ...
}
```

In SwiftUI, We define a SearchField component, and we restrict the platform and system versions that it applies to. The above code indicates that SearchField is applicable to versions `iOS13`  or `OSX 10.15`  or greater. Both `tvOS` and `watchOS` are not available.

```swift
@available(platform version , platform version ..., *)
```

* platform：Specify specific platforms，for example `iOS`, `macCatalyst`, `macOS/OSX`, `tvOS` 或者 `watchOS`, **You can also specify Extension Target , such as`ApplicationExtension` or `macOSApplicationExtension`, this is one of the important tool to deal with the problem raised by the opening, this section will in detail later**.
* version：A version number consisting of one, two, or three positive integers, separated by a period (.), to denote the major, minor, and patch version.
* Zero or more versioned platforms in a comma-delimited (,) list.
* An asterisk (*), denoting that the API is available for all other platforms. An asterisk is always required for platform availability annotations to handle potential future platforms.



## API Availability

As we go through the software development process, we constantly improve by introducing new APIs and discarding old ones, and the corresponding @available API allows us to do this part of the work.

```swift
// With introduced, deprecated, and/or obsoleted
@available(platform | *
          , introduced: version , deprecated: version , obsoleted: version
          , renamed: "..."
          , message: "...")

// With unavailable
@available(platform | *, unavailable , renamed: "..." , message: "...")
```

* A platform, same as before, or an asterisk (*) for all platforms.
* Either introduced, deprecated, and/or obsoleted…
	* An introduced version, denoting the first version in which the API is available
	*	A deprecated version, denoting the first version when using the API generates a compiler warning
	* An obsoleted version, denoting the first version when using the API generates a compiler error
* …or unavailable, which causes the API to generate a compiler error when used
* renamed with a keypath to another API; when provided, Xcode provides an automatic “fix-it”
* A message string to be included in the compiler warning or error

Unlike shorthand specifications, this form allows for only one platform to be specified. So if you want to annotate availability for multiple platforms, you’ll need stack @available attributes.

##  #available

In Swift, you can predicate if, guard, and while statements with an availability condition, #available, to determine the availability of APIs at runtime. Unlike the @available attribute, an #available condition can’t be used for Swift language version checks.

The syntax of an #available expression resembles that of an @available attribute:

```swift
if | guard | while #available(platform version , platform version ..., *) …
```

You can’t combine multiple #available expressions using logical operators like && and ||, but you can use commas, which are equivalent to &&. In practice, this is only useful for conditioning Swift language version and the availability of a single platform (since a check for more than one would be either redundant or impossible).

Some of the above content is quoted from https://nshipster.com/available/

# Solve Problem
Let's discuss the concrete solution to the problem of  App Extension Compiling Error,use Swift API Availability.

Take the following `GoToAppSystemSetting` function as an example:

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

In this function,  use  `UIApplication.shared.open` API, Extension Target can not use, So need add Swift API Availability

With this modification, the compilation will pass if APPLICATION_EXTENSION_API_ONLY is turned on to true, because the availability range of the API is clearly marked.

## Call Function Chain

After the problem of a single API function is solved, you also need to consider the chain of function calls. For example, if `UIApplication.shared.keyWindow` is used in function A, then function A needs to be marked as an API that cannot be used by Extension Target;

```swift
@available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
func A() {
    ...
    //调用UIApplication.shared.keyWindow
    ...
}
```

Next, if function B calls function A, and both function C call function B, then both functions B and C also need the same `@available` tag;

```swift
@available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
func B() {
    ...
    A（） //Function B call Function A
    ...
}

@available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
func C() {
    ...
    B（） //Function C call Function B
    ...
}

```

## Protocol Function

Next, we continue to look at a problem related to the protocol function. There is a protocol DialogueViewProtocol, which declares a show method as shown in the following code:

```swift
public protocol DialogueViewProtocol{
    func show(cancelHander acancelHander:(() -> Void)? ,comfirmHander acomfirmHander:(() -> Void)?)
}
```
We have two styles of DialogueView components that need to be implemented, namely DialogueView_1 and DialogueView_2, in order to extract common code, they both inherit from DialogueView;

```swift
//DialogueView_1 inherit DialogueView
public class DialogueView_1:DialogueView{
   
}
//DialogueView_2 inherit DialogueView
public class DialogueView_2:DialogueView{

}
//DialogueView_1 conform to protocol DialogueViewProtocol , Implement show function
extension DialogueView_1:DialogueViewProtocol{
    public func show(cancelHander acancelHander: (() -> Void)?,comfirmHander acomfirmHander:(() -> Void)?){
     ...
     
    }
    
//DialogueView_2 conform to protocol DialogueViewProtocol , Implement show function  
extension DialogueView_2:DialogueViewProtocol{
    public func show(cancelHander acancelHander: (() -> Void)?,comfirmHander acomfirmHander:(() -> Void)?){
     ...
     
    }
```
When implementing two different styles of DialogueView subclasses, they all obey DialogueViewProtocol and implement different show methods;

In the show function, both methods use `UIApplication.shared.keyWindow`, which obviously cannot be used by Extension, so the same `@available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE. ")` mark. After this modification, the problem that the API is not used by Extension is solved, but a new problem is introduced, which causes the show method to be invisible, and the two child views do not implement the DialogueViewProtocol protocol;

So how to solve it? We need to continue to transform the code and implement the protocol in the DialogueView base class to solve this problem;

```swift
//Base Class DialogueView conform to DialogueViewProtocol
extension DialogueView:DialogueViewProtocol{
    public func show(cancelHander acancelHander:(() -> Void)? ,comfirmHander acomfirmHander:(() -> Void)?){
        //empty 
    }
}

extension DialogueView_1{
//use API Availability,override show function 
    @available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    public override func show(cancelHander acancelHander: (() -> Void)?,comfirmHander acomfirmHander:(() -> Void)?){
        ... 
    }
}

extension DialogueView_2{
//use API Availability,override show function 
    @available(iOSApplicationExtension, unavailable, message: "This method is NS_EXTENSION_UNAVAILABLE.")
    public override func show(cancelHander acancelHander: (() -> Void)?,comfirmHander acomfirmHander:(() -> Void)?){
        ...
    }
}

```
DialogUeView follows DialogUeView to realize the empty implementation of show function, and then override show function in DialogUeView_1 and DialogUeView_2. It should be noted that the show function in DialogUView does not need the @available flag because it does not use the Extension Restriction API, while DialogUEView_1 and DialogUEView_2 are required.

## Objective-C Function API

A function written in Objective-C may also use an API that is not available with Extension. The Objective-C language also provides' NS_EXTENSION_UNAVAILABLE_IOS 'to flag it. There are many examples of this in system functions:

```objc
./EventKitUI.framework/Headers/EKEventViewController.h:NS_EXTENSION_UNAVAILABLE_IOS("EventKitUI is not supported in extensions")
./Foundation.framework/Headers/NSObjCRuntime.h:#define NS_EXTENSION_UNAVAILABLE_IOS(_msg)  __IOS_EXTENSION_UNAVAILABLE(_msg)
./UIKit.framework/Headers/UIAlertView.h:- (instancetype)initWithTitle:(NSString *)title message:(NSString *)message delegate:(id /*<UIAlertViewDelegate>*/)delegate cancelButtonTitle:(NSString *)cancelButtonTitle otherButtonTitles:(NSString *)otherButtonTitles, ... NS_REQUIRES_NIL_TERMINATION NS_EXTENSION_UNAVAILABLE_IOS("Use UIAlertController instead.");
./UIKit.framework/Headers/UIApplication.h:- (void)beginIgnoringInteractionEvents NS_EXTENSION_UNAVAILABLE_IOS("");               // nested. set should be set during animations & transitions to ignore touch and other events
```

```
NS_EXTENSION_UNAVAILABLE_IOS（"..."）
```

Mark Extension cannot use these APIs, there is a parameter behind, which can be used as a reminder, what API to replace

### Common API

Mark the function in the header file

```objc
+ (UIImage *)launchImage NS_EXTENSION_UNAVAILABLE_IOS("");
```

### Override System API

If the system method is override, such as inheriting UIViewController and overriding statusBarStyle, then marking it needs to be implemented in the function itself.

```objc
- (UIStatusBarStyle)statusBarStyle NS_EXTENSION_UNAVAILABLE_IOS("") {
    return [UIApplication sharedApplication].statusBarStyle;
}
```

# Conclusions

Due to the changes of new version Xcode12.5, we need to use the Swift API Availability for not extension-only APIs to use `@available(iOSApplicationExtension, unavailable)` mark, Objective-C language also has corresponding `NS_EXTENSION_UNAVAILABLE_IOS("...")` mark can be used; you also need to add mark to the upstream function on the current mark function call chain, in addition, You may also need to implement the default implementation for protocol functions to solve the problem of invisible mark @available.

# Reference

* https://stackoverflow.com/questions/33308196/checking-for-protocol-availability-in-swift
* https://forums.swift.org/t/availability-checking-for-protocol-conformances/42066/8
* https://www.cnblogs.com/lxlx1798/articles/13060713.html
* https://apollozhu.github.io/2017/06/20/swift-and-ns-extension-unavailable/
* https://swift-cast.com/2021/04/38/
* https://nshipster.com/available/
* https://davedelong.com/blog/2019/04/09/conditional-compilation-part-3/



