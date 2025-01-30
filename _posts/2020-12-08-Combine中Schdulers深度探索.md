---
title: "Combine中Schdulers深度探索"
description: "探讨如何利用Combine中Schedulers对任务执行进行管理。将会介绍以下几方面内容：什么是Schedulers？Combine的默认调度机制;切换Schedulers的操作符;通过实例深入理解Schedulers的调度;Combine中Schedulers的类型和使用."
category: programming
tags: iOS,Combine,Schedulers
---

本文主要探讨如何利用Combine中Schedulers对任务执行进行管理。假定读者已经了解Combine的基本原理，想要进一步对Combine中任务调度进行详细了解。由于Schedulers与GCD（Grand Dispatch Queue），线程（thread），RunLoop都有关联，所以也需要读者有这方面的基础了解才能更好的读懂下文。

本文将会介绍以下几方面内容：

* 什么是Schedulers？
* Combine的默认调度机制
* 切换Schedulers的操作符
* 通过实例深入理解Schedulers的调度
* Combine中Schedulers的类型和使用

# 什么是Schedulers

Scheduler是一个协议（protocol），定义了什么时候（when）和在什么地方（where）执行一个闭包，其中什么地方（where）意味着 runloop，dispatch queue，operator queue，三选一使用哪个；什么时候（when）意味着Combine事件流的虚拟时间，也就是Combine中Publisher，Operator，Subscriber中具体实现的上下文是执行在哪个Scheduler上。

你可能注意到了Scheduler的定义刻意避免了对线程的任何引用，这是因为你的代码具体执行在哪个线程，是由你选择的Scheduler决定的，也就是说Scheduler协议的具体实现（DispatchQueue，OperationQueue都是遵循Scheduler协议的）定义了任务调度，同时你需要注意Scheduler和线程并不是完全的对应关系，指定一个Scheduler后也可能在不同的线程上执行任务。

# Combine的默认调度机制

默认情况下，即不使用任何后续我们将要讲到的scheduler，Combine将会默认使用上游Publisher产生的线程发送到下游
**示例1**
```swift
  var cancellables = Set<AnyCancellable>()
  
  let intSubject = PassthroughSubject<Int, Never>()

  intSubject.sink(receiveValue: { value in
     print(value)
     print(Thread.current)
  }).store(in: &cancellables)

  intSubject.send(1)

  DispatchQueue.global().async {
    intSubject.send(2)
  }

  let queue = OperationQueue()
  queue.maxConcurrentOperationCount = 1
  queue.underlyingQueue = DispatchQueue(label: "com.donnywals.queue")

  queue.addOperation {
    intSubject.send(3)
  }
```

输出

```
1
<NSThread: 0x600003b80780>{number = 1, name = main}
2
<NSThread: 0x600003b99300>{number = 6, name = (null)}
3
<NSThread: 0x600003bb2080>{number = 4, name = (null)}
```

sink的receiveValue被三次调用，每次被调用在不同线程，`.send(1)`是在主线程发出的，此时sink输出也是在主线程，而`send(2)`和`send(3)`都是在子线程发出，则对应sink收到也是在子线程

# 切换Scheduler的操作符

我们经常需要在后台运行一些消耗资源的操作，这样当用户在界面上操作时，能够避免阻塞主线线程，不影响户操作。Combine中提供了Schedulers对任务执行进行切换，主要通过两个操作符：`subscribe(on:)`和`receive(on:)`

## receive(on:)
receive(on:)影响它使用位置下游的scheduler
**示例2**
```swift
publisher
.operatorA
.receive(on:)
.operatorB
.sink
```
receive(on:)插在两个OperatorA，B之间，会影响B，影响sink，不会影响A，所以是影响使用位置下游的scheduler

## subscribe(on:)
subscribe(on:)稍微复杂一些，如果你了解Comebine的Backpressure机制，会记得遵循Publisher协议需要实现`receive`方法，遵循Subscription协议需要实现`request`，`cancel`方法，`subscribe(on:)`就是用来控制上面提到的这些方法的切换调度；一般来说，`subscribe(on:)`会影响一个Combine异步事件链条上的Publisher和Operator，但是不是一定，它取决于实现Publisher协议对象的具体内部实现，如果其内部实现指定了线程，将不会遵循外部`subscribe(on:)`的设置，反之，如果没有指定一般会使用。

