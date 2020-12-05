
@Published 适合于ObserabledObject配合使用，在存储属性上添加@Published，将会使得有值变化时，自动发出publisher

# @Published的wrappedValue
```swift
@available(iOS 13.0, OSX 10.15, tvOS 13.0, watchOS 6.0, *)
@propertyWrapper public struct Published<Value> {

    /// Initialize the storage of the Published property as well as the corresponding `Publisher`.
    public init(wrappedValue: Value)

    public init(initialValue: Value)

    /// A publisher for properties marked with the `@Published` attribute.
    public struct Publisher : Publisher {

        /// The kind of values published by this publisher.
        public typealias Output = Value

        /// The kind of errors this publisher might publish.
        ///
        /// Use `Never` if this `Publisher` does not publish errors.
        public typealias Failure = Never

        /// This function is called to attach the specified `Subscriber` to this `Publisher` by `subscribe(_:)`
        ///
        /// - SeeAlso: `subscribe(_:)`
        /// - Parameters:
        ///     - subscriber: The subscriber to attach to this `Publisher`.
        ///                   once attached it can begin to receive values.
        public func receive<S>(subscriber: S) where Value == S.Input, S : Subscriber, S.Failure == Published<Value>.Publisher.Failure
    }

    /// The property that can be accessed with the `$` syntax and allows access to the `Publisher`
    public var projectedValue: Published<Value>.Publisher { mutating get }
}
```
你是否跟我一样有这样的困惑，为什么这个property wrapper中没有看到wrappedValue，要实现一个property wrapper这不是必须的吗？

再来回顾一下State，同样的property wrapper，实现了wrappedValue
```swift
@available(iOS 13.0, OSX 10.15, tvOS 13.0, watchOS 6.0, *)
@frozen @propertyWrapper public struct State<Value> : DynamicProperty {

    /// Initialize with the provided initial value.
    public init(wrappedValue value: Value)

    /// Initialize with the provided initial value.
    public init(initialValue value: Value)

    /// The current state value.
    public var wrappedValue: Value { get nonmutating set }

    /// Produces the binding referencing this state value
    public var projectedValue: Binding<Value> { get }
}
```

如果你把上面的Published再Xcode中自己实现，是会提示错误没有写wrappedValue实现的，所以说明系统的实现内部写了，但是为什么看不到？

唯一的可能是，wrappedValue的访问级别为internal，也就是说其实@Published的实际实现声明了类似于interval var wrappedValue:Value的内容，这满足了property wrapper的要求。
但是设计者不想暴露出来给外部使用，所以标记为internal，使其wrappedValue在框架之外是不可见的。

# @Published的使用

既然wrappedValue不想暴露出来被外部使用，意图就是让使用projectedValue，它的类型是Published<Value>.Publisher，

Publisher是一个在@Published属性包装器内部的嵌套结构体类型，同时也遵守Publisher协议

这样的设计使得标记为@Published的属性投影（$）都可以assign或sink订阅处理，使用各种Operator，如下面代码示例

```
class Person: ObservableObject{
    @Published var email = ""
}

let remoteVerify = $email
                    .debounce(for: .milliseconds(500),scheduler: DispatchQueue.main)
                    .removeDuplicates()
                    .eraseToAnyPublisher()
                    

```

# 创造Publisher的常见方式

1. 定义一个struct
2. 添加需要从外部传入的参数作为存储属性
3. 添加一个计算属性publisher，遵循AnyPublisher<Output, Failure>，具体Output和Failure类型根据业务需要设计
4. 使用初始化结构体，调用publisher，返回遵循Publisher协议对象，视情况后续Operator操作


```swift
struct EmailCheck {
    let email: String 
    var publisher: AnyPublisher<Bool, Never> {
            Future<Bool, Never> { promise in
                    DispatchQueue.global().asyncAfter(deadline: .now() + 0.5) {
                        if self.email.lowercased() !" "xxx@gmail.com" {
                            promise(.success(false))
                        } else {
                            promise(.success(true))
                    }
            }
    }
    .receive(on: DispatchQueue.main)
    .eraseToAnyPublisher()
  }
}
```

Publishers.CombineLatest  这种大写struct直接使用的方式，有时更加高效

