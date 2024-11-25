---
title: "Swift Macro系列1-基础篇"
description: "欢迎来到 Swift 宏的世界。我们将探索 Swift 5.9 版本中引入的一项革命性特性——宏（Macro），它为我们提供了一种在编译时处理源代码的能力。宏可以帮助我们自动化代码生成，减少模板代码，并使得代码更加简洁和易于理解。"
category: macro
tags: Swift Macro,attached macro,freestanding macro,SwiftSyntax
---

欢迎来到 Swift 宏的世界。我们将探索 Swift 5.9 版本中引入的一项革命性特性——宏（Macro），它为我们提供了一种在编译时处理源代码的能力。宏可以帮助我们自动化代码生成，减少模板代码，并使得代码更加简洁和易于理解。我们将从宏的基本概念开始，逐步深入了解关联宏（attached macro）和独立宏（freestanding macro）的区别与应用。接着，我们会学习如何通过 SwiftSyntax 库来解析和操作 Swift 代码的抽象语法树（AST），这是实现宏功能的基础。

在这篇文档中，我们将遵循以下步骤来学习 Swift 宏和 SwiftSyntax：

1. 理解宏的基本概念：我们将从宏的定义和作用开始，了解它如何在编译时转换源代码，以及如何通过宏避免编写重复的代码。
2. 探索宏的分类：我们会区分关联宏和独立宏，并探讨它们各自的使用场景和特点。
3. 学习宏的实现：通过具体的代码示例，我们将学习如何实现一个宏，包括宏的声明和实现的分离，以及如何遵守相应的协议。
4. 深入 SwiftSyntax：我们将探索 SwiftSyntax 库，了解它如何提供操作 Swift 源代码的高级 API，并学习如何使用这些工具来解析和生成代码。
  

通过这篇文档的学习，你将能够掌握 Swift 宏的基础知识，了解如何在你的项目中有效地使用宏，以及如何通过 SwiftSyntax 来增强你的代码处理能力。让我们开始这段旅程，一起探索 Swift 宏的奥秘。

# 宏Macro是什么

宏在编译时会将你的源代码进行转换，这样你就可以避免手动编写重复的代码。在编译过程中，Swift会在通常的代码构建之前将代码中的宏展开。**扩展宏总是一种增量操作：宏会添加新的代码，但不会删除或修改现有的代码。**

宏的输入和宏展开后的输出都会被检查，以确保它们是语法上正确的 Swift 代码。同样，您传递给宏的值以及由宏生成的代码中的值也会被检查，以确保它们具有正确的类型。此外，如果宏的实现在展开宏时遇到错误，编译器会将其视为编译错误。这些保证使得更易于理解使用了宏的代码，并且使得更易于识别诸如错误地使用宏或存在 bug 的宏实现等问题。

苹果按照宏不同的使用场景，将宏分成两个大类：关联宏（attached macro）和独立宏（freestanding macro） 。

了解了宏的基本概念后，我们进一步探讨宏的分类，特别是关联宏（attached macro）的特点和应用。

## 关联宏（attached macro）

必须和另一个已有的类型或者是声明关联使用，以「@」号开头

每个宏都有一个或多个角色，这些角色在宏声明的开始部分以属性的形式定义。

每个角色需要遵循不同的协议，实现对应函数，在函数内部实现宏展开的内容。

| 角色                         | 协议                 | 描述                                    |
| :--------------------------- | -------------------- | :-------------------------------------- |
| `@attached(peer)`            | PeerMacro            | 为关联的声明添加一段新的声明            |
| `@attached(accessor)`        | AccessorMacro        | 为关联的声明添加存取代码（get、set 等） |
| `@attached(memberAttribute)` | MemberAttributeMacro | 为关联的类型或扩展添加新特性            |
| `@attached(member)`          | MemberMacro          | 为关联的类型或扩展添加新的声明          |
| `@attached(conformance)`     | ExtensionMacro       | 为关联的类型或扩展添加新的协议遵循      |