上面的描述可能还是让你一头雾水，下面将会使用具体的代码实例解释。

# 通过实例深入理解Schedulers的调度

我们将Publisher分成两类进行分析，一类是自定义实现的Publisher，另一类是系统提供的Publisher。

## 自定义Publisher

**完整示例3**

```swift
//先创建一个ComputationSubscription（Subscription）和ExpensiveComputation（Publisher）
final class ComputationSubscription<Output>: Subscription {
  private let duration: TimeInterval
  private let sendCompletion: () -> Void
  private let sendValue: (Output) -> Subscribers.Demand
  private let finalValue: Output
  private var cancelled = false

  init(duration: TimeInterval, sendCompletion: @escaping () -> Void, sendValue: @escaping (Output) -> Subscribers.Demand, finalValue: Output) {
    self.duration = duration
    self.finalValue = finalValue
    self.sendCompletion = sendCompletion
    self.sendValue = sendValue
  }

  func request(_ demand: Subscribers.Demand) {
    if !cancelled {
      print("Beginning expensive computation on thread \(Thread.current.number)")
    }
    Thread.sleep(until: Date(timeIntervalSinceNow: duration))
    if !cancelled {
      print("Completed expensive computation on thread \(Thread.current.number)")
      _ = self.sendValue(self.finalValue)
      self.sendCompletion()
    }
  }

  func cancel() {
    cancelled = true
  }
}

extension Publishers {

  public struct ExpensiveComputation: Publisher {
    public typealias Output = String
    public typealias Failure = Never

    public let duration: TimeInterval

    public init(duration: TimeInterval) {
      self.duration = duration
    }

    public func receive<S>(subscriber: S) where S : Subscriber, Failure == S.Failure, Output == S.Input {
      Swift.print("ExpensiveComputation subscriber received on thread \(Thread.current.number)")
      let subscription = ComputationSubscription(duration: duration,
                                                 sendCompletion: { subscriber.receive(completion: .finished) },
                                                 sendValue: { subscriber.receive($0) },
                                                 finalValue: "Computation complete")

      subscriber.receive(subscription: subscription)
    }
  }
}
//--------------------------------------
//继续使用上面的代码，完成一个完整的订阅流程

// 1
let computationPublisher = Publishers.ExpensiveComputation(duration: 3)

// 2
let queue = DispatchQueue(label: "serial queue")


let currentThread = Thread.current.number
print("Start computation publisher on thread \(currentThread)")

let subscription = computationPublisher
  .subscribe(on: queue)  // 3
    .map({ (val) -> String in
//        print(val)
        //4
        print("Thread.current.number=\(Thread.current.number)")
        return val
    })
  .receive(on: DispatchQueue.main)   //5
  .sink { value in
    let thread = Thread.current.number
    print("Received computation result on thread \(thread): '\(value)'")
  }
```

输出

```swift
Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 6
Beginning expensive computation on thread 6
Completed expensive computation on thread 6
Thread.current.number=6
Received computation result on thread 1: 'Computation complete'
```

1. 定义了一个耗时的ExpensiveComputation，在指定时间后发出String
2. 定义了一个串行队列
3. `subscribe(on:)`的使用影响Publisher的线程使用，Operator也是一种Publisher，map中操作也是在子线程，`.subscribe(on:)`的位置在map前或者后面，没有区别
4. 使用Thread.current.number输出当前所在线程，在playground中，1是主线程
5. `receive(on:)`用于控制其位置下游的scheduler，一般使用在sink前，用于Subscirber侧的线程控制，即sink中输出值的运行线程，但不仅仅限于此，如果`receive(on:)`位置移动到map的前面，也会影响map的线程

下面修改一下代码，调整`receive(on:)`位置，移动到`subscribe(on:)`后面，map前面

**示例3-局部修改**

