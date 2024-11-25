---
title: "Swift Macro系列3-实践篇attached(accessor) 等"
description: "本文是《Swift Macro系列》的第三篇"
category: macro
tags: Swift Macro,attached macro,freestanding macro,SwiftSyntax
---

本文是《Swift Macro系列》的第三篇，旨在通过详细的步骤和代码示例，指导读者如何设计和实现`@attached(accessor)`，`attached(memberAttribute)`,`@attached(conformance)`和`@attached(member)`宏。

## @attached(accessor)	

accessor 意为存取器，当关联到属性后，可以操作其 `get`、`set`、`willSet` 和 `didSet` 方法。

假设我们有一个 Point 类型，其中的 x、y参数都是希望可以直接操作字典进行赋值和获取的，每增加一个参数都需要重复编写 get 和 set 实现，也没法通过 property wrapper 访问其他的存储属性来简化代码。

```swift
struct Point {
  var x: Int = 1
  var y: Int = 2
}
```

这种场景下 `@attached(accessor)` 宏就能发挥作用，通过添加宏，预期的展开结果如下图所示：
<img src="/assets/images/截屏2024-11-22%2009.54.41.png" alt="截屏2024-11-22 09.54.41.png" style="zoom:50%;" />

源码AST结构图

![截屏2024-11-22 10.54.33.png](/assets/images/截屏2024-11-22%2010.54.33.png)

1. **SourceFile**: 这是AST的根节点，代表整个源文件。

2. **CodeBlockItemList**: 这个节点包含了源文件中的所有代码块项。

3. **CodeBlockItem**: 这是`CodeBlockItemList`中的一个元素，代表一个代码块项。

4. **StructDecl**: 这个节点表示一个结构体声明。在这个例子中，它声明了一个名为`Point`的结构体。

5. **AttributeList** 和 **DeclModifierList**: 这两个节点分别表示属性列表和声明修饰符列表。在这个例子中，它们都是空的，因为`Point`结构体没有特殊的属性或修饰符。

6. **struct Point**: 这是结构体的名称。

7. **MemberBlock**: 这个节点包含了结构体成员的代码块。

8. **MemberBlockItemList**: 这个节点包含了结构体成员的列表。

9. **MemberBlockItem**: 这是`MemberBlockItemList`中的一个元素，代表一个成员。

10. **VariableDecl**: 这个节点表示一个变量声明。

11. **AttributeList** 和 **DeclModifierList**: 同样，这两个节点表示属性列表和声明修饰符列表，在这个例子中也是空的。

12. **var**: 这是变量声明的关键字。

13. **PatternBindingList** 和 **PatternBinding**: 这些节点表示模式绑定列表和模式绑定。在这个例子中，它们表示变量的名称和类型。

14. **IdentifierPattern**: 这个节点表示变量的名称，例如`x`和`y`。

15. **TypeAnnotation**: 这个节点表示变量的类型注解。

16. **IdentifierType**: 这个节点表示类型标识符，例如`Int`。

17. **InitializerClause**: 这个节点表示变量的初始化子句。

18. **IntegerLiteralExpr**: 这个节点表示整数字面量表达式，例如`1`和`2`，它们是变量`x`和`y`的初始值。

整个结构体`Point`包含两个属性`x`和`y`，它们都是`Int`类型，并且分别被初始化为`1`和`2`。这个AST展示了Swift代码的结构化表示，是后续撰写宏实现的基础。

创建expansion函数

```swift
public struct DictionaryStoragePropertyMacro: AccessorMacro {
  public static func expansion<
    Context: MacroExpansionContext,
    Declaration: DeclSyntaxProtocol
  >(
    of node: AttributeSyntax,
    providingAccessorsOf declaration: Declaration,
    in context: Context
  ) throws -> [AccessorDeclSyntax] {
  	//添加code
  }
```

由于添加`DictionaryStoragePropertyMacro`宏的目标是对变量添加计算属性的get/set方法，所以declaration找到VariableDeclSyntax，VariableDeclSyntax 表示变量声明，对应于示例中的`var x: Int = 1`。

