---
title: "Swift Macro系列2-实践篇@attached(peer)"
description: "本文是《Swift Macro系列》的第二篇，旨在通过详细的步骤和代码示例，指导读者如何设计和实现一个@attached(peer)宏。我们将从宏的基本功能分析开始，逐步深入到expansion函数的实现，并探讨如何通过SwiftSyntax库来操作抽象语法树（AST）。"
category: macro
tags: Swift Macro,attached macro,freestanding macro,SwiftSyntax
---

本文是《Swift Macro系列》的第二篇，旨在通过详细的步骤和代码示例，指导读者如何设计和实现一个@attached(peer)宏。我们将从宏的基本功能分析开始，逐步深入到expansion函数的实现，并探讨如何通过SwiftSyntax库来操作抽象语法树（AST）。

# 实现宏的流程

1.分析设计宏要实现的功能是什么？

*  确定是要为现有结构增加功能（此时应使用关联宏），还是实现一个完全独立的新功能（此时应使用独立宏）。

2.遵守对应宏协议，实现expansion函数

* 分析源代码结构的函数declaration是类型/扩展，还是函数？ 是否符合当前宏的使用条件，即进行类型判断，安全防护，抛出错误处理。
* 通过分析现有类型的AST结构，添加需要实现的功能。一般包括：函数签名和返回类型提取；函数名称提取；
* 从新的声明中移除宏声明，即`@xxxx`或`#xxxxx`这类在源代码开头添加的宏，避免宏被重复展开。
* 设置新声明参数，生成新声明。


# @attached(peer)

Peer Macros是一种宏，它能够在**相同作用域内为现有声明生成新的声明**，特别适用于增强代码组织和可维护性。

这个角色除了能关联到常见的类型、参数、方法以外，甚至能关联 import 和操作符的声明，并为其添加新的声明。以下面的方法为例，

定义了一个 Swift 宏 `AddAsyncMacro`，它是一个用于扩展非异步函数以支持 `async` 属性的宏。在这个例子中，`AddAsyncMacro` 会在原始的非异步函数上生成一个新的 `async` 函数版本。

添加后的展开效果如图所示：

![截屏2024-11-21 11.19.37.png](/assets/images/截屏2024-11-21%2011.19.37.png)

由于我们是在相同作用域在添加新的函数，所以适合使用关联宏peer类型，

接下来实现PeerMacro协议，即实现expansion函数，不同的协议类型，需要分别实现不同参数的expansion函数。

```swift
public protocol PeerMacro: AttachedMacro {
  static func expansion(
    of node: AttributeSyntax,
    providingPeersOf declaration: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax]
}
```
解释其中参数和返回值含义：
* `of node: AttributeSyntax`： 此参数代表宏的属性语法节点，也就是代码中应用的宏标签。在 AST（抽象语法树）中，它对应于代表该宏Attribute的位置。
* `providingPeersOf declaration: some DeclSyntaxProtocol`：此参数代表应用宏的声明语法节点。在 AST 中，它通常对应于一个具体的声明节点，如函数声明 (FunctionDecl) 或变量声明 (VarDecl)。这个参数提供了宏操作的目标，你可以在其中创建新的“同类”声明。
* `in context: some MacroExpansionContext`：此参数提供宏扩展执行的上下文信息，包含可用的符号、诊断信息等。虽然它不直接对应于 AST 中的一个节点，但它为宏的扩展过程提供环境支持。
* 返回值是`DeclSyntax`类型数组，`DeclSyntax` 就是专门用来表示声明（Declarations）的节点类型。声明可以是变量、函数、类、结构体、枚举等 Swift 代码中的元素。因此，返回值代表你要添加的内容结构。

