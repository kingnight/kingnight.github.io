---
title: "Be careful with Obj-C bridging in Swift"
description: "Be careful with Obj-C bridging in Swift"
category: programming
tags: iOS,Swift
---

>原文链接：[https://swiftrocks.com/be-careful-with-objc-bridging-in-swift](https://swiftrocks.com/be-careful-with-objc-bridging-in-swift)
>版权归原作者所有，The copyright belongs to the original；
>翻译仅供个人学习，Translation is for personal study only；


# 1. as的两种用途的区别

## 1.1 as 操作符，将类型转换为它继承的父类或者协议

```swift
let myViewController = MyViewController() 
let viewController = myViewController as UIViewController
```

myViewController和viewController之间的功能没有变化，因为操作符所做的只是限制你可以从该类型访问什么。在内部，它们仍然是相同的对象。

## 1.2 as 也是Obj-C桥接操作符

```swift
let string = "MyString" 
let nsstring = string as NSString
```

表面上两者是一样的，但是这个例子与之前的view controller完全不同，String不是从NSString继承，它们是不同的对象，内部不同的实现方式，这种方式能够正常工作是as操作符作为语法糖，相当于：

```swift
let string = "MyString" 
let nsstring: NSString = string._bridgeToObjectiveC()
```

这个方法来自于_ObjectiveCBridgeable协议，它允许对象在需要时自动将Swift类型转换为Objective-C的等价类型，并提供了我们看到的自由的as强制转换行为:

```swift
extension Int8 : _ObjectiveCBridgeable {
    @_semantics("convertToObjectiveC")
    public func _bridgeToObjectiveC() -> NSNumber {
        return NSNumber(value: self)
    }
}
```

这样会有什么问题呢?考虑下面的例子:

```swift
let string = "MyString"
let range = string.startIndex..<string.endIndex

let roundTrip = (string as NSString) as String
roundTrip[range]
```

你认为最后一行会发生什么?

这段代码现在运行得很好，但实际上在Swift 4会导致崩溃!

从Swift的角度来看，这段代码并没有什么问题，因为从技术上讲，将String转换为NSString，再转换回String，在技术上什么都没做。

但从桥接的角度来看，最终的字符串与第一个字符串是不同的对象! "转换"字符串到NSString的行为实际上是创建一个全新的NSString，它有自己的存储，当它被"转换"回String时也会重复创建。这会使范围值与最终字符串不兼容，从而导致崩溃。


## 2 元类型

让我们看一个不同的例子。协议可以通过使用@objc向Obj-C公开，从Swift方面来说，@objc允许元类型被用作Obj-C的协议指针。

```swift
@objc(OBJCProto) protocol SwiftProto { }

let swiftProto: SwiftProto.Type = SwiftProto.self
let objcProto: Protocol = SwiftProto.self as Protocol
// or, from the Obj-C side, NSProtocolFromString("OBJCProto")
```

如果我们比较两个swift元类型，它们通常是相等的:

```swift
ObjectIdentifier(SwiftProto.self) == ObjectIdentifier(SwiftProto.self) // true
```

同样，如果我们向上转换元类型为Any.Type时，条件仍然为真，因为它们仍然是同一个对象:

```swift
ObjectIdentifier(SwiftProto.self as Any.Type) == ObjectIdentifier(SwiftProto.self) // true
```

所以如果，我把它向上转换成别的东西，比如AnyObject，这仍然是正确的，对吧?

```swift
ObjectIdentifier(SwiftProto.self as AnyObject) == ObjectIdentifier(SwiftProto.self) // false
```

不相等，因为我们不再是向上转换了! "强制转换"到AngObject也是一种桥接语法糖，它将元类型转换为Protocol，因为它们不是同一个对象，所以条件不再为真。如果我们直接把它当作Protocol，同样的事情也会发生:

```swift
ObjectIdentifier(SwiftProto.self) == ObjectIdentifier(SwiftProto.self) // true ObjectIdentifier(SwiftProto.self as Protocol) == ObjectIdentifier(SwiftProto.self) // false
```

如果你的Swift方法不能预测它的参数来自哪里，像这样的情况可能会非常令人困惑，因为正如我们上面看到的，同样的对象可以完全改变操作的结果，这取决于它是否桥接。如果这还不够，当你面对同样的方法在不同的语言中有不同的实现时，事情会变得更糟:

```swift
String(reflecting: SwiftProto.self) // __C.OBJCProto
String(reflecting: SwiftProto.self as Any.Type) // __C.OBJCProto
String(reflecting: SwiftProto.self as AnyObject) // Protocol 0x...
String(reflecting: SwiftProto.self as Protocol) // Protocol 0x...
```

尽管从Swift的角度来看，它们看起来都是同一个对象，但当桥接开始时，结果却不一样，因为协议描述的实现与Swift的元类型不同。如果你试图将类型转换为字符串，你需要确保你总是使用它们的桥接版本:

```swift
func identifier(forProtocol proto: Any) -> String {
    // We NEED to use this as an AnyObject to force Swift to convert metatypes
    // to their Objective-C counterparts. If we don't do this, they are treated as
    // different objects and we get different results.
    let object = proto as AnyObject
    //
    if let objcProtocol = object as? Protocol {
        return NSStringFromProtocol(objcProtocol)
    } else if let swiftMetatype = object as? Any.Type {
        return String(reflecting: swiftMetatype)
    } else {
        crash("Type identifiers must be metatypes -- got \(proto) of type \(type(of: proto))")
    }
}
```

如果你不将类型转换为AnyObject，同样的协议可能会给你两个不同的结果，这取决于你的方法是如何被调用的(例如，Swift和Obj-C中提供的参数)。这是最常见的桥接问题的来源，就像几个版本之前NSString的一个类似的例子，与String相比，一个方法有不同的实现，这导致了Swift字符串自动转换为NSString的情况下的问题。

## 结论

我个人认为，使用as作为桥接的语法糖并不是最好的主意。从开发人员的角度来看，很明显string. bridgeToObjectiveC()可能会导致对象改变，而as则表示相反的情况。ObjectiveCBridgeable是一个公共协议，但不支持范型使用。一般来说，要注意实现它的自定义类型，并在进行upcasting时要格外注意，以确保你没有在无意的情况下桥接类型。


# 补充

## Protocol是什么？

Class

**Protocol**
    Framework
- Objective-C Runtime

**Declaration**
    class Protocol

## ObjectIdentifier是什么？

**ObjectIdentifier**

A unique identifier for a class instance or metatype.

**Declaration**

@frozen struct ObjectIdentifier

**Overview**

In Swift, only class instances and metatypes have unique identities. There is no notion概念 of identity for structs, enums, functions, or tuples.