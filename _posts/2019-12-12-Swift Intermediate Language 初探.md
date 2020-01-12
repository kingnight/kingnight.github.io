---
title: "Swift Intermediate Language 初探"
description: "Swift语言是构建在LLVM架构上，标准的三段式设计，并且使用LLVM中间表示(IR)和LLVM不同后端生成机器语言，除此之外，Swift编译器在生成IR之前，还设计了一门新的高级中间语言Swift Intermediate Language，简称SIL，SIL会对Swift进行高级别的语意分析和优化，我们通过解读SIL代码就能去了解Swift背后的一些实现细节，帮助我们理解一些问题"
category: programming
tags: swift, ios, compiler
---
# 引言

通常，当我们学习一门高级编程语言时，更多专注于它的语法特性，关键字如何使用，是否是面向对象编程，很少关注其背后的编译器是如何处理高级语言，把它转换成低级语言，甚至是机器语言的过程。但是当你遇到一些无法理解的表面现象，没有办法通过文档或者前人经验解读问题的实质，想要尝试探究编程语言背后的实现细节，那么就需要跨越语言本身的了解，向下去专研编译器的实现细节；但是编译器的构成和实现细节往往十分复杂和繁杂，令人生畏，阻碍了人们的学习热情。

Swift语言是构建在LLVM架构上，标准的三段式设计，并且使用LLVM中间表示(IR)和LLVM不同后端生成机器语言，除此之外，Swift编译器在生成IR之前，还设计了一门新的高级中间语言Swift Intermediate Language，简称SIL，SIL会对Swift进行高级别的语意分析和优化，我们通过解读SIL代码就能去了解Swift背后的一些实现细节，帮助我们理解一些问题。

本篇文章，首先简要介绍LLVM架构的组成，了解SIL优化过程位于其中哪个环节；然后通过一个典型的，带有协议实现的例子解读生成的SIL文件，探究Swift语言背后的实现细节。

# LLVM

![2a60bf12d3314b9b6d44a1a2f1563082](/assets/images/LLVMCompiler1.png)

