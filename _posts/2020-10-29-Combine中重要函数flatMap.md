---
title: "Combine中重要函数flatMap"
description: "flatMap是Combine中很重要的一个操作符，本文将介绍flatMap的作用，Result类型中的flatMap如何使用，Publisher中flatMap有什么使用的注意问题。"
category: programming
tags: iOS,Combine,flatmap
---

flatMap
=======

* flatMap操作符可以用来把多个上游下发的publishers扁平化成一个单独的publisher，向下游传递

* flatMap返回的publisher经常与上游下发的publisher的类型不一致

* 使用flatMap的常见使用场景是，上游传递下来的publisher中的元素作为参数，传递给一个新的publisher，把新的publisher向下传递

* 系统的Result类型也有对应的flatMap方法，目标类似，把第一个Result中元素取出，变换后，放入一个新的Result中，下发

* flatMap函数（）中传入的函数或者闭包，都是返回一个Publisher类型的

Result类型中的flatMap
-----------------

传入闭包参数是

```swift
/// A value that represents either a success or a failure, including an
/// associated value in each case.
@frozen public enum Result<Success, Failure> where Failure : Error {

    /// A success, storing a `Success` value.
    case success(Success)

    /// A failure, storing a `Failure` value.
    case failure(Failure)

    /// Returns a new result, mapping any success value using the given
    /// transformation.
    ///
    /// Use this method when you need to transform the value of a `Result`
    /// instance when it represents a success. The following example transforms
    /// the integer success value of a result into a string:
    ///
    ///     func getNextInteger() -> Result<Int, Error> { /* ... */ }
    ///
    ///     let integerResult = getNextInteger()
    ///     // integerResult == .success(5)
    ///     let stringResult = integerResult.map({ String($0) })
    ///     // stringResult == .success("5")
    ///
    /// - Parameter transform: A closure that takes the success value of this
    ///   instance.
    /// - Returns: A `Result` instance with the result of evaluating `transform`
    ///   as the new success value if this instance represents a success.
    public func map<NewSuccess>(_ transform: (Success) -> NewSuccess) -> Result<NewSuccess, Failure>

    /// Returns a new result, mapping any failure value using the given
    /// transformation.
    ///
    /// Use this method when you need to transform the value of a `Result`
    /// instance when it represents a failure. The following example transforms
    /// the error value of a result by wrapping it in a custom `Error` type:
    ///
    ///     struct DatedError: Error {
    ///         var error: Error
    ///         var date: Date
    ///
    ///         init(_ error: Error) {
    ///             self.error = error
    ///             self.date = Date()
    ///         }
    ///     }
    ///
    ///     let result: Result<Int, Error> = // ...
    ///     // result == .failure(<error value>)
    ///     let resultWithDatedError = result.mapError({ e in DatedError(e) })
    ///     // result == .failure(DatedError(error: <error value>, date: <date>))
    ///
    /// - Parameter transform: A closure that takes the failure value of the
    ///   instance.
    /// - Returns: A `Result` instance with the result of evaluating `transform`
    ///   as the new failure value if this instance represents a failure.
    public func mapError<NewFailure>(_ transform: (Failure) -> NewFailure) -> Result<Success, NewFailure> where NewFailure : Error

    /// Returns a new result, mapping any success value using the given
    /// transformation and unwrapping the produced result.
    ///
    /// - Parameter transform: A closure that takes the success value of the
    ///   instance.
    /// - Returns: A `Result` instance with the result of evaluating `transform`
    ///   as the new failure value if this instance represents a failure.
    public func flatMap<NewSuccess>(_ transform: (Success) -> Result<NewSuccess, Failure>) -> Result<NewSuccess, Failure>

    /// Returns a new result, mapping any failure value using the given
    /// transformation and unwrapping the produced result.
    ///
    /// - Parameter transform: A closure that takes the failure value of the
    ///   instance.
    /// - Returns: A `Result` instance, either from the closure or the previous 
    ///   `.success`.
    public func flatMapError<NewFailure>(_ transform: (Failure) -> Result<Success, NewFailure>) -> Result<Success, NewFailure> where NewFailure : Error

    /// Returns the success value as a throwing expression.
    ///
    /// Use this method to retrieve the value of this result if it represents a
    /// success, or to catch the value if it represents a failure.
    ///
    ///     let integerResult: Result<Int, Error> = .success(5)
    ///     do {
    ///         let value = try integerResult.get()
    ///         print("The value is \(value).")
    ///     } catch error {
    ///         print("Error retrieving the value: \(error)")
    ///     }
    ///     // Prints "The value is 5."
    ///
    /// - Returns: The success value, if the instance represents a success.
    /// - Throws: The failure value, if the instance represents a failure.
    public func get() throws -> Success

    /// Creates a new result by evaluating a throwing closure, capturing the
    /// returned value as a success, or any thrown error as a failure.
    ///
    /// - Parameter body: A throwing closure to evaluate.
    public init(catching body: () throws -> Success)
}
```



Publisher中flatMap
-----------------

使用flatMap时的错误类型配置
=================

```swift
URLSession.shared.dataTaskPublisher(for: someURL)
  .flatMap({ output -> AnyPublisher<Data, Error> in
  })
```

上面代码会提示错误，