举例：你可以使用`OptionSet`协议来表示bitset类型，其中每个比特代表一个集合的成员。在自定义类型中采用这个协议可以让你对这些类型执行集合相关的操作，如成员关系测试、并集和交集。

考虑下面的代码，它没有使用宏：

```swift
struct SundaeToppings: OptionSet {
    let rawValue: Int
    static let nuts = SundaeToppings(rawValue: 1 << 0)
    static let cherry = SundaeToppings(rawValue: 1 << 1)
    static let fudge = SundaeToppings(rawValue: 1 << 2)
}
```

在这段代码中， `SundaeToppings` 选项集中的每个选项都包含对初始化函数的调用，这很冗余且需要手动操作。在添加新选项时很容易出错，比如在行尾输入错误的数字。

这里有一个使用宏的代码版本：

```swift
@OptionSet<Int>
struct SundaeToppings {
    private enum Options: Int {
        case nuts
        case cherry
        case fudge
    }
}
```

本版本的 `SundaeToppings` 调用了 `@OptionSet` 宏。该宏读取私有枚举中的case列表，为每个选项生成一组常量，并添加对 `OptionSet` 协议的遵从。

以下是扩展后的 `@OptionSet` 宏的示例。

```swift
struct SundaeToppings {
    private enum Options: Int {
        case nuts
        case cherry
        case fudge
    }


    typealias RawValue = Int
    var rawValue: RawValue
    init() { self.rawValue = 0 }
    init(rawValue: RawValue) { self.rawValue = rawValue }
    static let nuts: Self = Self(rawValue: 1 << Options.nuts.rawValue)
    static let cherry: Self = Self(rawValue: 1 << Options.cherry.rawValue)
    static let fudge: Self = Self(rawValue: 1 << Options.fudge.rawValue)
}
extension SundaeToppings: OptionSet { }
```

除了关联宏，Swift 还提供了独立宏（freestanding macro），它们在用法上有所不同。

## 独立宏（freestanding macro）

独立宏在使用上无需和任何类型关联，以「#」号开头。独立宏可以声明一个新的类型，或者作为一段代码（表达式）的替换。

| 角色                         | 协议             | 描述                     |
| :--------------------------- | ---------------- | :----------------------- |
| `@freestanding(expression)`  | ExpressionMacro  | 创建一个有返回值的表达式 |
| `@freestanding(declaration)` | DeclarationMacro | 创建一个或多个声明       |

```swift
func myFunction() {
    print("Currently running \(#function)")
    #warning("Something's wrong")
}
```

在第一行代码中， `#function` 调用了 Swift 标准库中的 `function()` 宏。当您编译此代码时，Swift 会调用该宏的实现，将 `#function` 替换为当前函数的名称。当您运行此代码并调用 `myFunction()` 时，它会打印 "Currently running myFunction()"。在第二行代码中， `#warning` 调用了 Swift 标准库中的 `warning(_:)` 宏以生成自定义编译时警告。

独立的宏可以产生一个值，比如 ` #function` 就会这样做，也可以在编译时执行一个操作，比如 ` #warning` 就会这样做。



## 命名规则

在命名规则上

* 关联宏使用大写驼峰式命名法，类似于结构和类的名称。

* 独立宏的名称使用小写驼峰式命名法，类似于变量和函数的名称。

# 宏展开

当编写使用宏的 Swift 代码时，编译器会调用宏的实现来展开它们。

具体来说，Swift 以如下方式扩展宏：

* 编译器会读取代码，并创建一个内存中的语法表示。
* 编译器将内存中的部分表示发送给宏实现程序，该程序会展开宏。
* 编译器将宏调用替换为其展开形式。
* 编译器继续进行编译，使用扩展后的源代码。

```swift
let magicNumber = #fourCharacterCode("ABCD")
```

` #fourCharacterCode ` 宏接受一个长度为 4 个字符的字符串，并返回一个无符号 32 位整数，该整数对应于字符串中字符的 ASCII 值之和。某些文件格式使用这样的整数来标识数据，因为它们格式紧凑且在调试器中仍有可读性。