上图是[LLVM](http://llvm.org/)标准的三段式设计架构图，它分为前端，中间优化器，后端；前端负责解析、验证和诊断输入代码中的错误，然后将解析后的代码转换为LLVM Intermediate Representation（简称LLVM IR），LLVM IR被设计用来承载在编译器的优化器部分中可以找到的中级分析和转换。它的设计考虑到了许多特定的目标，包括支持轻量级运行时优化、跨功能/过程间优化、整个程序分析和积极的重构转换，等等。这些分析和优化传递可以改进代码，然后发送到后端代码生成器中生成真实机器码。

![592028397a6f731ba2f299f23dd5afa9](/assets/images/swiftc_pipeline.png)

Swift语言也是基于LLVM架构的，你可以看到与现有Objective-C语言有很多相似之处，有相同的后端结构，前端的工作流程也基本类似，词法分析，标记，构建AST，类型检查，主要区别在于生成IR之前，AST之后，增加了SIL，它是AST和LLVM IR之间的另一种中间代码表示形式，这么做的目的是希望弥补一些Clang编译器的缺陷，如无法执行一些高级分析，可靠的诊断和优化，而AST和LLVM IR都不是合适的选择。因此，SIL应运而生，用来解决现有的缺陷。

# 问题
Swift语言中很重要的一个特性是protocol extension，通过这特性可以为类型添加默认实现，我们下面就来看一个具体的例子：
```swift
protocol ListDataProtocol {
    func testFunc()   //#1 此行重点
}

class TestClass:ListDataProtocol {
    func testFunc() {
        print("ahaha")
    }
}

extension ListDataProtocol{
    func testFunc() {
        print("bhbhb")
    }
}

let obj:TestClass = TestClass()
obj.testFunc()

let obj2:ListDataProtocol = TestClass()
obj2.testFunc()

```
[demo源码下载地址](https://gist.github.com/kingnight/bd22fc50798f5c5eafc313dd8f191d7a)

首先请读者来回答几个问题：

* 上面这段代码的运行结果是什么？
* 请问如果标记为#1的协议中函数定义testFunc，存在和被注释掉时，输出结果是否相同？
* 如果输出结果不相同，那么你能解释为什么不同吗？

首先揭晓答案，输出结果

* 协议中存在函数定义
```swift
ahaha
ahaha
```

* 协议中注释函数定义
```swift
ahaha
bhbhb
```
# SIL要点

在开始使用SIL语言具体分析问题之前，我们先来通过一些要点，简单的了解一下SIL的特点：

* SIL依赖于swift的类型系统和声明，所以SIL语法是swift的延伸。一个.sil文件是一个增加了SIL定义的swift源文件。
* sil文件中没有隐式import，如果使用swift或者Buildin标准组件的话必须明确的引入
* SIL的类型是通过“\$”符号进行标记的。SIL的类型系统和swift的密切相关。所以“\$”之后的类型会根据swift的类型语法进行语法分析。
* SIL函数由一个或多个block组成，一个block是一个指令的线性序列，每个block中的最后一条指令将控制转移到另一个block，或从函数返回。

# 问题分析

下面我们使用命令行把上面问题这段代码生成对应的SIL文件
```shell
swiftc -emit-silgen  -Onone demo.swift > demo.swift.sil
```
-Onone 意味着告诉编译器不做任何优化，能够方便我们更好的对比源码和生成的SIL之间的关系

*接下来我们从头开始一段一段的分析*

## 头部声明和变量
```swift
sil_stage raw

import Builtin
import Swift
import SwiftShims

protocol ListDataProtocol {
  func testFunc()
}

class TestClass : ListDataProtocol {
  func testFunc()
  init()
  @objc deinit
}

extension ListDataProtocol {
  func testFunc()
}
```
stage raw表明是SIL的第一个阶段，这个阶段会进行基础的Guaranteed Optimization（担保优化）和语法诊断；在后续的第二个阶段会进行General SIL Optimization，生成canonical SIL（正式SIL），最后由IRGen将前一步的结构降级为LLVM IR。由这几个连续的步骤构成一个完整的编译过程。

下面继续看变量的定义部分
```swift
@_hasStorage @_hasInitialValue let obj: TestClass { get }

@_hasStorage @_hasInitialValue let obj2: ListDataProtocol { get }

// obj
sil_global hidden [let] @$s4demo3objAA9TestClassCvp : $TestClass

// obj2
sil_global hidden [let] @$s4demo4obj2AA16ListDataProtocol_pvp : $ListDataProtocol
```
sil_global标记变量为全局变量，hidden标记只针对同一个Swift模块中的对象可见，这里明确定义了后面我们需要使用的两个对象obj和obj2

## Main函数

SIL语言中Main函数依然是程序的入口
```swift
// main
sil [ossa] @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
```
* // main 注释代码
* @main 表示方法的名字是main，SIL中所有标识符均以@符号开头；@convention(c) 很重要，它表示使用C函数的方式进行调用，convention这个标识用于明确指定当函数调用时参数和返回值应该如何被处理，除了这种方式还有：
@convention(swift) 纯Swift函数的默认调用方式
@convention(method) [柯里化](https://zh.wikipedia.org/zh/%E6%9F%AF%E9%87%8C%E5%8C%96)的函数调用方式
@convention(witness_method) 协议方法调用，它等同于convention(method)除了在处理范型类型参数时
@convention(objc_method) Objective-C方式调用
* bb0=basic block zero，表示一个代码块，SIL中没有分支语句，只有入口和出口；%0： 传入basic block的第一个参数是int32类型，%符号表示一个寄存器，由于SIL类似LLVM IR是[Static Single Assignment \(SSA\)](https://en.wikipedia.org/wiki/Static_single_assignment_form)IR，它类似常量，一旦赋值之后就不能被修改，如果再需要新的寄存器，就增加寄存器编号，这样操作有利于编译器的优化；后续进行降级操作时，才会把这些带编号的虚拟寄存器转换成对应体系结构的真实寄存器。

```swift
  alloc_global @$s4demo3objAA9TestClassCvp        // id: %2
  %3 = global_addr @$s4demo3objAA9TestClassCvp : $*TestClass // users: %8, %7
  %4 = metatype $@thick TestClass.Type            // user: %6
```
* alloc_global 初始化全局变量的内存；// id: %2 这种类似注释语句在SIL中称为定位注释，标记id为2
* global_addr  创建上一步全局变量的地址引用，赋值给寄存器%3，// users: %8, %7  表示会被id为8，7的代码用到
* metatype 获得元类型，@thick描述元类型代表的形式，是引用一个对象类型或是其子类；还有@thin代表一个确切的具体类型，不需要存储；@objc代表使用Objective-C类对象表示，而不是纯Swift类型对象表示

```swift
  // function_ref TestClass.__allocating_init()
  %5 = function_ref @$s4demo9TestClassCACycfC : $@convention(method) (@thick TestClass.Type) -> @owned TestClass // user: %6
  %6 = apply %5(%4) : $@convention(method) (@thick TestClass.Type) -> @owned TestClass // user: %7
  store %6 to [init] %3 : $*TestClass             // id: %7
```
* function_ref 创建函数引用，这个被调用函数是前面提到的[柯里化](https://zh.wikipedia.org/zh/%E6%9F%AF%E9%87%8C%E5%8C%96)的函数，@owned 代表函数接收者负责销毁返回值
* apply 调用函数，传入参数是寄存器%4，这两句综合起来，就是创建函数引用，然后调用函数，传入参数，执行完返回结果存储在寄存器%6
* store 存储寄存器%6值到内存地址%3

## VTable和Witness Table

在继续分析main函数后半部分之前，我们看先来看SIL文件中两张表VTable和Witness Table。

### VTable
VTable用来表示类对象方法的动态派发（dynamic dispatch），如果看到SIL代码中出现class_method 和 super_method，这些都是通过sil_vtable进行追踪的；sil_vtable中包含类对象中每个方法的映射，包括从父对象继承的方法；

举个例子：

```SWIFT
class A {
  func foo()
  func bar()
}

sil @A_foo : $@convention(thin) (@owned A) -> ()
sil @A_bar : $@convention(thin) (@owned A) -> ()

sil_vtable A {
  #A.foo!1: @A_foo
  #A.bar!1: @A_bar
}

class B : A {
  func bar()
}

sil @B_bar : $@convention(thin) (@owned B) -> ()

sil_vtable B {
  #A.foo!1: @A_foo
  #A.bar!1: @B_bar
}
```
A类中两个方法foo和bar，对应生成的sil_vtable A中A_foo和A_bar，当B继承A时，重写了bar方法，那么sil_vtable B中从A继承的A_bar方法就会被B_bar替换，这种在SIL的虚函数中查找衍生类的重载方法，由Swift的AST负责。同时为了避免SIL方法是[thunks](https://en.wikipedia.org/wiki/Thunk_%28programming%29)，方法名是连接在原始方法实现之前，如#A.bar!1: @B_bar

### Witness Table

* Witness Table用来代表泛型类型方法的动态派发，一个泛型类型的所有的泛型实例共享一个Witness Table，同样衍生类也都会继承基类的Witness Table。
* 每一个遵循协议（protocol conformance）的对象都有一个唯一标识，会生成一张Witness Table

回过头看看要解决问题的这个对象TestClass，它生产的两张表
```swift
sil_vtable TestClass {
  #TestClass.testFunc!1: (TestClass) -> () -> () : @$s4demo9TestClassC8testFuncyyF	// TestClass.testFunc()
  #TestClass.init!allocator.1: (TestClass.Type) -> () -> TestClass : @$s4demo9TestClassCACycfC	// TestClass.__allocating_init()
  #TestClass.deinit!deallocator.1: @$s4demo9TestClassCfD	// TestClass.__deallocating_deinit
}

sil_witness_table hidden TestClass: ListDataProtocol module demo {
  method #ListDataProtocol.testFunc!1: <Self where Self : ListDataProtocol> (Self) -> () -> () : @$s4demo9TestClassCAA16ListDataProtocolA2aDP8testFuncyyFTW	// protocol witness for ListDataProtocol.testFunc() in conformance TestClass
}
```
sil_vtable 中包含了所有函数，包括我们没有手动实现的init和deinit，编译器帮我们自动生成了，再看sil_witness_table，了解问题真相的时刻快要到了！
如果你对比过protocol ListDataProtocol 中是否注释掉func testFunc()这行生成的SIL有什么不同？就会发现，很明显的一点是sil_witness_table ListDataProtocol中是否有ListDataProtocol.testFunc会生成，如果注释掉func testFunc()，sil_witness_table表中内容为空，那么就说明sil_witness_table中的函数生成以协议中是否声明该函数为准，即使你在协议扩展中实现了函数，如果没有在协议定义中声明函数，就不会在Witness Table中生成对应函数签名

## 函数调用
我们再来对比一下是否注释掉func testFunc()生成的SIL在调用testFunc()方法的区别：
* 协议无函数定义
```SWIFT
  %19 = open_existential_addr immutable_access %13 : $*ListDataProtocol to $*@opened("1F8DEA44-060A-11EA-A00A-ACDE48001122") ListDataProtocol // users: %21, %21
  // function_ref ListDataProtocol.testFunc()
  %20 = function_ref @$s4demo16ListDataProtocolPAAE8testFuncyyF : $@convention(method) <τ_0_0 where τ_0_0 : ListDataProtocol> (@in_guaranteed τ_0_0) -> () // user: %21
  %21 = apply %20<@opened("1F8DEA44-060A-11EA-A00A-ACDE48001122") ListDataProtocol>(%19) : $@convention(method) <τ_0_0 where τ_0_0 : ListDataProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %19
```

* 协议有函数定义
```SWIFT
%19 = open_existential_addr immutable_access %13 : $*ListDataProtocol to $*@opened("37F31910-060A-11EA-BC47-ACDE48001122") ListDataProtocol // users: %21, %21, %20
  %20 = witness_method $@opened("37F31910-060A-11EA-BC47-ACDE48001122") ListDataProtocol, #ListDataProtocol.testFunc!1 : <Self where Self : ListDataProtocol> (Self) -> () -> (), %19 : $*@opened("37F31910-060A-11EA-BC47-ACDE48001122") ListDataProtocol : $@convention(witness_method: ListDataProtocol) <τ_0_0 where τ_0_0 : ListDataProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %19; user: %21
  %21 = apply %20<@opened("37F31910-060A-11EA-BC47-ACDE48001122") ListDataProtocol>(%19) : $@convention(witness_method: ListDataProtocol) <τ_0_0 where τ_0_0 : ListDataProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %19
```

* open_existential_addr 获得existential container内的具体值

* 协议有函数定义：witness_method 通过查sil_witness_table表获得函数指针，这行也是一个thunk函数，通过“：”分割好几个部分，前面提到它的规则是方法名是连接在原始方法实现之前，所以调用的是@convention(witness_method: ListDataProtocol)，通过查找关键字witness_method: ListDataProtocol，你能看到witness的函数实现。

* 协议无函数定义：使用function_ref直接调用ListDataProtocol.testFunc，所以执行的是extension中的testFunc实现

```swift
// protocol witness for ListDataProtocol.testFunc() in conformance TestClass
sil private [transparent] [thunk] [ossa] @$s4demo9TestClassCAA16ListDataProtocolA2aDP8testFuncyyFTW : $@convention(witness_method: ListDataProtocol) (@in_guaranteed TestClass) -> () {
// %0                                             // user: %1
bb0(%0 : $*TestClass):
  %1 = load_borrow %0 : $*TestClass               // users: %5, %3, %2
  %2 = class_method %1 : $TestClass, #TestClass.testFunc!1 : (TestClass) -> () -> (), $@convention(method) (@guaranteed TestClass) -> () // user: %3
  %3 = apply %2(%1) : $@convention(method) (@guaranteed TestClass) -> ()
  %4 = tuple ()                                   // user: %6
  end_borrow %1 : $TestClass                      // id: %5
  return %4 : $()                                 // id: %6
} // end sil function '$s4demo9TestClassCAA16ListDataProtocolA2aDP8testFuncyyFTW'
```

此函数是witness_method: ListDataProtocol的实现，内部使用class_method查找和调用，最终指向sil_vtable中的方法，调用到#TestClass.testFunc

从上面的分析，我们可以看到Witness Table是指向一个witness的函数的调用，不是做一个引用，而这个witness的函数调用真正实现协议的方法


**完整SIL文件源码下载**： 

* [协议无函数定义](https://gist.github.com/kingnight/ede0443a66f25824f1d72360bb07b77a)
* [协议有函数定义](https://gist.github.com/kingnight/16fb074fd78d9872c4dbec0b414493f4)

## 小结

经过前面的分析我们能得出下面的结论：遵循协议的对象，如果协议有函数定义且对象实现函数定义，则实例化的协议对象，调用协议方法时，执行对象中的实现；反之，会执行扩展中的默认实现；

# 结语
这篇文章中我们通过一个典型的，带有协议实现的例子，利用生成的SIL文件解读语言背后的实现逻辑，帮助我们更好的理解编译器内部语言的实现细节，解答一些表面上似懂非懂的困惑。

另外，我们也看到读懂SIL并非十分困难，没有汇编那么晦涩难懂，它与Swift的语法有很多相似之处，我们可以在遇到一些疑问时考虑一下这个工具，让它成为我们的小帮手，更好的理解语言背后的实现细节！

# 参考

* https://github.com/apple/swift/blob/master/docs/SIL.rst
* https://www.jianshu.com/p/c2880460c6cd
* https://benng.me/2017/08/27/high-level-sil-optimization-in-the-swift-compiler/
* https://gpake.github.io/2019/03/06/tryToReadSIL/#more
* https://medium.com/@slavapestov/how-to-talk-to-your-kids-about-sil-type-use-6b45f7595f43
* https://github.com/apple/swift-evolution/blob/master/proposals/0160-objc-inference.md
* https://dmtopolog.com/code-optimization-for-swift-and-objective-c/