```swift
    guard let varDecl = declaration.as(VariableDeclSyntax.self),
      let binding = varDecl.bindings.first,
      let identifier = binding.pattern.as(IdentifierPatternSyntax.self)?.identifier,
      binding.accessorBlock == nil,
      let type = binding.typeAnnotation?.type
    else {
      return []
    }
```

varDecl.bindings类型是PatternBindingListSyntax，通过查找第一个元素，获得标识符（identifier，例如y）和类型（type，例如Int）

```swift
    // Ignore the "_storage" variable.
    if identifier.text == "_storage" {
      return []
    }

    guard let defaultValue = binding.initializer?.value else {
      throw CustomError.message("stored property must have an initializer")
    }
```

忽略内置添加的存储属性`_storage`，否则就会为`_storage`也添加计算属性；并且`_storage`必须有初始值。

```swift
    return [
      """
      get {
        _storage[\(literal: identifier.text), default: \(defaultValue)] as! \(type)
      }
      """,
      """
      set {
        _storage[\(literal: identifier.text)] = newValue
      }
      """
    ]
```

返回需要添加的计算属性

虽然上面代码优化了字典参数的写法，但也引入了另外一个问题：所有属性前面必须加上 `@DictionaryStorageProperty` 模板代码，这又造成了重复编码，尤其是对含有大量属性的类型声明。要解决这个问题，需要引入下面这个新角色 `@attached(memberAttribute)`。

## @attached(memberAttribute)

这个角色可以为类型或扩展添加新特性。要解决上面模板代码的问题。

设计流程是这样的，先使用`@DictionaryStorage`宏修饰`Point`整体，然后展开后添加下面三项内容，对每个属性加`@DictionaryStorageProperty`宏，增加`_storage`属性。

![截屏2024-11-22 11.27.16.png](/assets/images/截屏2024-11-22%2011.27.16.png)

接着，`@DictionaryStorageProperty`宏在展开时添加计算属性，以实现完整功能。

![截屏2024-11-22 11.29.43.png](/assets/images/截屏2024-11-22%2011.29.43.png)

看下一下代码具体实现，我们新建一个宏：

```swift
extension DictionaryStorageMacro: MemberAttributeMacro {
  public static func expansion(
    of node: AttributeSyntax,
    attachedTo declaration: some DeclGroupSyntax,
    providingAttributesFor member: some DeclSyntaxProtocol,
    in context: some MacroExpansionContext
  ) throws -> [AttributeSyntax] {
    guard let property = member.as(VariableDeclSyntax.self),
      property.isStoredProperty
    else {
      return []
    }

    return [
      AttributeSyntax(
        leadingTrivia: [.newlines(1), .spaces(2)],
        attributeName: IdentifierTypeSyntax(
          name: .identifier("DictionaryStorageProperty")
        )
      )
    ]
  }
}
```

判断成员是变量，且是存储属性；添加新的声明标识AttributeSyntax，即宏@DictionaryStorageProperty，并配置格式：在新的一行，2个空格。这里注意的是此expansion函数，有member参数，**参数类型**: some DeclSyntaxProtocol，在宏的扩展过程中，member 参数用于表示当前正在处理的成员声明。宏可以根据这个成员声明来生成或修改代码。

## @attached(member)

member 角色可以为类型、扩展添加初始化方法、参数等新的声明，为类和结构体添加存储属性，甚至为枚举添加新的 case。

下面代码预期实现@CaseDetection宏，为枚举类型的每个case添加检测判断。

![截屏2024-11-22 11.57.56.png](/assets/images/截屏2024-11-22%2011.57.56.png)

实现

```swift
public enum CaseDetectionMacro: MemberMacro {
  public static func expansion(
    of node: AttributeSyntax,
    providingMembersOf declaration: some DeclGroupSyntax,
    in context: some MacroExpansionContext
  ) throws -> [DeclSyntax] {
    declaration.memberBlock.members
      .compactMap { $0.decl.as(EnumCaseDeclSyntax.self) }
      .map { $0.elements.first!.name }
      .map { ($0, $0.initialUppercased) }
      .map { original, uppercased in
        """
        var is\(raw: uppercased): Bool {
          if case .\(raw: original) = self {
            return true
          }

          return false
        }
        """
      }
  }
}
```

具体步骤如下：

