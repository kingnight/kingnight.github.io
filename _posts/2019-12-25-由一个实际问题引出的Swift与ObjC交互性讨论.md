---
title: "由一个实际问题引出的Swift与ObjC交互性讨论"
description: "由一个实际问题引出的Swift与ObjC交互性讨论"
category: programming
tags: swift,ios,compiler
---

> 本篇文章由一个项目中的实际问题引出，TableView是常用的基础控件，我们经常在一个基类中实现Tableview的基础代理方法，然后再根据业务需求情况，设计不同的子类继承基类，重写部分代理方法，简化业务实现。我们要探讨的问题就在继承中出现。。。。

# 问题引出

先来看这段代码，是常见的tableView相关的使用场景

```swift
protocol ListDataProtocol {
    
}

class BaseViewController<P: ListDataProtocol>:UIViewController,UITableViewDelegate,UITableViewDataSource {
    var tableView: UITableView
    var presenter: P?
    override func viewDidLoad() {
        //省略非必要实现细节
    }
    
    override init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?) {
        //省略非必要实现细节
    }
    
    required init?(coder: NSCoder) {
        //省略非必要实现细节
    }
    
    //MARK: UITableViewDataSource
    // cell的个数
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return 10
    }
    // UITableViewCell
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        //省略非必要实现细节
    }
      
    //MARK: UITableViewDelegate
    // 设置cell高度
    func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        return 44.0
    }
}

class ListData: NSObject, ListDataProtocol{
    
}

class ViewController: BaseViewController<ListData> {
    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view.
    }

    // 选中cell后执行此方法
   func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        print(indexPath.row)
   }
}
```

首先我们简单了解一下这段代码，基类BaseViewController包含iOS中UITableView的DataSource和Delegate的基础实现，子类ViewController继承基类，由于基类中不知道如何处理具体的业务细节，所以只包含必须的代理实现，而其他可选的代理实现，预期由子类实现，所以在子类ViewController中实现了`table(_:didSelectRowAt:)`，用于处理Cell选中事件，那么问题来了，请问执行这段代码时，点击cell是否会如预期输出`print(indexPath.row)`

![由一个实际问题引出的Swift与ObjC交互性讨论](/assets/images/u=562522174,1600702944&fm=26&gp=0.jpg)

答案是：没有输出，`table(_:didSelectRowAt:)`不会被调用执行，那么为什么呢？

显然我们查找文档和Google都不能很容易找到答案，所以想到了利用Swift提供的中间语言SIL（Swift Intermediate Language），如果你不了解SIL，建议首先阅读 [Swift Intermediate Language 初探](wwww)


# 生成SIL

```Shell
swiftc -emit-silgen -target x86_64-apple-ios13.0-simulator -sdk $(xcrun --show-sdk-path --sdk iphonesimulator)  -Onone test.swift > test.swift.sil
```

* 由于我们需要使用到UIKit这类系统库，所以需要使用-sdk参数指定依赖的SDK路径
* -target 指定生成代码对应的target
* -Onone 不进行任何优化


# SIL分析

## thunk函数