```swift
// 1
let computationPublisher = Publishers.ExpensiveComputation(duration: 0)

// 2
let queue = DispatchQueue(label: "serial queue")

// 3
let currentThread = Thread.current.number
print("Start computation publisher on thread \(currentThread)")

let subscription = computationPublisher
    .subscribe(on: queue)
    .receive(on: DispatchQueue.main)
    .map({ (val) -> String in
//        print(val)
        print("Thread.current.number=\(Thread.current.number)")
        return val
    })
  .sink { value in
    let thread = Thread.current.number
    print("Received computation result on thread \(thread): '\(value)'")
  }
```

输出

```swift
Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 6
Beginning expensive computation on thread 6
Completed expensive computation on thread 6
Thread.current.number=1
Received computation result on thread 1: 'Computation complete'
```

注意：map切到主线程执行，publisher仍然在子线程执行，`receive(on:)`成功影响其下游的scheduler，map位于其下游，所以切到主线程，而Publisher被`subscribe(on:)`控制还位于子线程

**示例3-局部修改2**

```swift
  public struct ExpensiveComputation: Publisher {
    public typealias Output = String
    public typealias Failure = Never

    public let duration: TimeInterval

    public init(duration: TimeInterval) {
      self.duration = duration
    }

    public func receive<S>(subscriber: S) where S : Subscriber, Failure == S.Failure, Output == S.Input {
        DispatchQueue.main.async {    //note
            Swift.print("ExpensiveComputation subscriber received on thread \(Thread.current.number)")
            let subscription = ComputationSubscription(duration: duration,
                                                       sendCompletion: { subscriber.receive(completion: .finished) },
                                                       sendValue: { subscriber.receive($0) },
                                                       finalValue: "Computation complete")

            subscriber.receive(subscription: subscription)
        }
    }
  }
```

这段代码唯一一处修改是Publisher的实现ExpensiveComputation中，`receive`方法的内部实现，指定了运行在主线程。

此时再执行下面代码，注意`subscribe`和`receive`位置恢复到初始状态

```swift
let subscription = computationPublisher
  .subscribe(on: queue)  // 3
    .map({ (val) -> String in
//        print(val)
        //4
        print("Thread.current.number=\(Thread.current.number)")
        return val
    })
  .receive(on: DispatchQueue.main)   //5
  .sink { value in
    let thread = Thread.current.number
    print("Received computation result on thread \(thread): '\(value)'")
  }
```

输出

```swift
Start computation publisher on thread 1
ExpensiveComputation subscriber received on thread 1
Beginning expensive computation on thread 6
Completed expensive computation on thread 6
Thread.current.number=1
Received computation result on thread 1: 'Computation complete'
```

此时，map不再执行在subscribe指定的子线程，而是主线程，这是Publisher内部实现决定的；

## 系统Publisher

下面继续介绍系统提供的Publisher，主要介绍PassthroughSubject和CurrentValueSubject；这里我们会频繁使用`Thread.current`这个系统提供的方法，打印线程信息，比如
```
<NSThread: 0x600001a6c900>{number = 1, name = main}
```
就代表当前在线程1，线程的名字是main，也就是主线程，在playground中主线程number等于1，

### PassthroughSubject

**示例4**

```swift
let intSubject = PassthroughSubject<Int, Never>()

intSubject
    .subscribe(on: DispatchQueue.global())
    .sink(receiveValue: { value in
        print(Thread.current)
    })
    .store(in: &cancellables)
        
intSubject.send(1)
intSubject.send(2)
intSubject.send(3)
```

输出

```SWIFT
<NSThread: 0x600001a6c900>{number = 1, name = main}
<NSThread: 0x600001a6c900>{number = 1, name = main}
<NSThread: 0x600001a6c900>{number = 1, name = main}
```

subscribe切到子线程，但是sink并没有在子线程输出，而是仍然在主线程，开头"Combine的默认调度机制"小节已经证实是按subject.send所在线程，不受`subscribe(on:)`影响

**示例5**

```swift
let intSubject = PassthroughSubject<Int, Never>()

intSubject
    .map({ (num) -> Int in
        print("map:\(Thread.current)")
        return num + 1
    })
    .subscribe(on: DispatchQueue.global())
    .sink(receiveValue: { value in
        print("sink:\(Thread.current)")
    })
    .store(in: &cancellables)
//  sleep(5)

    intSubject.send(1)
    intSubject.send(2)
    intSubject.send(3)
```