要扩展上面代码中的宏，编译器会读取 Swift 文件并创建一个称为抽象语法树（AST）的内存中表示形式。AST 使代码结构显式化，这使得编写与该结构交互的代码变得更加容易——比如编译器或宏实现。以下是简化了一些额外细节的上述代码的 AST 表示形式：
![865a789fe0ebd080d85cc9b0949b0dc1.png](/assets/images/865a789fe0ebd080d85cc9b0949b0dc1.png)

上图显示了该代码在内存中的结构表示方式。AST中的每个元素都对应于源代码的一部分。“Constant declaration”AST元素下面有两个子元素，代表常量声明的两个部分：其名称和值。“Macro call”元素下面有子元素，代表宏的名称和传递给宏的参数列表。

在构建AST的过程中，编译器会检查源代码是否是合法的Swift代码。例如， `#fourCharacterCode` 需要一个单个参数，该参数必须是一个字符串。如果你尝试传递一个整数参数，或者忘记了字符串字面量的结尾的引号（ `"` ），那么在编译过程中的这个阶段就会出现错误。

编译器会找到代码中你调用宏的位置，并加载实现这些宏的外部二进制文件。对于每个宏调用，编译器会将部分AST传递给该宏的实现。以下是部分AST的表示形式：
![ae7d0a17f29880e77a986ddcbaed9c40.png](/assets/images/ae7d0a17f29880e77a986ddcbaed9c40.png)

使用` #fourCharacterCode `宏时，其实现代码在展开宏时会将该部分AST作为输入。宏的实现代码仅操作其接收到的输入部分AST，这意味着无论前后的代码是什么，宏始终以相同的方式展开。这种限制有助于简化宏展开的可理解性，并有助于加快代码构建速度，因为Swift可以避免展开未发生变化的宏。

Swift通过限制实现宏的代码来帮助宏作者避免意外读取其他输入：

- 传递给宏实现的AST仅包含表示宏的AST元素，而不包含该宏前后的任何代码。
- 宏实现在沙箱环境中运行，从而防止它访问文件系统或网络。

除了这些保护措施外，宏的作者还应确保不会读取或修改除宏输入以外的任何内容。例如，宏的展开不应依赖于当前的日期和时间。

执行 `#fourCharacterCode` 会生成一个包含扩展代码的新AST。以下是该代码返回给编译器的内容：
![253cb4930d3888f478a3d44b0a1575f8.png](/assets/images/253cb4930d3888f478a3d44b0a1575f8.png)

当编译器接收到这个扩展时，它会用包含宏调用的AST元素替换包含宏扩展的AST元素。在宏扩展之后，编译器再次检查以确保程序仍然是语法上正确的Swift代码，并且所有类型都是正确的。这会产生一个可以像往常一样编译的最终AST：

![256113f29ecb86fc3ed90aa4f119c57f.png](/assets/images/256113f29ecb86fc3ed90aa4f119c57f.png)

这个AST与下面的Swift代码相对应：

```swift
let magicNumber = 1145258561 as UInt32
```

在这个例子中，输入源代码只有一条宏定义，但一个真正的程序可能包含多个相同的宏定义和对不同宏的多次调用。编译器会逐个展开宏。

如果一个宏出现在另一个宏内部，则首先展开外部宏 - 这样可以使外部宏在展开内部宏之前对其进行修改。

掌握了宏的展开机制后，我们现在来看看如何在实际中创建和使用宏。

# 创建宏

在大多数 Swift 代码中，当您实现一个符号（例如函数或类型）时，通常没有单独的声明。但是，对于宏，声明和实现是分开的。宏的声明包含其名称、所使用的参数以及可以在何处使用以及生成何种代码。宏的实现包含通过生成 Swift 代码来展开宏的代码。

完整的实现一个宏，需要包含三个部分：
* 实现宏：利用SwiftSyntax库，分析现有代码，遵循特定协议，撰写宏实现expansion函数，完成特点功能代码添加；
* 声明宏：声明宏的类型及角色，指定宏的参数，实现模块名称和模块类型等信息；
* 创建插件：创建插件列表和添加新的宏。