```swift
Instance method flatMap(maxPublishers:_:) requires the types URLSession.DataTaskPublisher.Failure (aka URLError) and Error be equivalent
```



dataTaskPublisher的Output是 ((data: Data, response: URLResponse)) ， Failure是URLError，由于与flatMap返回Failure的Error类型不一致，所以错误

改为

```swift
URLSession.shared.dataTaskPublisher(for: someURL)
  .mapError({ $0 as Error })
  .flatMap({ output -> AnyPublisher<Data, Error> in

  })
```



Combine中所有Failure错误类型都是遵循Error，所以可以自由转换

问题 "flatMap(maxPublishers:_:) is only available in iOS 14.0 or newer"
=====================================================================

```swift
let strings = ["https://donnywals.com", "https://practicalcombine.com"]
strings.publisher
  .map({ url in URL(string: url)! })
  .flatMap({ url in
    return URLSession.shared.dataTaskPublisher(for: url)
  })
```

If you're using Xcode 12, this code will result in the flatMap(maxPublishers:_:) is only available in iOS 14.0 or newer

 compiler error.

原因是：上游错误Failure类型是Never，下游dataTaskPublisher的Failure类型是URLError，所以两者不匹配；iOS14后Combine将会自动将上游Never转成URLError，这种转换由于很明确所以没有歧义，但是在iOS13，Combine不能做这种推断，需要开发者明确的告诉Combine需要转成URLError，所以需要改动如下：

```swift
//iOS13
let strings = ["https://donnywals.com", "https://practicalcombine.com"]
strings.publisher
  .map({ url in URL(string: url)! })
  .setFailureType(to: URLError.self) // this is required for iOS 13
  .flatMap({ url in
    return URLSession.shared.dataTaskPublisher(for: url)
  })
```

这就解释了为什么编译器提示错误flatMap(maxPublishers:_:)只能使用在iOS14及后续版本

使用FlatMap将上游Failure从Never变为Error（iOS14+）
----------------------------------------

如果不修复这个错误，cmd+click flatMap 函数，跳转到定义处，**从Never转换成P.Failure**

```swift
@available(macOS 11.0, iOS 14.0, tvOS 14.0, watchOS 7.0, *)
extension Publisher where Self.Failure == Never {

  public func flatMap<P>(maxPublishers: Subscribers.Demand = .unlimited, _ transform: @escaping (Self.Output) -> P) -> Publishers.FlatMap<P, Publishers.SetFailureType<Self, P.Failure>> where P : Publisher
}
```



**这个extension的条件是extension Publisher where Self.Failure == Never，也就是必须Failure是Never**

使用FlatMap将上游Failure从Error变为Never（iOS14+）
----------------------------------------

另一个iOS14新增flatMap函数也很相似，但是与上面这个相反，**当publisher可失败，可以使用flatMap创建一个Publisher的Failure是Never的**

```swift
@available(macOS 11.0, iOS 14.0, tvOS 14.0, watchOS 7.0, *)
extension Publisher {

    /// Transforms all elements from an upstream publisher into a new publisher up to a maximum number of publishers you specify.
    ///
    /// - Parameters:
    ///   - maxPublishers: Specifies the maximum number of concurrent publisher subscriptions, or ``Combine/Subscribers/Demand/unlimited`` if unspecified.
    ///   - transform: A closure that takes an element as a parameter and returns a publisher that produces elements of that type.
    /// - Returns: A publisher that transforms elements from an upstream  publisher into a publisher of that element’s type.
    public func flatMap<P>(maxPublishers: Subscribers.Demand = .unlimited, _ transform: @escaping (Self.Output) -> P) -> Publishers.FlatMap<Publishers.SetFailureType<P, Self.Failure>, Self> where P : Publisher, P.Failure == Never
}
```

举例

```swift
URLSession.shared.dataTaskPublisher(for: someURL)
  .flatMap({ _ in
    return Just(10)
  })
```



这段代码没有意义，只是用来举例，Output是（Int，Response），Failure是URLError，转成Just，Failure是Never

如果上面代码想在iOS13也有效，需要使用setFailureType(to:)

```swift
URLSession.shared.dataTaskPublisher(for: someURL)
  .flatMap({ _ -> AnyPublisher<Int, URLError> in
    return Just(10)
      .setFailureType(to: URLError.self) // this is required for iOS 13
      .eraseToAnyPublisher()
  })
```



Never到Never
-----------

还有一种flatMap，针对的也是上游的Failure是Never，转换成的Publisher是（Output，Never）

```swift
@available(macOS 11.0, iOS 14.0, tvOS 14.0, watchOS 7.0, *)
extension Publisher where Self.Failure == Never {
    public func flatMap<P>(maxPublishers: Subscribers.Demand = .unlimited, _ transform: @escaping (Self.Output) -> P) -> Publishers.FlatMap<P, Self> where P : Publisher, P.Failure == Never
}
```



原有的iOS13支持的FlatMap
------------------

```swift
extension Publisher {
    public func flatMap<T, P>(maxPublishers: Subscribers.Demand = .unlimited, _ transform: @escaping (Self.Output) -> P) -> Publishers.FlatMap<P, Self> where T == P.Output, P : Publisher, Self.Failure == P.Failure
}
```



参考
==

-   https://www.donnywals.com/configuring-error-types-when-using-flatmap-in-combine/