如果运行在模拟器，没有输出，这是因为，由于subscribe切换执行到子线程异步执行，代码马上继续执行，当subscription建立前，send已经发出，所以没有任何输出；

如果把`sleep(5)`的注释打开，由于延迟了足够的时间才执行send，这时sink已经执行完成，所以会有输出;

运行在Playground，输出

```swift
map:<NSThread: 0x600001f54440>{number = 1, name = main}
sink:<NSThread: 0x600001f54440>{number = 1, name = main}
map:<NSThread: 0x600001f54440>{number = 1, name = main}
sink:<NSThread: 0x600001f54440>{number = 1, name = main}
map:<NSThread: 0x600001f54440>{number = 1, name = main}
sink:<NSThread: 0x600001f54440>{number = 1, name = main}
```

发现PassthroughSubject不受`subscribe(on:)`影响，取决于send所在线程

## CurrentValueSubject

**示例6**

```swift
let intSubject = CurrentValueSubject<Int, Never>(0)

intSubject
   .subscribe(on: DispatchQueue.global())
   .map({ (num) -> Int in
         print("num=\(num),map:\(Thread.current)")
         return num + 1
   })
   .receive(on: DispatchQueue.global())
   .sink(receiveValue: { value in
        print("num=\(value),sink:\(Thread.current)")
   })
   .store(in: &cancellables)
        
sleep(5)
intSubject.send(1)
intSubject.send(2)
```

输出

```swift
num=0,map:<NSThread: 0x600001e48180>{number = 3, name = (null)}
num=1,sink:<NSThread: 0x600001e48180>{number = 3, name = (null)}
num=1,map:<NSThread: 0x600001e20900>{number = 1, name = main}
num=2,map:<NSThread: 0x600001e20900>{number = 1, name = main}   
num=2,sink:<NSThread: 0x600001e2d340>{number = 6, name = (null)}
num=3,sink:<NSThread: 0x600001e48180>{number = 3, name = (null)}
```

重组一下，按map和sink配对输出，便于对照

```swift
num=0,map:<NSThread: 0x600001e48180>{number = 3, name = (null)}
num=1,sink:<NSThread: 0x600001e48180>{number = 3, name = (null)}

num=1,map:<NSThread: 0x600001e20900>{number = 1, name = main}
num=2,sink:<NSThread: 0x600001e2d340>{number = 6, name = (null)}

num=2,map:<NSThread: 0x600001e20900>{number = 1, name = main}   
num=3,sink:<NSThread: 0x600001e48180>{number = 3, name = (null)}
```

发现CurrentValueSubject，初始值0按`subscribe(on:)`执行调度，后续map所在线程send按发出线程，sink按`receive(on:)`指定的sechduler

# Combine中Schedulers的类型和使用

苹果提供了Scheduler协议的几个具体实现

* ImmediateScheduler：简单的Scheduler，在当前线程执行代码，在没有指定subscribe(on:), receive(on:)的时候，是默认实现；如果你执行延迟任务（如调用`schedule(after:)`），使用这个Scheduler将会遇到致命错误。
* RunLoop：关联Fundation的Thread对象
* DispatchQueue：可以是串行或并行，main或global，通常使用串行global执行后台任务，主线程main执行UI相关任务
* OperationQueue，一个控制执行的队列，与GCD相似，使用OperationQueue.main为UI相关任务服务，使用其他队列执行后台任务

下面针对每一种类型介绍如下：

## ImmediateScheduler