首先通过命令行工具创建Macro模版

```swift
swift package init --type macro
```

## 实现宏

```swift
/// Implementation of the `stringify` macro, which takes an expression
/// of any type and produces a tuple containing the value of that expression
/// and the source code that produced the value. For example
///
///     #stringify(x + y)
///
///  will expand to
///
///     (x + y, "x + y")
public enum StringifyMacro: ExpressionMacro {
  public static func expansion(
    of node: some FreestandingMacroExpansionSyntax,
    in context: some MacroExpansionContext
  ) -> ExprSyntax {
    guard let argument = node.argumentList.first?.expression else {
      fatalError("compiler bug: the macro does not have any arguments")
    }

    return "(\(argument), \(literal: argument.description))"
  }
}
```

不同类型的宏实现需要遵守对应的协议，下图展示了Swift宏协议之间的继承关系

![未命名.png](/assets/images/未命名.png)



箭头指向代表protocol协议的继承关系，子协议指向父协议，与前述章节介绍的两类宏及其附属的不同的角色宏一致。

## 声明宏

```swift
// MARK: - Stringify Macro

/// "Stringify" the provided value and produce a tuple that includes both the
/// original value as well as the source code that generated it.
@freestanding(expression)
public macro stringify<T>(_ value: T) -> (T, String) = #externalMacro(module: "MacrosImplementation", type: "StringifyMacro")
```

### 引入宏声明

你使用 `macro` 关键字引入一个宏声明。