1. **`compactMap { $0.decl.as(EnumCaseDeclSyntax.self) }`**:过滤并转换为 `EnumCaseDeclSyntax` 类型，忽略不符合条件的声明。
  
2. **`.map { $0.elements.first!.name }`**:提取每个枚举情况的名称。
  
3. **`.map { ($0, $0.initialUppercased) }`**:将枚举名称转换为元组，包含原始名称和首字母大写的名称。
  
4. **`.map { original, uppercased in ... }`**:生成一个字符串模板，定义一个布尔属性，用于检测枚举实例是否为特定枚举情况。
  
   - 模板内容：
     ```swift
     var is\(raw: uppercased): Bool {
       if case .\(raw: original) = self {
         return true
       }
       return false
     }
     ```
   - 这个模板会生成类似 `var isFoo: Bool { ... }` 的属性，用于检测枚举实例是否为 `.foo` 情况。

代码片段通过遍历枚举情况，生成一组布尔属性，用于检测枚举实例是否为特定枚举情况。

## @attached(conformance)

为关联的类型或扩展添加新的协议遵循；接下来，看一个具体例子，预期对类型实现`OptionSet`协议；

`OptionSet` 协议在 Swift 中用于表示一组可以组合的选项。它类似于位掩码（bitmask），允许你通过组合不同的选项来创建复合选项。

```swift
struct ShippingOptions: OptionSet {
    let rawValue: Int

    static let nextDay    = ShippingOptions(rawValue: 1 << 0)
    static let secondDay  = ShippingOptions(rawValue: 1 << 1)
    static let priority   = ShippingOptions(rawValue: 1 << 2)
    static let standard   = ShippingOptions(rawValue: 1 << 3)
}

// 使用示例
var options: ShippingOptions = [.nextDay, .priority]

if options.contains(.nextDay) {
    print("Next day shipping is selected.")
}

options.insert(.standard)
print("Current options: \(options)")
```

设计@MyOptionSet宏，展开效果


![截屏2024-11-22 15.01.41.png](/assets/images/截屏2024-11-22%2015.01.41.png)

```swift
extension OptionSetMacro: ExtensionMacro {
  public static func expansion(
    of node: AttributeSyntax,
    attachedTo declaration: some DeclGroupSyntax,
    providingExtensionsOf type: some TypeSyntaxProtocol,
    conformingTo protocols: [TypeSyntax],
    in context: some MacroExpansionContext
  ) throws -> [ExtensionDeclSyntax] {
    // Decode the expansion arguments.
    guard let (structDecl, _, _) = decodeExpansion(of: node, attachedTo: declaration, in: context) else {
      return []
    }

    // If there is an explicit conformance to OptionSet already, don't add one.
    if let inheritedTypes = structDecl.inheritanceClause?.inheritedTypes,
      inheritedTypes.contains(where: { inherited in inherited.type.trimmedDescription == "OptionSet" })
    {
      return []
    }

    return [try ExtensionDeclSyntax("extension \(type): OptionSet {}")]
  }
}
```

- 代码实现了一个宏扩展，用于为符合条件的结构体添加 OptionSet 协议的扩展
- 调用 decodeExpansion 方法解码宏扩展参数，确保宏正确附加到结构体声明。
- 检查结构体是否已经显式遵循 OptionSet 协议。如果已经遵循，返回空数组，不添加新的扩展。
- 如果结构体未遵循 OptionSet 协议，生成一个新的扩展声明，使其遵循 OptionSet 协议。返回包含该扩展声明的数组。

### 角色组合

在上面的 `MyOptionSet` 宏的例子中，需要使用多个关联宏角色的组合共同对代码进行优化，才能实现完整功能，上面代码为了简化说明，只是列出了实现ExtensionMacro宏的部分

```swift
@attached(member, names: arbitrary)
@attached(extension, conformances: OptionSet)
public macro MyOptionSet<RawType>() = #externalMacro(module: "MacrosImplementation", type: "OptionSetMacro")
```

关于角色组合的使用需要满足下面几个原则：

1. 一个宏可以有多个关联角色（attached roles）
2. Swift 会在宏使用的地方将所有适用的角色都还原成对应实现
3. 在使用的地方，至少需要有一个角色满足场景



## To be continue