```swift
var subscription = Set<AnyCancellable>()

let source = Timer
  .publish(every: 1.0, on: .main, in: .common)
  .autoconnect()
  .scan(0) { counter, _ in counter + 1 }
  
source
    //1
    .map{ value -> Int in
        print("1:\(Thread.current)")
        return value
    }
    // 2
    .receive(on: ImmediateScheduler.shared)
    // 3
    .map{ value -> Int in
        print("2:\(Thread.current)")
        return value
    }
    .eraseToAnyPublisher()
    .sink { _ in
        //print(out)
    }
    .store(in: &subscription)
```
部分输出摘录
```
1:<NSThread: 0x600000c2c2c0>{number = 1, name = main}
2:<NSThread: 0x600000c2c2c0>{number = 1, name = main}
1:<NSThread: 0x600000c2c2c0>{number = 1, name = main}
2:<NSThread: 0x600000c2c2c0>{number = 1, name = main}
```
这个例子中，为了明确知道Scheduler调度后，具体在哪个线程执行，在注释1和3处，加入了两个map，他们的闭包内部没有任何实际意义，只是用来打印线程，直接返回上游的结果；

由于ImmediateScheduler在当前线程执行，playground的main thread 是1，所以都在主线程执行

再上面代码上做一点修改代码,在注释1前面增加，`receive(on: DispatchQueue.global())`

```swift
source
    //4
    .receive(on: DispatchQueue.global())
    //1
    .map{ value -> Int in
        print("1:\(Thread.current)")
        return value
    }
    // 2
    .receive(on: ImmediateScheduler.shared)
    // 3
    .map{ value -> Int in
        print("2:\(Thread.current)")
        return value
    }
    .eraseToAnyPublisher()
    .sink { _ in
        //print(out)
    }
    .store(in: &subscription)
```
输出
```
1:<NSThread: 0x6000018393c0>{number = 6, name = (null)}
2:<NSThread: 0x6000018393c0>{number = 6, name = (null)}
1:<NSThread: 0x600001829300>{number = 3, name = (null)}
2:<NSThread: 0x600001829300>{number = 3, name = (null)}
```
此时，Publisher不在运行在主线程，而是在global queue


## RunLoop

RunLoop产生早于DispatchQueue，是在线程级别管理输入资源的方式，我们的应用程序的主线程默认有一个关联的Runloop,你也可以通过调用RunLoop.current获得Fundation框架提供的当前线程；现在Runloop使用场景更少，DispatchQueue更常用，但是特定场景下Runloop还是很有用，比如Timer执行在RunLoop上；

为了方便对比，我们继续利用ImmediateScheduler小节提供的代码示例，再其基础上做修改

```swift
var subscription = Set<AnyCancellable>()

let source = Timer
  .publish(every: 1.0, on: .main, in: .common)
  .autoconnect()
  .scan(0) { counter, _ in counter + 1 }
  
source
    //4
    .receive(on: DispatchQueue.global())
    //1
    .map{ value -> Int in
        print("1:\(Thread.current)")
        return value
    }
    // 2
    .receive(on: RunLoop.current)
    // 3
    .map{ value -> Int in
        print("2:\(Thread.current)")
        return value
    }
    .eraseToAnyPublisher()
    .sink { _ in
        //print(out)
    }
    .store(in: &subscription)
```

输出
```
1:<NSThread: 0x6000024e8680>{number = 5, name = (null)}
2:<NSThread: 0x6000024c8300>{number = 1, name = main}
1:<NSThread: 0x6000024e8680>{number = 5, name = (null)}
2:<NSThread: 0x6000024c8300>{number = 1, name = main}
```

这段代码注释2处，使用`RunLoop.current`代替`ImmediateScheduler.shared`，需要注意，`RunLoop.current`是什么？它是与在调用时处于当前状态的线程相关联的RunLoop。由于从主线程调用程序，因此`RunLoop.current`是主线程的RunLoop。

## DispatchQueue

DispatchQueue遵循scheduler协议，通过向系统管理的调度队列提交任务，可以在多核硬件上并发执行代码；DispatchQueue可以是串行（默认）或并行；

**注意:在使用subscribe(on:)、receive(on:)或任何其他接受调度程序参数的操作符时，绝不应该假定调度程序的线程每次都是相同的。**