第一行指定了宏的名称及其参数 - 名称是 `stringify` ，并且它不接受任何参数。第二行使用了 Swift 标准库中的 [`externalMacro(module:type:)`](https://developer.apple.com/documentation/swift/externalmacro(module:type:)) 宏，告诉 Swift 该宏的实现位置在哪里。在此情况下， `MacrosImplementation` 模块包含一个名为 `StringifyMacro` 的类型，该类型实现了 `#stringify` 宏。

> 宏总是以 ` public` 的形式声明。由于声明宏的代码与使用该宏的代码位于不同的模块中，因此无法在任何地方应用非公开宏。

### 宏的角色

宏声明定义了宏的角色——在源代码中可以调用该宏的位置以及该宏可以生成的代码类型。每个宏都有一个或多个角色，您可以在宏声明的开始部分作为属性来编写这些角色。一个macro可以添加多个宏的角色，这在关联宏中比较常见。

```swift
@attached(member)
@attached(extension, conformances: OptionSet)
public macro OptionSet<RawType>() =
        #externalMacro(module: "SwiftMacros", type: "OptionSetMacro")
```

这个声明中出现了两次 `@attached` 属性，一次对应每个宏角色。第一次使用（ `@attached(member)` ）表示宏向您应用的类型添加了新的成员。 `@OptionSet` 宏添加了 `init(rawValue:)` 初始化器，该初始化器是 `OptionSet` 协议所要求的，以及一些额外的成员。第二次使用（ `@attached(extension, conformances: OptionSet)` ）告诉您， `@OptionSet` 添加了对 `OptionSet` 协议的遵循。 `@OptionSet` 宏将您应用宏的类型扩展为添加对 `OptionSet` 协议的遵循。

### 宏的角色附加属性

宏的声明还提供了有关宏生成的符号名称的信息。当宏声明提供一个符号列表时，可以保证生成的只有使用这些符号的声明，这有助于您理解和调试生成的代码。独立宏没有附加属性，下面针对关联宏做介绍。

对于[关联宏](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/#attached)，peer、member和accessor宏角色需要一个`names：`参数，列出宏生成的符号的名称。如果宏在扩展内部添加声明，extension宏角色也需要一个`names：`参数。当宏声明包含`names：`参数时，宏实现必须只生成具有与该列表匹配的名称的符号。也就是说，宏不需要为每个列出的名称生成符号。这个参数的值是一个列表，包含以下一个或多个：

1. named 其中name是固定符号的名称，用于预先知道的名称。

```swift
@attached(member, names: named(_storage))
public macro DictionaryStorage() = #externalMacro(module: "MacrosImplementation", type: "DictionaryStorageMacro")
```

_storage就是要添加的属性的名称

```swift
extension DictionaryStorageMacro: MemberMacro {
  public static func expansion(
    of node: AttributeSyntax,
    providingMembersOf declaration: some DeclGroupSyntax,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    return ["\n  var _storage: [String: Any] = [:]"]
  }
}
```

2. overloaded 重载的名称与现有符号相同。

```swift
@attached(peer, names: overloaded)
public macro AddAsync() = #externalMacro(module: "MacrosImplementation", type: "AddAsyncMacro")
```

​	AddAsyncMacro会对现有的函数，添加一个与现有函数同名同参数的异步函数，所以是被认为重载overloaded

3. prefixed 其中prefix在符号名称前，用于以固定字符串开头的名称。

4. suffixed 其中后缀附加在符号名称之后，用于以固定字符串结尾的名称。

5. arbitrary 在宏扩展之前无法确定的任意名称。举例：

```swift
@attached(member, names: named(RawValue), named(rawValue),
        named(`init`), arbitrary)
@attached(extension, conformances: OptionSet)
public macro OptionSet<RawType>() =
        #externalMacro(module: "SwiftMacros", type: "OptionSetMacro")
```

在上述声明中， `@attached(member)` 宏在 `names:` 标签之后为每个由 `@OptionSet` 宏生成的符号添加了参数。宏为名为 `RawValue` 、 `rawValue` 和 `init` 的符号添加了声明，因为这些名称在宏声明之前就已经知道了，因此宏声明直接列出了这些符号。

宏声明还包括在名称列表之后的 `arbitrary` ，这样宏就可以生成在使用宏时才确定名称的声明。例如，当 `@OptionSet` 宏应用于上面的 `SundaeToppings` 时，它生成与枚举情况相对应的类型属性， `nuts` 、 `cherry` 和 `fudge` 。

## 创建插件

```swift
@main
struct SwiftMacrosLibraryPlugin: CompilerPlugin {
    let providingMacros: [Macro.Type] = [
 				...
        StringifyMacro.self,
				...
    ]
}
```

一个Package中可以包含多个Swift Macro，这些Macro都需要添加到`providingMacros`中.

从“实现宏”到最后的“创建插件”，这个流程顺序是编写代码的流程，实际代码执行时，执行顺序是倒叙的，即从创建插件开始查找、执行；

创建宏需要深入理解SwiftSyntax库，它为宏的实现提供了基础架构。

# SwiftSyntax

[SwiftSyntax](https://github.com/apple/swift-syntax)库提供了用于检查、处理和操作Swift源代码的高级、安全且高效的API数据结构和算法。SwiftSyntax库是Swift解析器、swift-format和Swift宏等工具的基础。

Swift Macro 与 SwiftSyntax 之间存在密切的关系，因为 SwiftSyntax 提供了一组库，它们可以在源码精确的树状表示（即 SwiftSyntax 树）上工作。这个树状结构构成了 Swift 宏系统的基础，宏扩展节点被表示为 SwiftSyntax 节点，而宏生成的也是一个 SwiftSyntax 树，以便插入到源文件中。

SwiftSyntax 允许开发者使用抽象语法树（AST）以结构化的方式与 Swift 代码进行交互。宏的实现使用 SwiftSyntax 模块来处理 Swift 代码，无论是独立宏（Freestanding Macros）还是附加宏（Attached Macros），它们的实现都需要符合 SwiftSyntax 中相应的协议。

具体来说，Swift Macro 的实现依赖于 SwiftSyntax 提供的接口和协议，例如 `ExpressionMacro`、`MemberMacro` 等，这些协议定义了宏如何扩展抽象语法树（AST）。宏的实现会接收到使用宏的代码的 AST，并在宏的库内部调用相应的方法（如 `expansion(of:in:)`），来找到宏的参数并计算结果，最终返回一个新的 AST 节点。

![b2012fc5f61dc2db2469d1b518021a47.png](/assets/images/b2012fc5f61dc2db2469d1b518021a47.png)
## Syntax Nodes 语法节点

语法树由称为语法节点的元素组成。为了帮助对语法节点进行分类和组织，SwiftSyntax为语法节点定义了一个层次结构的协议。

在这个层次结构的顶部是 `SyntaxProtocol` 协议。每一种协议都有对应的结构体类型遵循该协议。

```swift
public struct Syntax: SyntaxProtocol, SyntaxHashable {
  let data: SyntaxData

  /// Needed for the conformance to ``SyntaxProtocol``.
  ///
  /// Needed for the conformance to ``SyntaxProtocol``. Just returns `self`.
  public var _syntaxNode: Syntax {
    return self
  }

  init(_ data: SyntaxData) {
    self.data = data
  }
}
```

Syntax节点表示一棵节点树，其中`_syntaxNode`属性遵守SyntaxProtocol，返回自己；通过`data`这个属性，所有语法节点都可以访问其底层的`SyntaxData`，其中包含关于节点结构、位置和原始语法的基本信息。

`SyntaxProtocol` 进一步细化，包含以下子协议：

- [`DeclSyntaxProtocol`](https://swiftpackageindex.com/swiftlang/swift-syntax/600.0.1/documentation/swiftsyntax/declsyntaxprotocol) 对于像 `struct` 、 `class` 、 `enum` 和 `protocol`  这样的声明。
  - `struct DeclSyntax: DeclSyntaxProtocol`

- [`StmtSyntaxProtocol`](https://swiftpackageindex.com/swiftlang/swift-syntax/600.0.1/documentation/swiftsyntax/stmtsyntaxprotocol) 对于像 `if` 、 `switch` 和 `do` 这样的语句。
  - `struct StmtSyntax: StmtSyntaxProtocol`

- [`ExprSyntaxProtocol`](https://swiftpackageindex.com/swiftlang/swift-syntax/600.0.1/documentation/swiftsyntax/exprsyntaxprotocol) 对于像函数调用、字面量和闭包这样的表达式
  - `struct ExprSyntax: ExprSyntaxProtocol `

- [`TypeSyntaxProtocol`](https://swiftpackageindex.com/swiftlang/swift-syntax/600.0.1/documentation/swiftsyntax/typesyntaxprotocol) 对于像 `Array<String>` 、 `[String: String]` 和 `some Protocol`这样的类型
  - `struct TypeSyntax: TypeSyntaxProtocol`

- [`PatternSyntaxProtocol`](https://swiftpackageindex.com/swiftlang/swift-syntax/600.0.1/documentation/swiftsyntax/patternsyntaxprotocol) 对于像 `case (_, let x)` 这样的模式
  - `struct PatternSyntax: PatternSyntaxProtocol`


语法节点构成了语法树的“分支”，因为它们通常是一组或多个语法节点的高级集合。这些分支共同构成了源代码的语法结构。这种结构被编译器和静态分析器用于以高级方式处理源代码。

有一种特殊的语法节点是 `SyntaxCollection` (`protocol SyntaxCollection: SyntaxProtocol`)，它代表具有可变数量子节点的语法。例如，代码块值可以在一对大括号之间包含零个或多个语句。为了表示这些子节点， `CodeBlockSyntax` 值有一个 `statements` 访问器，它返回一个 `CodeBlockItemListSyntax` 值。这个语法集合中的元素是 `CodeBlockItemSyntax` 值。

用一个简单的图来解释这个概念：

```swift
+----------------+        +----------------+        +------------------------+
| CodeBlockSyntax |------>| statements     |------>| CodeBlockItemListSyntax |
+----------------+        +----------------+        +------------------------+
|                |        |                |        |                				 |
| statements     |        | elements       |        | elements       				 |
+----------------+        +----------------+        +------------------------+
                                                 |                |
                                                 |                |
                                                 +----------------+
                                                 |                |
                                                 |                |
                                                 +----------------+
                                                 |                |
                                                 |                |
                                                 +----------------+
                                                 |                |
                                                 |                |
                                                 +----------------+
|                     |  |                     |  |                     |
| CodeBlockItemSyntax |  | CodeBlockItemSyntax |  | CodeBlockItemSyntax |
+---------------------+  +---------------------+  +---------------------+
```

在这个图中：

1. `CodeBlockSyntax`：代表整个代码块，它是一个语法节点，可以包含零个或多个语句。
2. `statements`：是`CodeBlockSyntax`的一个访问器，用来获取代码块中的所有语句。
3. `CodeBlockItemListSyntax`：是`statements`返回的值，代表代码块中所有语句的集合。
4. `elements`：是`CodeBlockItemListSyntax`中的元素，每个元素都是一个`CodeBlockItemSyntax`。
5. `CodeBlockItemSyntax`：代表单个的代码块项，可能是一个语句、一个声明等。

这个结构允许SwiftSyntax库以一种灵活的方式来处理代码块中的语句，因为语句的数量可以是零个或多个，而`SyntaxCollection`正是为了处理这种可变数量的子节点而设计的。

## 常见的DeclSyntax类型

SwiftSyntax中定义了很多符合`DeclSyntaxProtocol`的结构体，代表了Swift语法中的各种声明类型。每个结构体对应于一种特定的声明，并包含表示该声明不同部分的属性。下面是对每个结构体的简要解释：
1. AccessorDeclSyntax：表示属性或下标中的访问器声明（例如get、set）。

2. ActorDeclSyntax：表示一个actor声明，它是一个引用类型，保护它的可变状态不受数据竞争的影响。

3. AssociatedTypeDeclSyntax：表示协议中的关联类型声明。

4. ClassDeclSyntax：表示一个类声明。

5. DeinitializerDeclSyntax：表示类的反初始化声明。

6. EditorPlaceholderDeclSyntax：表示编辑器上下文中声明的占位符。

7. EnumCaseDeclSyntax：表示枚举中的枚举case声明。

8. EnumDeclSyntax：表示枚举声明。

9. ExtensionDeclSyntax：表示类型的扩展声明。

10. FunctionDeclSyntax：表示函数声明。

11. IfConfigDeclSyntax：表示声明的条件编译块（#if, #else, #endif）。

12. ImportDeclSyntax：表示一个导入声明。

13. InitializerDeclSyntax：表示初始化器声明。

14. MacroDeclSyntax：表示一个宏声明。

15. MacroExpansionDeclSyntax：表示声明上下文中的宏扩展。

16. MissingDeclSyntax：表示语法树中缺失的声明。

17. OperatorDeclSyntax：表示操作符声明。

18. PoundSourceLocationSyntax：表示#sourceLocation指令。

19. PrecedenceGroupDeclSyntax：表示操作符的优先级组声明。

20. ProtocolDeclSyntax：表示协议声明。

21. StructDeclSyntax：表示结构体声明。

22. SubscriptDeclSyntax：表示下标声明。

23. TypeAliasDeclSyntax：表示typealias声明。

24. VariableDeclSyntax：表示变量声明（let或var）。

   这些结构体都提供了对应声明类型的结构化表示，便于以抽象语法树的形式操作和分析Swift源代码。

## Syntax Tokens 语法标记

语法树的叶子节点包含 `TokenSyntax` 类型的值。一个token语法值代表语法中的一个基本单元以及与其相关的一些细微差别，比如标识符以及其周围的空白。所有token的组合代表了源代码的词法结构。这种结构被linter和格式化器用来分析源代码的文本内容。

```swift
struct TokenSyntax: SyntaxProtocol, SyntaxHashable
```

TokenSyntax 遵循协议SyntaxProtocol和SyntaxHashable，SyntaxHashable遵循Hashable协议，

表示单个标记的语法节点。语法树的所有源代码都由标记布局节点表示，不要单独包含任何源代码。

## Syntax Trivia 语法知识

琐碎的内容包括空白、注释、编译器指示和错误的源代码。琐碎的内容对文档的语法结构贡献最大，但通常与它的语义结构关系不大。因此，编译器等工具在处理源代码时通常会忽略琐碎的内容。然而，对于编辑器、IDE、格式化器和重构引擎等工具来说，维护琐碎的内容是很重要的。SwiftSyntax使用 `Trivia` 类型明确表示琐碎的内容。与标记语法相关的琐碎内容可以使用 `leadingTrivia` 和 `trailingTrivia` 访问器进行检查。

例如下面代码，创建AttributeSyntax节点，其中内容一个开头从新的一行开始且空格为2的，标识是“DictionaryStorageProperty”

```swift
      AttributeSyntax(
        leadingTrivia: [.newlines(1), .spaces(2)],
        attributeName: IdentifierTypeSyntax(
          name: .identifier("DictionaryStorageProperty")
        )
      )
```

## Navigating the Syntax Tree解析语法树

SwiftSyntax库提供了多种遍历语法树结构的方法。每个语法节点都通过 `parent` 访问器包含对其父节点的引用，并通过 `children(viewMode:)` 方法包含对其子节点的引用。语法树的子节点总是按顺序排列，而相关的 `viewMode` 允许客户端表达处理缺失和意外语法的意图。

语法节点还包含访问这些子节点的强类型API。例如， `ClassDeclSyntax` 提供了一个 `identifier` 来获取类的名称，以及一个 `ClassDeclSyntax/members` 来获取表示包含成员的花括号块的语法节点。

大多数语法分析器希望同时处理多种语法，并按照从根节点到叶子节点的常规顺序遍历这些语法。SwiftSyntax提供了一套标准的类和协议，用于使用访客风格的API进行语法遍历操作。

一个 [`SyntaxVisitor`](https://swiftpackageindex.com/swiftlang/swift-syntax/600.0.1/documentation/swiftsyntax/syntaxvisitor) 类可以用来从根节点遍历到叶子节点的源代码。为了检查特定类型的语法节点，客户端需要重写相应的` visit`方法，以接受该类型的语法。访问方法作为语法树中顺序遍历的一部分被调用。为了提供后序遍历，可以重写相应的` visitPost`方法。

能够重写特定语法节点的访客可以实现为 `SyntaxRewriter` 的一个子类。与 `SyntaxVisitor` 类似，客户端需要重写接受感兴趣语法节点的相应 `visit` 方法，并返回一个重写的语法节点作为结果。

# 属性包装器（Property Wrapper）和宏（Macro）

属性包装器（Property Wrapper）和宏（Macro）是 Swift 中两个不同的功能，它们在代码生成和代码结构方面有各自的用途和特点。以下是它们之间的主要区别：

### 属性包装器（Property Wrapper）

- 属性包装器是一种属性修饰符，它在声明属性时使用，并且可以在属性声明周围“插入”代码。
- 它通过定义一个包含 `wrappedValue` 属性的结构体或类，并使用 `@propertyWrapper` 属性来实现。
- 属性包装器在编译时不会改变代码的结构，而是在运行时生成额外的代码来实现包装器的逻辑。
- 属性包装器通常用于单个属性，为属性添加额外的功能，如存储属性的封装、验证、计算等。

### 宏（Macro）

- 宏在编译时将源代码中的宏调用替换为宏定义的代码。
- 它通过在编译时期替换文本来实现，可以生成任意的代码结构。
- 宏可以作用于整个声明或表达式，不限于属性，可以生成任意的代码片段。
- 宏的自定义性更高，可以生成复杂的代码结构，但也需要更多的控制和理解。

# 参考

* https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/
* https://docs.swift.org/swift-book/documentation/the-swift-programming-language/macros/
* https://developer.apple.com/documentation/swift/applying-macros/
* https://mp.weixin.qq.com/s/8_cLO9Ym77npx8n-4v0Dow
* https://xiaozhuanlan.com/topic/1403528796
* https://swift-ast-explorer.com/
* https://github.com/swiftlang/swift-evolution/blob/main/proposals/0402-extension-macros.md
* https://engineering.traderepublic.com/get-ready-for-swift-macros-fe21d3867e02
* https://swiftpackageindex.com/swiftlang/swift-syntax/600.0.1/documentation/swiftsyntax
* 