[thunk函数](https://en.wikipedia.org/wiki/Thunk_%28programming%29)，是一个不得不提的概念，我借用[阮一峰老师博客](http://www.ruanyifeng.com/blog/2015/05/thunk.html)一段话来说明：
>编程语言刚刚起步时，编译器怎么写比较好。一个争论的焦点是["求值策略"](https://zh.wikipedia.org/wiki/%E6%B1%82%E5%80%BC%E7%AD%96%E7%95%A5)，即函数的参数到底应该何时求值。

```javascript
var x = 1;

function f(m){
  return m * 2;     
}

f(x + 5)
```
>一种意见是["传值调用"](https://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_value)（call by value），即在进入函数体之前，就计算 x + 5 的值（等于6），再将这个值传入函数 f 。C语言就采用这种策略。
```
f(x + 5)
// 传值调用时，等同于
f(6)
```
>另一种意见是["传名调用"](https://zh.wikipedia.org/wiki/%E6%B1%82%E5%80%BC%E7%AD%96%E7%95%A5#.E4.BC.A0.E5.90.8D.E8.B0.83.E7.94.A8_.28Call_by_name.29)（call by name），即直接将表达式 x + 5 传入函数体，只在用到它的时候求值。Hskell语言采用这种策略。
```
f(x + 5)
// 传名调用时，等同于
(x + 5) * 2
```
>传值调用和传名调用，哪一种比较好？回答是各有利弊。传值调用比较简单，但是对参数求值的时候，实际上还没用到这个参数，有可能造成性能损失。
>
>**编译器的"传名调用"实现，往往是将参数放到一个临时函数之中，再将这个临时函数传入函数体。这个临时函数就叫做 Thunk 函数。**

Swift SIL当中的thunk函数的基本思想与上面描述的是一致的，但是作用稍有不同，我们来看文章开头这个例子中，`BaseViewController.viewDidLoad`函数对应的SIL片段，借此了解thunk函数的作用。

```swift
// BaseViewController.viewDidLoad()
sil hidden [ossa] @$s4test18BaseViewControllerC11viewDidLoadyyF : $@convention(method) <P where P : ListDataProtocol> (@guaranteed BaseViewController<P>) -> () {
// %0                                             // users: %89, %88, %87, %80, %79, %78, %72, %71, %53, %11, %10, %2, %1
bb0(%0 : @guaranteed $BaseViewController<P>):
//暂时省略无关代码

bb1:                                              // Preds: bb0
//暂时省略无关代码

// %70                                            // users: %77, %75, %74
bb2(%70 : @owned $UIView):                        // Preds: bb0
//暂时省略无关代码
} // end sil function '$s4test18BaseViewControllerC11viewDidLoadyyF'

// @objc BaseViewController.viewDidLoad()
sil hidden [thunk] [ossa] @$s4test18BaseViewControllerC11viewDidLoadyyFTo : $@convention(objc_method) <P where P : ListDataProtocol> (BaseViewController<P>) -> () {
// %0                                             // user: %1
bb0(%0 : @unowned $BaseViewController<P>):
//暂时省略无关代码
  // function_ref BaseViewController.viewDidLoad()
  %3 = function_ref @$s4test18BaseViewControllerC11viewDidLoadyyF : $@convention(method) <τ_0_0 where τ_0_0 : ListDataProtocol> (@guaranteed BaseViewController<τ_0_0>) -> () // user: %4
  %4 = apply %3<P>(%2) : $@convention(method) <τ_0_0 where τ_0_0 : ListDataProtocol> (@guaranteed BaseViewController<τ_0_0>) -> () // user: %7
//暂时省略无关代码                             // id: %7
} // end sil function '$s4test18BaseViewControllerC11viewDidLoadyyFTo'

```
由于篇幅所限，这段代码中省略了一些解释问题非必要的代码，如果需要查看完整代码请参见文末下载地址。

你会发现生成了两段`viewDidLoad`，第一段是一个原生的Swift函数，使用`convention(method)`方式调用和处理，它内部有三个block块，分别实现一些具体的功能：bb0完成一些变量的分配和初始化；bb1完成代码诊断相关工作；bb2完成BaseViewController中TableView的Delegate和DataSource的设置，这三部分构成了完整的`viewDidLoad`函数。

第二段，它是一个thunk函数，它的标识id是`@$$s4test18BaseViewControllerC11viewDidLoadyyFTo`，对比一下第一段id是`@$s4test18BaseViewControllerC11viewDidLoadyyF`，第二段的id是在第一段的基础上增加了两个单词**To**，并且通过@convention(objc_method)可以看到，它是使用ObjC方式调用的；它的内部使用Swift原生方式调用第一段的函数；

整个流程如下图所示：
![由一个实际问题引出的Swift与ObjC交互性讨论](/assets/images/SIL交互图-1.png)


简单总结一下，在SIL中生成了一个新的thunk函数，这个函数用于暴露给ObjC进行交互，而thunk函数内部会调用真正的原生Swift函数。那么也就是说，如果一个函数需要被ObjC可见，需要包装为一个支持ObjC方式调用的thunk函数。


## Swift消息派发方式

简单来说，Swift有两种消息派发方式：

* 静态派发(static dispatch),静态派发在执行的时候，会直接跳到方法的实现，静态派发可以进行inline和其他编译期优化,值类型总是会使用静态派发，因为不存在继承可变的可能；
* 动态派发(dynamic dispatch)，动态派发在执行的时候，会根据运行时(Runtime)，采用table的方式，找到方法的执行体，然后执行。动态派发也就没有办法像静态派发那样，进行编译期优化。

有些文章还会提到，第三种派发方式，Objective-C派发，其实Swift代码当中的这种方式的本质就是我们前面提到的thunk函数机制，它涉及两个Swift关键字@objc和dynamic：@objc意味着你的Swift代码，如类，方法，属性，将会被Objective-C可见；dynamic意味着你要是用Objective-C动态派发；

但是这两个参数其实并不是单独使用的，如果使用了dynamic就必须添加@objc，但是反过来可以单独@objc，区分开来使得每个关键字的含义更清晰。使用@objc和dynamic组合生成的函数，采用Objective-C派发，不会出现在SIL的VTable表中的；而单独添加@objc的函数是会出现在VTable表中的， 采用的Swift动态派发。

另外，需要提到的是@objc可见性，在Swift4以前，从NSObject继承的类中的方法是由编译器自动暴露给Objective-C可见，但是从Swift4以后，需要手动添加@objc明确指明，编译器不再进行推断。反映到SIL也是一致的，如果生成的函数标记为@objc才能被Objective-C可见。

关于VTable和Witness Table如果不很了解，建议还是先阅读[Swift Intermediate Language 初探](www)


# 答案探寻

有了前面的基础，我们来探索答案，把焦点定位到`table(_:didSelectRowAt:)`关键字

```SWIFT
sil_vtable BaseViewController {
//暂时省略无关代码
  #BaseViewController.tableView!1: <P where P : ListDataProtocol> (BaseViewController<P>) -> (UITableView, Int) -> Int : @$s4test18BaseViewControllerC05tableC0_21numberOfRowsInSectionSiSo07UITableC0C_SitF	// BaseViewController.tableView(_:numberOfRowsInSection:)
  #BaseViewController.tableView!1: <P where P : ListDataProtocol> (BaseViewController<P>) -> (UITableView, IndexPath) -> UITableViewCell : @$s4test18BaseViewControllerC05tableC0_12cellForRowAtSo07UITableC4CellCSo0jC0C_10Foundation9IndexPathVtF	// BaseViewController.tableView(_:cellForRowAt:)
  #BaseViewController.tableView!1: <P where P : ListDataProtocol> (BaseViewController<P>) -> (UITableView, IndexPath) -> CGFloat : @$s4test18BaseViewControllerC05tableC0_14heightForRowAt12CoreGraphics7CGFloatVSo07UITableC0C_10Foundation9IndexPathVtF	// BaseViewController.tableView(_:heightForRowAt:)
  #BaseViewController.deinit!deallocator.1: @$s4test18BaseViewControllerCfD	// BaseViewController.__deallocating_deinit
}

sil_vtable ViewController {
//暂时省略无关代码
  #BaseViewController.tableView!1: <P where P : ListDataProtocol> (BaseViewController<P>) -> (UITableView, Int) -> Int : @$s4test18BaseViewControllerC05tableC0_21numberOfRowsInSectionSiSo07UITableC0C_SitF [inherited]	// BaseViewController.tableView(_:numberOfRowsInSection:)
  #BaseViewController.tableView!1: <P where P : ListDataProtocol> (BaseViewController<P>) -> (UITableView, IndexPath) -> UITableViewCell : @$s4test18BaseViewControllerC05tableC0_12cellForRowAtSo07UITableC4CellCSo0jC0C_10Foundation9IndexPathVtF [inherited]	// BaseViewController.tableView(_:cellForRowAt:)
  #BaseViewController.tableView!1: <P where P : ListDataProtocol> (BaseViewController<P>) -> (UITableView, IndexPath) -> CGFloat : @$s4test18BaseViewControllerC05tableC0_14heightForRowAt12CoreGraphics7CGFloatVSo07UITableC0C_10Foundation9IndexPathVtF [inherited]	// BaseViewController.tableView(_:heightForRowAt:)
  #ViewController.tableView!1: (ViewController) -> (UITableView, IndexPath) -> () : @$s4test14ViewControllerC05tableB0_14didSelectRowAtySo07UITableB0C_10Foundation9IndexPathVtF	// ViewController.tableView(_:didSelectRowAt:)
  #ViewController.deinit!deallocator.1: @$s4test14ViewControllerCfD	// ViewController.__deallocating_deinit
}
```
由于基类BaseViewController中没有实现`table(_:didSelectRowAt:)`，所以基类sil_vtable表中没有对应函数签名，这很正常；子类ViewController有实现，所以出现`table(_:didSelectRowAt:)`，但是请注意它不是一个thunk函数。

对比如下图所示：
![由一个实际问题引出的Swift与ObjC交互性讨论](/assets/images/SIL交互图-2.png)


再看`table(_:didSelectRowAt:)`的实现部分SIL

```swift
// ViewController.tableView(_:didSelectRowAt:)
sil hidden [ossa] @$s4test14ViewControllerC05tableB0_14didSelectRowAtySo07UITableB0C_10Foundation9IndexPathVtF : $@convention(method) (@guaranteed UITableView, @in_guaranteed IndexPath, @guaranteed ViewController) -> () {
// %0                                             // user: %3
// %1                                             // users: %13, %4
// %2                                             // user: %5
bb0(%0 : @guaranteed $UITableView, %1 : $*IndexPath, %2 : @guaranteed $ViewController):
  //省略内部实现
} // end sil function '$s4test14ViewControllerC05tableB0_14didSelectRowAtySo07UITableB0C_10Foundation9IndexPathVtF'
```

首先没有@objc的标记，没有thunk的标记，所以根据前面的知识可得，虽然ViewController实现了`table(_:didSelectRowAt:)`，但是这个函数在生成的SIL中是一个纯粹的原生Swift函数，编译器没有帮助它生成对应的Objective-C可见的thunk函数，所以无法被UITabview回调使用，到此我们已经回答了开头的问题。

那么为什么编译器不去生成呢？什么情况下编译器会生成thunk函数呢？

# 对比试验

我们先来做两个对比试验：

## Test1
开篇问题的代码中，如果你删除跟presenter泛型相关的代码，删除ListDataProtocol协议，重新运行一下，结果。。。。`print(indexPath.row)`正常输出了，怎么解释呢？？？

我们还是借助SIL语言，分析去掉前述内容后生成的SIL：
* ViewController的sil_vtable表中有`table(_:didSelectRowAt:)`与当前一致；
* 除原始的`table(_:didSelectRowAt:)`实现外，生成了带@objc标记的thunk函数；
* BaseViewController和ViewController都被直接标记为@objc（没有泛型，都是基于UIKit，需要依赖Objective-C runtime，所以可以被默认推断Objective-C可见）

## Test2
还是开篇问题的代码中，如果在基类实现`table(_:didSelectRowAt:)`，在子类修改`table(_:didSelectRowAt:)`添加`override`，重新运行一下，执行结果：`print(indexPath.row)`正常输出

查看修改后生成的SIL：
* 基类BaseViewController和子类的ViewController各自的sil_vtable中，都有`table(_:didSelectRowAt:)`，且子类中对应方法标记为`[override]`
* 除原始的`table(_:didSelectRowAt:)`实现外，生成了带@objc标记的thunk函数；
* BaseViewController和ViewController没有被标记@objc

对比一下，相同点，都生成了thunk函数，所以结果能够正常输出！不同点，没有泛型情况，编译器默认推断生成Objc可见；有泛型情况，只有基类实现对应方法，子类的实现方法才会被编译器推断生成thunk函数。

# 小结

查阅[SIL的官方文档](https://github.com/apple/swift/blob/master/docs/SIL.rst)有这样一段话:

>If a derived class conforms to a protocol through inheritance from its base class, this is represented by an inherited protocol conformance, which simply references the protocol conformance for the base class.

简单翻译一下：

如果派生类通过从其基类继承而符合协议，这种称为继承协议一致性表示（inherited protocol conformance），**它会简单引用基类的协议实现**

这句话就可以解释我们的问题了，设计编译器时在这种场景下，它只会简单引用基类的实现，所以基类中必须有`table(_:didSelectRowAt:)`，子类的`table(_:didSelectRowAt:)`才会被可见！


# 猜想
在文章的末尾，我想再提出一个问题，我们大胆猜想一下为什么编译器设计为**简单引用基类的协议实现**，我认为是编译效率的考量！以Tableview为例，它的DataSource和Delegate中绝大部分是可选的实现，如果编译器不按照现在的逻辑处理，就需要层层遍历所有子类，查找所有可能的代理函数实现，然后把他们都转成thunk函数，才能实现文章开头的预期效果，显然是一个十分耗时的过程，并且这样操作就如同swift调整@objc的推断一样，是在代替使用者做决策。

如有错误欢迎批评指正！

**所有源文件**[下载地址](https://github.com/kingnight/Swift-ObjC-interaction-SIL)

# 参考

* https://en.wikipedia.org/wiki/Thunk
* https://swiftunboxed.com/interop/objc-dynamic/
* https://github.com/apple/swift-evolution/blob/master/proposals/0160-objc-inference.md
* https://useyourloaf.com/blog/objc-warnings-upgrading-to-swift-4/
* https://medium.com/@slavapestov/how-to-talk-to-your-kids-about-sil-type-use-6b45f7595f43