在分析时，在实现expansion函数之前，我们通常会使用工具如[Swift AST Explorer](https://swift-ast-explorer.com/)来分析源代码结构，这有助于我们更好地理解AST并实现宏。

首先利用[Swift AST Explorer](https://swift-ast-explorer.com/)，分析源代码结构

```swift
@AddAsync
func d(a: Int, for b: String, _ value: Double, completionBlock: @escaping (Bool) -> Void) -> Void {
  completionBlock(true)
}
```

函数AST结构如下图所示，添加了关键的标注信息，对应SwiftSynatx中对应的类型或结构字段名

![截屏2024-11-21 11.57.50.png](/assets/images/截屏2024-11-21%2011.57.50.png)

首先简要分析此AST结构

1. **CodeBlockItemList**：这是一个包含多个代码块项目的列表。
2. **CodeBlockItem**：这是CodeBlockItemList中的一个项目。
3. **FunctionDecl**：这是一个函数声明。
4. **AttributeList**：这是一个宏的属性列表，包含了修饰函数的宏的信息。
     **Attribute**：这是一个属性。
	 **@**：这是一个属性标记，表示该参数是逃逸的。
	 **IdentifierType**：这是一个标识符类型。
5. **DeclModifierList**：这是一个声明修饰符列表，包含了函数的修饰符。
**func**：这是函数关键字。
 **d**：这是函数的名称。
6. **FunctionSignature**：这是函数签名。
7. **FunctionParameterClause**：这是函数参数的括号部分。
8. **FunctionParameterList**：这是函数参数的列表。
9. **FunctionParameter**：这是单个函数参数。

结合实现expansion函数的流程，将一步步展示如何实现这个功能的PeerMacro

1. 判断声明类型应该是函数，否则抛出错误

```swift
 // Only on functions at the moment.
    guard var funcDecl = declaration.as(FunctionDeclSyntax.self) else {
      throw CustomError.message("@addAsync only works on functions")
    }
```

expansion函数的declaration参数类型是DeclSyntaxProtocol，是通用类型，而函数类型FunctionDeclSyntax是遵循DeclSyntaxProtocol的结构体，所示需要使用as进行类型转换

2. 特定限制检查，该宏应用于满足特定条件的函数，包括：不是异步函数；将completion处理程序作为最后一个参数并返回Void。

   此处需要重点分析AST结构中，signature下面的parameterClause和returnClause参数，

   * parameterClause参数是FunctionParameterClauseSyntax类型，也是遵循SyntaxProtocol的结构体，该结构体通常用于解析和生成 Swift 代码中的函数参数子句，例如函数的参数列表部分。
   * parameterClause.parameters是FunctionParameterListSyntax类型，是一个集合类型，查看AST图片结构中可以看到多个FunctionParameter，其中每一个参数就是一个FunctionParameterSyntax结构体，用于表示 Swift 语法中的函数参数。例如当前示例中d函数参数`a: Int`,参数` b: String`。
   * returnClause参数是ReturnClauseSyntax类型，`ReturnClauseSyntax` 是一个结构体，遵循 `SyntaxProtocol` 和 `SyntaxHashable` 协议，它用于表示 Swift 语法中的返回子句。例如当前示例中d函数返回 `-> Void` 可以被解析为一个 `ReturnClauseSyntax` 实例，其中 `Void` 是返回类型。

```swift
// This only makes sense for non async functions.
    if funcDecl.signature.effectSpecifiers?.asyncSpecifier != nil {
      throw CustomError.message(
        "@addAsync requires an non async function"
      )
    }

    // This only makes sense void functions
    if funcDecl.signature.returnClause?.type.as(IdentifierTypeSyntax.self)?.name.text != "Void" {
      throw CustomError.message(
        "@addAsync requires an function that returns void"
      )
    }

    // Requires a completion handler block as last parameter
    guard let completionHandlerParameterAttribute = funcDecl.signature.parameterClause.parameters.last?.type.as(AttributedTypeSyntax.self),
      let completionHandlerParameter = completionHandlerParameterAttribute.baseType.as(FunctionTypeSyntax.self)
    else {
      throw CustomError.message(
        "@addAsync requires an function that has a completion handler as last parameter"
      )
    }

    // Completion handler needs to return Void
    if completionHandlerParameter.returnClause.type.as(IdentifierTypeSyntax.self)?.name.text != "Void" {
      throw CustomError.message(
        "@addAsync requires an function that has a completion handler that returns Void"
      )
    }
```

参数completionHandlerParameterAttribute是signature.parameterClause集合中最后一个元素（即d函数的completionBlock参数）的类型的type就是，如下图所示局部展开的AST中的AttriubutedType字段。

<img src="/assets/images/截屏2024-11-21%2015.27.53.png" alt="截屏2024-11-21 15.27.53.png" style="zoom:50%;" />

在分析了参数和AST结构之后，我们分享一些有用的AST查看技巧，这将帮助我们更有效地理解和操作AST。
* AST展示的节点名称通常在末尾去掉`Syntax`，例如`AttributedType`实际上代表的是`AttributedTypeSyntax`。
* 点击AST结构展开或收起，鼠标点中其中节点会在左侧展示对应节点的子属性，这些属性不直接展示在AST结构中。

completionHandlerParameterAttribute的baseType如下图所示，是FunctionType。对应示例中d函数`completionBlock: @escaping (Bool) -> Void`中闭包参数，即`(Bool) -> Void`。

![截屏2024-11-21 15.34.20.png](/assets/images/截屏2024-11-21%2015.34.20.png)


3. 在分析源代码声明基础上，添加新功能

通过前面分析，completionHandlerParameterAttribute和completionHandlerParameter两个参数，分别拿到completionBlock闭包和其闭包参数，然后返回类型和结果处理：确定返回类型是否为Result类型，如果是则提取Result类型。

```swift
    let returnType = completionHandlerParameter.parameters.first?.type
//parameters是TupleTypeElementListSyntaxSyntaxCollection，returnType是Void
    let isResultReturn = returnType?.children(viewMode: .all).first?.description == "Result"
    let successReturnType = isResultReturn ? returnType!.as(IdentifierTypeSyntax.self)!.genericArgumentClause?.arguments.first!.argument : returnType
//
```

这里特别区分不同

```swift
func c(a: Int, for b: String, _ value: Double, completionBlock: @escaping (Result<String, Error>) -> Void) -> Void {
      completionBlock(.success("a: \(a), b: \(b), value: \(value)"))
    }
```

转成异步函数,宏会根据闭包的返回类型（Void 或 Result）生成不同的异步函数体。

```swift
func c(a: Int, for b: String, _ value: Double) async throws -> String {
  try await withCheckedThrowingContinuation { continuation in
    c(a: a, for: b, value) { returnValue in

      switch returnValue {
      case .success(let value):
        continuation.resume(returning: value)
      case .failure(let error):
        continuation.resume(throwing: error)
      }
    }
  }
}
```

如果闭包返回类型是 Result，使用 try await withCheckedThrowingContinuation 处理可能的错误。修改返回类型为闭包的返回类型（如果是 Result，则提取 Result 的第一个泛型参数）。

如果闭包返回类型是 Void，使用 await withCheckedContinuation 处理无错误的情况。直接调用 continuation.resume(returning: ()) 处理 Void 返回类型。

接下来，移除源代码声明中的completionHandler参数，因为转换后不再需要

```swift
    // Remove completionHandler and comma from the previous parameter
    var newParameterList = funcDecl.signature.parameterClause.parameters
    newParameterList.removeLast()
    var newParameterListLastParameter = newParameterList.last!
    newParameterList.removeLast()
    newParameterListLastParameter.trailingTrivia = []
    newParameterListLastParameter.trailingComma = nil
    newParameterList.append(newParameterListLastParameter)
```

4. 从新的声明中移除宏声明，避免宏被重复展开

```swift
    let newAttributeList = funcDecl.attributes.filter {
      guard case let .attribute(attribute) = $0,
        let attributeType = attribute.attributeName.as(IdentifierTypeSyntax.self),
        let nodeType = node.attributeName.as(IdentifierTypeSyntax.self)
      else {
        return true
      }

      return attributeType.name.text != nodeType.name.text
    }
```

这部分代码完全几乎可以不做修改，在很多同类型Peer宏展开中使用，与具体的功能无关。

5.设置新声明参数，返回新的声明内容。

```swift
    let callArguments: [String] = newParameterList.map { param in
      let argName = param.secondName ?? param.firstName

      let paramName = param.firstName
      if paramName.text != "_" {
        return "\(paramName.text): \(argName.text)"
      }

      return "\(argName.text)"
    }

    let switchBody: ExprSyntax =
      """
            switch returnValue {
            case .success(let value):
              continuation.resume(returning: value)
            case .failure(let error):
              continuation.resume(throwing: error)
            }
      """

    let newBody: ExprSyntax =
      """

        \(raw: isResultReturn ? "try await withCheckedThrowingContinuation { continuation in" : "await withCheckedContinuation { continuation in")
          \(raw: funcDecl.name)(\(raw: callArguments.joined(separator: ", "))) { \(raw: returnType != nil ? "returnValue in" : "")

      \(raw: isResultReturn ? switchBody : "continuation.resume(returning: \(raw: returnType != nil ? "returnValue" : "()"))")
          }
        }

      """


```
在这段代码中，我们首先创建了一个`callArguments`数组，它包含了新异步函数调用时所需的所有参数。然后，我们根据闭包的返回类型（是否为`Result`类型）来决定如何生成异步函数体。如果闭包返回`Result`类型，我们将使用`try await withCheckedThrowingContinuation`来处理可能的错误，并根据`Result`的值来决定是继续执行还是抛出错误。如果闭包返回`Void`类型，我们将使用`await withCheckedContinuation`来处理无错误的情况。

接下来，我们更新函数声明，添加异步关键字（`async`），并根据需要添加`throws`关键字：
```swift
    // add async
    funcDecl.signature.effectSpecifiers = FunctionEffectSpecifiersSyntax(
      leadingTrivia: .space,
      asyncSpecifier: .keyword(.async),
      throwsSpecifier: isResultReturn ? .keyword(.throws) : nil
    )

    // add result type
    if let successReturnType {
      funcDecl.signature.returnClause = ReturnClauseSyntax(leadingTrivia: .space, type: successReturnType.with(\.leadingTrivia, .space))
    } else {
      funcDecl.signature.returnClause = nil
    }

    // drop completion handler
    funcDecl.signature.parameterClause.parameters = newParameterList
    funcDecl.signature.parameterClause.trailingTrivia = []

    funcDecl.body = CodeBlockSyntax(
      leftBrace: .leftBraceToken(leadingTrivia: .space),
      statements: CodeBlockItemListSyntax(
        [CodeBlockItemSyntax(item: .expr(newBody))]
      ),
      rightBrace: .rightBraceToken(leadingTrivia: .newline)
    )

    funcDecl.attributes = newAttributeList

    funcDecl.leadingTrivia = .newlines(2)

    return [DeclSyntax(funcDecl)]
```

最后，我们更新函数的参数列表，移除原有的completion handler参数，并设置新的函数体和属性：

通过这些步骤，我们成功地将一个非异步函数转换为一个异步函数，并确保了代码的正确性和可读性。

构建由几个层级组层：
* ExprSyntax：表示一个表达式，例如变量赋值、函数调用、算术运算等。
* CodeBlockItemSyntax：表示代码块中的一个条目，可以是语句（StmtSyntax）、声明（DeclSyntax）或表达式（ExprSyntax）。ExprSyntax可以作为 CodeBlockItemSyntax 的一部分，表示代码块中的具体操作。
* CodeBlockSyntax：表示一个代码块，通常包含一系列CodeBlockItemSyntax组成。
* DeclSyntax：表示一个声明，例如函数声明、变量声明、类型声明等。包含 CodeBlockSyntax 作为其主体部分。

通过这种方式，你可以构建复杂的声明、代码块和表达式结构。

## 小结

在本文中，我们深入探讨了Swift宏的实现流程，特别关注了@attached(peer)宏的详细实践。我们从分析宏的功能需求出发，逐步实现了expansion函数，这是宏实现的核心。通过分析源代码结构，我们学会了如何为现有函数添加新的异步版本，这是通过PeerMacro协议实现的。我们详细讨论了如何检查函数的签名、参数和返回类型，以及如何根据这些信息生成新的声明。此外，我们还学习了如何从源代码中移除宏声明，以避免宏的重复展开，并且设置了新的声明参数，以生成和返回新的声明内容。

通过本文的指导，读者应该能够理解如何在Swift中实现和使用宏，特别是关联宏，以及如何利用SwiftSyntax库来操作和分析抽象语法树（AST）。这些知识不仅有助于提高代码的组织性和可维护性，还能在编译时生成更高效、更简洁的代码。随着Swift语言的不断发展，宏的使用将为开发者提供更多的灵活性和能力，以构建更加强大的Swift应用程序。希望本文能作为您在Swift宏世界中探索和实践的起点，激发您在项目中有效地利用宏的潜力。
