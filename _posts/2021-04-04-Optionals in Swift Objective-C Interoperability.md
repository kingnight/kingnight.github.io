---
title: "Optionals in Swift Objective-C Interoperability"
description: "Optionals in Swift Objective-C Interoperability"
category: programming
tags: iOS,Swift
---

>原文链接：[https://fabiancanas.com/blog/2020/1/9/swift-undefined-behavior.html](https://fabiancanas.com/blog/2020/1/9/swift-undefined-behavior.html)
>版权归原作者所有，The copyright belongs to the original；
>翻译仅供个人学习，Translation is for personal study only；

# 混编代码中Objective-C对象

```
@interface Something : NSObject

@property(nonatomic,nonnull) UIScrollView *scrollView;

@end
```

但是在实现上有一个问题。它是空的。给定下面的实现，一个新实例的滚动视图属性将为nil，这违反了它的接口。编译器不会报错。

```
@implementation Something : NSObject

@end
```

scrollview属性是nonnull(非空的)，在Swift中它不是可选的，在初始化时应该给出初值。那么当我们从Swift中使用时会发生什么呢?

```
let thing: Something = Something()
let scrollView: UIScrollView = thing.scrollView
        
let contentSize = scrollView.contentSize
        
scrollView.flashScrollIndicators()
//Displays the scroll indicators momentarily.
```

已经显式地向scrollView变量添加了类型，以表明它们不是可选的。这些显式类型是编译器推断的类型，如果它们不存在的话。Swift不认为它们是可选的，也不以任何方式将它们视为可选的。

下面继续将会发生什么呢

大多数人认为它会崩溃。它不会崩溃。它实例化一个something 为另一个thing。然后thing的"scrollview"被读取并放入scrollView，并退出没有任何问题。

# 检测nil


假设我们想要防范这种情况。问题是，由于Swift不认为这个值可以为nil，所以检查它并不容易。

## 方式一

比较非可选值会产生一个警告:"在检查可选值时使用的非可选表达式类型'UIScrollView'"。

```
let thing: Something = Something()
guard let scrollView: UIScrollView = thing.scrollView else {
     struct AnonymousError:Error {}
     throw AnonymousError()
}
```
//Initializer for conditional binding must have Optional type, not 'UIScrollView'

## 方式二

```
let thing: Something = Something()
let scrollView: UIScrollView = thing.scrollView
if scrollView == nil {
   print("The compiler says we won't get here.")
   print("But if we run the program, we do")
}
```
//Comparing non-optional value of type 'UIScrollView' to 'nil' always returns false

编译器提示，非可选值不应该与nil比较，它总是false。但是在运行时，nil被检测到，然后打印语句输出。

这些都不理想，因为有警告。我们可以写一个函数来擦掉这个变量的非空属性吗?

```
func isNil(_ o: Any?) -> Bool {
    switch o {
    case .none:
        return true
    case .some(_):
        return false
    }
}

if isNil(scrollView) {
    print("This doesn't print.")
}
//无效
```

```
func isNil(_ o: AnyObject?) -> Bool {
    switch o {
    case .none:
        return true
    case .some(_):
        return false
    }
}

if isNil(scrollView) {
    print("It works if we make it an AnyObject?")
}
//有效
```

我们可以使用像第二个isNil这样的函数来检测、断言并根据这种不寻常的情况调整我们的逻辑。

# Swift扩展（Extension）


如果你对Objective-C类做一个Swift扩展并在这些不应该存在的nil东西上调用它们，那些方法仍然会被调用。

仍然使用出乎意料的nil滚动视图实例，我们可以这样做:

```swift
extension UIScrollView {
    func doAThing() {
        print("doing it") // <- This will get called
    }
}
```

在这些情况下，你可以在对象的实例方法中，将self设为0x0。由于self上的类型是不可选的，静默的意外nil值可以继续传播。

扩展方法为新的意外行为提供了机会。它们执行代码，而不是在不寻常的"Obective-C"模式中运行，在这种模式中，消息nil返回类似于零的值或no-ops。因此，扩展方法可能会有像上面的print语句那样的副作用。它们也可以返回非零值:

```swift
extension UIScrollView {
    func oneHundred() -> Float {
        return 100 // <- Now scrollView.oneHundred() can return 100
    }
}
```

# Foundation Objects


NSCalendar是Foundation中的一个类。因此，如果我们像这样创建一个类，同样带有一个无效的空实现:

```
@interface CalendarProvider : NSObject
@property (nonatomic, nonnull) NSCalendar *calendar;
@end
```

在Swift中这样使用

```swift
let calendarProvider = CalendarProvider()
let calendar = calendarProvider.calendar
let weekStartsOn = calendar.firstWeekday
let weekdays: [String] = calendar.weekdaySymbols
```

**结果跟第一个scollview一样，正常运行？**

程序在第2行崩溃了。这是因为Objective-C的NSCalendar是Swift中的Calendar。但这不是仅仅重命名。它是桥梁到Swift Foundation Calendar类型。

## 补充
![879082ea80a48c66017ea4496c4109ea](/assets/images/截屏2021-04-04 下午8.48.40.png)


**这里发生的事情并不是当我们从calendarProvider中获得一个意外值时崩溃，而是Swift自动将NSCalendar对象的任何实例与C或Objective-C实现转换为Swift Foundation库中的Calendar对象。Swift Foundation库的Calendar有一个方法_unconditionallyBridgeFromObjectiveC，它是_ObjectiveCBridgeable协议的一部分，它将可选的<NSCalendar>转换为Foundation.Calendar。我们可以查看calendar._unconditionallyBridgeFromObjectiveC的源代码。**

```swift
public static func _unconditionallyBridgeFromObjectiveC(_ source: NSCalendar?) -> Calendar {
    var result: Calendar? = nil
    _forceBridgeFromObjectiveC(source!, result: &result)
    return result!
}
```

**有趣的是，桥接函数的参数是一个可选的<NSCalendar>。静态方法，通过它的签名，接受nil。然后发生了什么?在这种情况下，导致崩溃的罪魁祸首是一个force unwrap，它将我们从意想不到的行为中解救出来。尽管传递给函数的值是可选的<NSCalendar>.some(nil)，这仍然不是一个有效值，我们仍然处于未定义的行为领域，所以force unwrap捕捉到这种情况是令人惊喜的。**


# Array Properties


Objective-C中的非null数组属性以一种非常奇怪的方式桥接到Swift中。下面的Objective-C类声明了一个公共的非空NSArray属性。

```
@interface OffendingObject : NSObject
@property (nonnull) NSArray *array;
@end
```

这里添加了一个description方法，以便稍后说明对象的状态。

```
@implementation OffendingObject

- (NSString *)description
{
    return [NSString stringWithFormat:
    @"%@"
    "array: %@",
    [super description],
            self.array];
}

@end
```

当我们尝试用下面的Swift程序与一个有问题的对象交互时会发生什么?记住，对象的数组是桥接到Swift数组属性的。

```swift
let obj = OffendingObject()
print(obj)
print(obj.array)
print(obj)
obj.array.append("thing")
print(obj)
```

输出

```
<OffendingObject: 0x600003c0c230>array: (null)      //1
[]                                              //2
<OffendingObject: 0x600003c0c230>array: (null)      //3
<OffendingObject: 0x600003c0c230>array: (
    thing
)                                              //4
```

程序中至少有两件非常奇怪的事情。第1行和第2行很简单。我们可以实例化一个OffendingObject，并打印它的描述。它的数组属性是用Objective-C给nil的描述格式字符串" (null) "来表示的。

在第三行，奇怪的事情开始发生。print(obj.array)访问Swift中的array属性。这个表达式应该会导致nil，这对Swift来说应该是个问题。相反，它描述了一个空数组，就好像OffendingObject根本没有违反契约一样。如果我们存储obj的值。在这点上，我们确实得到了一个空数组<Any>。它暗示了接下来会发生什么，这非常有趣。

通过打印对象的描述，我们在第4行再次检查对象的属性。它的数组属性仍然是nil。

这种情况看起来并不稳定。在某些情况下，如果Swift在预期的位置找不到数组，它会创建一个数组。

在第5行中，我们向对象数组中添加了一个字符串"thing"。有点令人惊讶的是，它没有崩溃。而OffendingObject以一个似乎凭空变出来的包含值的数组结束。

不过Swift数组和Objective-C数组很有趣。如果我们暂时不考虑新的空数组的来源，那么我们看到的关于数组的其他行为就有意义了。NSArray不能可变（mutated）。但swift数组则不同。从语义上讲，它们是值类型，而不是引用类型。因此，通过添加新元素来更改`var example: Array`与创建新数组并将其赋值给`var example`容器是一样的。改变的是`example`变量，而不是`Array`(数组)。

Swift代码"添加到"数组工作的原因，是因为OffendingObject与这些操作兼容。NSArray是不可变的，但array属性是readwrite。因此代码在属性中获取数组，创建一个带有这些内容和新对象的新数组，然后将新数组存储回OffendingObject的属性中。

这就解决了空数组从哪里来的问题。和上面的NSCalendar一样，NSArray被桥接到Swift数组，我们可以直接看看_unconditionallyBridgeFromObjectiveC的实现。

```swift
static public func _unconditionallyBridgeFromObjectiveC(_ source: NSArray?) -> Array {
    	if let object = source {
        	var value: Array<Element>?
       		 _conditionallyBridgeFromObjectiveC(object, result: &value)
        	return value!
    	} else {
        	return Array<Element>()
    	}
 }
```

这段代码所做的事情与上面的isNil函数非常相似。第2行上的if let 检查正确地识别了nil值，我们跳到第7行，在这里创建并返回一个新的空数组。

# 我们现在怎么办?


-   <https://bugs.swift.org/browse/SR-8622>
-   <https://bugs.swift.org/browse/SR-120>
-   <https://bugs.swift.org/browse/SR-8622?focusedCommentId=39184&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-39184>

# 参考


-   [https://swifter.tips/any-anyobject/](https://swifter.tips/any-anyobject)
-   <https://docs.swift.org/swift-book/LanguageGuide/TypeCasting.html>