```swift
var subscriptions = Set<AnyCancellable>()

let serialQueue = DispatchQueue(label: "Serial queue")
let sourceQueue =  DispatchQueue.main

// 1
let source = PassthroughSubject<Void, Never>()

// 2
let subscription =  sourceQueue.schedule(after: sourceQueue.now,
                                        interval: .seconds(1)) {
  source.send()
}

source
    .map{ _ in
        print("1:\(Thread.current)")
    }
    .receive(on: serialQueue,options: DispatchQueue.SchedulerOptions(qos: .userInteractive))
    .map{ _ in
        print("2:\(Thread.current)")
    }
    .eraseToAnyPublisher()
    .sink(receiveValue: { _ in
        
    })
    .store(in: &subscriptions)
```

输出

```
1:<NSThread: 0x600002490740>{number = 1, name = main}
2:<NSThread: 0x600002486340>{number = 6, name = (null)}
1:<NSThread: 0x600002490740>{number = 1, name = main}
2:<NSThread: 0x6000024a4780>{number = 4, name = (null)}
1:<NSThread: 0x600002490740>{number = 1, name = main}
2:<NSThread: 0x600002486340>{number = 6, name = (null)}
1:<NSThread: 0x600002490740>{number = 1, name = main}
2:<NSThread: 0x6000024a4780>{number = 4, name = (null)}
```

从输出结果可以看到DispatchQueue无法保证执行在哪个线程上，source发出后是指定在main，当切到serialQueue后，执行在一个串行队列serialQueue上，但是不能确定保持不变，这个例子中一会是6，一会是4；另外，你也会注意到DispatchQueue是唯一有option可设置的，有两种：
* .userInteractive： 用户操作的重要任务，os需要优先处理
* .background：后台任务

如果改动一下上面代码，把sourceQueue从DispatchQueue.main改成serialQueue，即前后切换的两个queue保持一致
```swift
let sourceQueue =  serialQueue //DispatchQueue.main
```
输出
```
1:<NSThread: 0x60000308c080>{number = 3, name = (null)}
2:<NSThread: 0x60000308c080>{number = 3, name = (null)}
1:<NSThread: 0x600003085300>{number = 4, name = (null)}
2:<NSThread: 0x600003085300>{number = 4, name = (null)}
```
线程number就会一直保持不变，即一直在一个线程上执行



## OperationQueue

```swift
let queue = OperationQueue()

let subscription = (1...10).publisher
  .receive(on: queue)
  .sink { value in
    print("Received \(value)")
  }
```
输出
```
Received 1
thread = <NSThread: 0x600002af80c0>{number = 3, name = (null)}
Received 2
thread = <NSThread: 0x600002ac4f80>{number = 5, name = (null)}
Received 3
thread = <NSThread: 0x600002ac5480>{number = 7, name = (null)}
Received 7
thread = <NSThread: 0x600002ac4680>{number = 8, name = (null)}
Received 6
thread = <NSThread: 0x600002af4780>{number = 9, name = (null)}
```

OperationQueue在底层使用OperationQueue执行任务，所以不保证具体执行在哪个线程；另外OperationQueue的maxConcurrentOperationCount属性也很明确的可以并发执行，所以输出结果的顺序并不保证；

简单修改一下，增加
```swift
queue.maxConcurrentOperationCount = 1
```

由于设置了最大并行操作计数等于1，所以等价串行队列，此时结果就会按顺序输出

```
Received 1 on thread 3
Received 2 on thread 3
Received 3 on thread 3
Received 4 on thread 3
Received 5 on thread 4
Received 6 on thread 3
Received 7 on thread 3
Received 8 on thread 3
Received 9 on thread 3
Received 10 on thread 3 
```

# 总结
本篇中使用了大量代码讲解Combine中Schedulers的运行机制，当你需要执行一些耗时或资源消耗型操作，就需要考虑适当使用Schedulers进行合理的任务调度，避免阻塞主线程；通过定义Scheduler这个新的概念，提醒使用者调度的核心是合理的选择调度逻辑（主线程或子线程），但是不能具体指定在哪个线程；同时，苹果提供了Scheduler协议的多种具体实现，一般来说，更推荐使用DispatchQueue。


# 参考

* https://developer.apple.com/documentation/combine/scheduler
* https://www.vadimbulavin.com/understanding-schedulers-in-swift-combine-framework/
* Combine_Asychronous_Programming_with_Swift，https://www.raywenderlich.com/
* Practical Combine，https://practicalcombine.com/