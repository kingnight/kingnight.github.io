在SwiftUI中，以单一数据源（single source of truth）为核心，构建了数据驱动状态更新的机制。其中引入了多种新的属性包装器（property wrapper），用来进行状态管理。本篇主要介绍@State和@Binding，将从简单的使用入手，通过一系列具体的代码实例展示它们的使用场景，并进步一探索State的内部实现

环境
* MacOS 10.15.5
* Xcode 12.0 beta 

# State 

>  A property wrapper type that can read and write a value managed by SwiftUI.

@State是一个属性包装器（property wrapper），被设计用来针对值类型进行状态管理；用于在Struct中mutable值类型

```swift
struct User {
    var firstName = "Bilbo"
    var lastName = "Baggins"
}

struct ContentView: View {
    @State private var user = User()  //1

    var body: some View {
        VStack {
            Text("Your name is \(user.firstName) \(user.lastName).")  //2
            TextField("First name", text: $user.firstName) //3
            TextField("Last name", text: $user.lastName)
        }
    }
}
```
* 对于 @State 修饰的属性的访问，只能发生在 body 或者 body 所调用的方法中。你不能在外部改变 @State 的值，只能@State初始化时，设置初始化值，如注释1处所示，它的所有相关操作和状态改变都应该是和当前 View 生命周期保持一致。
* 在引用包装为@State的属性是，如果是读写都有，引用属性需要$开头（注释3处），如果只读直接使用变量名即可（注释2处）
* State针对具体View的内部变量进行管理，不应该从外部被允许访问，所以应该标记为private（注释1处）

但是，如果把`struct User`替换为`class User`将会无效，为什么呢？

## State检测的是值类型

* 值类型仅有独立的拥有者，而class类型可以多个指向一个；对于两个SwiftUI View而言，即使发送给他们两个相同的struct对象，事实上他们每个View都得到了一份独立的struct的拷贝，所以其中一个View的struct值发生变化，对另一个没有影响；反之，如果是class则会互相影响；
* 当User是一个结构体时，每次我们修改这个结构体的属性时，Swift实际上是在创建一个新的结构体实例。@State能够发现这个变化，并自动重新加载我们的视图。现在如果改为class，我们有了一个类，这种行为就不再发生，Swift可以直接修改值。

还记得我们如何使用mutating关键字来修改结构方法的属性吗?
```swift
struct User {
	var name:String
	mutating func changeName(name:String) {
      self.name = name
  }
}
```
这是因为如果我们创建了作为变量的结构体属性，但结构体本身是常量，我们不能更改属性；当属性发生变化时，Swift需要能够销毁并重新创建整个结构体，而这对于常量结构体是不可能的。类不需要mutating关键字，因为即使类实例被标记为常量，Swift仍然可以修改变量属性。

如果User是一个类，属性本身就不会改变，所以@State不会注意到任何东西，也无法重新加载视图。即使类内的某个属性值发生变化，但@State不监听这些，所以视图不会被重新加载。

如果想要改变这种情况，使得class类被监听到变化，就不能使用State，需要使用@ObservedObject或@StateObject

# Binding
> A property wrapper type that can read and write a value owned by a source of truth.

Binding的作用是在保存状态的属性和更改数据的视图之间创建双向连接，将当前属性连接到存储在别处的单一数据源（single source of truth），而不是直接存储数据。将存储在别处的值语意的属性转换为引用语义，在使用时需要在变量名加$符号。

通常使用场景是把当前View中的@State值类型传递给其子View，如果直接传递State值类型，将会把值类型复制一份copy，那么如果子View中对值类型的某个属性进行修改，父View不会得到变化，所以需要把State转成Binding传递。

@Binding 修饰属性无需有初始化值，Binding可以配合@State或ObservableObject对象中的值属性一起使用，注意不是@ObservedObject属性包装器

```swift
struct Product:Identifiable {
    var isFavorited:Bool
    var title:String
    var id: String
}

struct FilterView: View {
    @Binding var showFavorited: Bool  //3

    var body: some View {
        Toggle(isOn: $showFavorited) {  //4
            Text("Change filter")
        }
    }
}

struct ProductsView: View {
    let products: [Product] = [
    Product(isFavorited: true, title: "ggggg",id: "1"),
    Product(isFavorited: false, title: "3333",id: "2")]

    @State private var showFavorited: Bool = false   //1

    var body: some View {
        List {
            FilterView(showFavorited: $showFavorited)  //2

            ForEach(products) { product in
                if !self.showFavorited || product.isFavorited {
                    Text(product.title)
                }
            }
        }
    }
}
```
这个例子展示了一个有过滤开关的列表，为了简化内容说明核心问题，只有两行内容，父视图是ProductsView，其中嵌套着子视图FilterView和列表元素，为了能够使得FilterView中对showFavorited的修改能够传递回父视图：
* 注释1，showFavorited使用@State修饰
* 注释2，在body中通过$showFavorited获得showFavorited对应的Binding传递给子视图FilterView
* 注释3，子视图FilterView中定义了`@Binding var showFavorited: Bool`引用传入参数
* 注释4，当切换开关后，由于@Binding机制的作用，会修改外层的单一数据源（single source of truth），所以列表中展示的内容会不断根据条件进行过滤

# 可变和不可变

观察下面示例
```swift
struct StateMutableView: View {
    @State private var flag = false
    private var anotherFlag = false

    mutating func changeAnotherFlag(_ value: Bool) {
        self.anotherFlag = value
    }
    
    var body: some View {
        Button(action: {
            //1 ok
            self.flag = true
            
            //2 Cannot assign to property: 'self' is immutable
            self.anotherFlag = true
            
            //3 Cannot use mutating member on immutable value: 'self' is immutable
            changeAnotherFlag(true)
        }) {
            Text("Test")
        }
    }
}
```
flag是标记为State的变量，anotherFlag是没有使用属性包装器的普通变量，同时增加了一个mutating的方法`changeAnotherFlag`被设计修改anotherFlag；

在body中通过几种方式对两个变量进行修改，注释1-3处，分别标记了修改结果和提示错误，显然flag可以被修改，而anotherFlag不可以，这是为什么？

这里涉及两个问题：
1. 为什么可以修改flag？
2. 为什么不可以修改anotherFlag？

先来看第二个问题

## 为什么不可以修改anotherFlag

### 计算属性getter

示例5
```swift
struct SimpleStruct {
    var anotherFlag: Bool {
        _anotherFlag = true
//      ^~~~~~~~~~~~
//      error: cannot assign to property: 'self' is immutable
        return _anotherFlag
    }

    private var _anotherFlag = false
}
```
_anotherFlag存储属性，anotherFlag计算属性
在getter属性中，self默认是nonmutating，是不能被修改的，所以报错

但是，可以有例外，如果getter被特殊标记为mutating，就可以被修改
```swift
struct SimpleStruct {
    var anotherFlag: Bool {
        mutating get {
            _anotherFlag = true
            return _anotherFlag
        }
    }

    private var _anotherFlag = false
}
```
并且还需要使用SimpleStruct时，声明实例为var
```swift
var s0 = SimpleStruct()
_ = s0.anotherFlag // ok, and modifies s0

let s1 = SimpleStruct()
_ = s1.anotherFlag
//  ^~ error: cannot use mutating getter on immutable value: 's1' is a 'let' constant
```

既然可以通过添加mutating，使得计算属性get中可以修改self，那么SwiftUI中前面示例的body属性可否添加呢？

查看View协议的定义
```swift
public protocol View {

    /// The type of view representing the body of this view.
    ///
    /// When you create a custom view, Swift infers this type from your
    /// implementation of the required `body` property.
    associatedtype Body : View

    /// Declares the content and behavior of this view.
    var body: Self.Body { get }
}
```
body是set，不能被改为mutating，所以如果你改为这样下面
```swift
struct SimpleView: View {
//     ^ error: type 'SimpleView' does not conform to protocol 'View' 
    var body: some View {
        mutating get { Text("Hello") }
    }
}
```
会报错，提示没有遵守View协议

小结：不可以修改anotherFlag的原因：body计算属性的的getter不可以被修改为mutating

## 为什么可以修改flag

由于SwiftUI设计之初就是希望构建的View树保持不变，这样才能高效的渲染UI，跟踪变化，当标记为@State的变量发生变化时，变量本身由于在Struct中不能发生变化，所以通过State为例的property wrapper本质是修改当前struct之外的变量

我们看一下State的定义
```swift
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
wrappedValue就是被标记为nonmutating set，直接使用state对象是用的wrappedValue,$符号使用的projectedValue

nonmutating有什么含义？

### 计算属性setter

在setter属性中，self默认是mutating，可以被修改；我们不能给一个不可变的量赋值，可以通过声明setter nonmutating使属性可赋值，这个nonmutating关键字向编译器表明，这个赋值过程不会修改这个struct本身，而是修改其他变量。

```swift
struct SimpleStruct {
    var anotherFlag: Bool {
        mutating get {
            _anotherFlag = true
            return _anotherFlag
        }
    }

    private var _anotherFlag: Bool {
        get {
            return UserDefaults.standard.bool(forKey: "storage")
        }
        nonmutating set {
            UserDefaults.standard.setValue(newValue, forKey: "storage")
        }
    }
}

let s0 = SimpleStruct()
var s1 = s0
_ = s1.anotherFlag // 同时影响s0和s1，他们内部的_anotherFlag都发生了变化
```
这个例子当中_anotherFlag修改了UserDefaults的值，会同时对s0和s1都产生影响，相当于起到了引用类型的作用，在实际编程中这当然是一个不好的范例，容易产生问题

小结：可以修改flag的原因，添加了property wrapper的属性，变量本身并没有变化，而是修改了由SwiftUI维护的当前struct之外的变量

# State内部实现
为了进一步深入分析，我们继续展示一个相对完整的例子

![stateDemo](/assets/images/stateDemo.png)

* 为了分析变量状态，在16行，User结构体init方法；39行，ContentView的init方法结束；47行，按钮点击执行函数部分，都加入了断点

* 由于@State针对值类型，为了打印出struct的地址，增加了address函数

* dump系统函数，能够打印出变量内部结构

  <img src="/assets/images/stateDemoRun.png" alt="stateDemoRun" style="zoom:60%;" />

运行界面如上图所示，本文输入框可以修改name，Count+1按钮使得count计数加1

打开断点，从头开始执行代码，首先执行到16行断点处，User初始化，此时self是User结构体本身

```swift
▿ User
 \- name : ""
 \- count : 0
```

继续执行到ContentView的初始化方法最后一行，此时self是ContentView，打印一下

```swift
▿ ContentView
 ▿ _user : State<User>
  ▿ _value : User
   \- name : ""
   \- count : 0
  \- _location : nil
```
出现了一个新的`_user`变量，类型是`State<User>`，这个变量内部属性`_value`类型是`User`;这意味着，加了@State属性包装器的user实例变量，由本身的`User`类型转变为一个新的`State<User>`类型，这个转变完成的新类型实例`_user`由SwiftUI负责生成和管理，它的内部包裹着真实的User实例，另外`_location`也值得注意，它目前是nil;

如果你注意到35行代码`user = User(name: "TT", count: 100)`发现它并不会改变内部`_user`，如果想要修改，只能采用下面方式，通过State提供的第二个初始化方法
```swift
_user = State(wrappedValue: User(name: "TT", count: 100))
```

与此同时，检查当前console的log输出
```swift
User init
ContentView init
140732783334216
▿ SwiftUI.State<DemoState.User>
  ▿ _value: DemoState.User
    - name: ""
    - count: 0
  - _location: nil
```
按照预期的执行顺序，User init执行，ContentView init执行，然后打印出了当前结构体的地址和`_user`内部结构

下一步，由于body执行完毕，页面渲染完整，现在点击Count+1按钮，断点停在47行，再观察内部变量情况
```swift
▿ ContentView
  ▿ _user : State<User>
    ▿ _value : User
      - name : ""
      - count : 0
    ▿ _location : Optional<AnyLocation<User>>
      ▿ some : <StoredLocation<User>: 0x600003c26a80>
```
`_location`不再是nil

```swift
140732783330824
▿ SwiftUI.State<DemoState.User>
  ▿ _value: DemoState.User
    - name: ""
    - count: 0
  ▿ _location: Optional(SwiftUI.StoredLocation<DemoState.User>)
    ▿ some: SwiftUI.StoredLocation<DemoState.User> #0
```
user的地址也发生了变化，开始时创建的user被销毁又重新创建了，这是因为@State 修饰的属性的它的所有相关操作和状态改变都应该是和当前视图生命周期保持一致，当视图没有被初始化完成时，无法完成状态属性和视图之间的绑定关系；当视图完成初始化和建立与State修饰状态的绑定关系后，`_location`就不再是nil，其中保存了众多标记视图关系和位置的信息，这里没有全部展示出来；

再点击一次Count+1按钮，count值变为2，user的地址将持续保持不变，生命周期与视图保持一致。

通过前面的分析，已经明确内部`_user`变量的存在，下面进一步分析State内部实现中wrappedValue和projectedValue的关系

```swift
(lldb) p _user
(State<DemoState.User>) $R6 = {
  _value = (name = "", count = 2)
  _location = 0x0000600003c26a80 {
    SwiftUI.AnyLocationBase = {}
  }
}

(lldb) p _user.wrappedValue
(DemoState.User) $R8 = (name = "", count = 2)

(lldb) p _user.projectedValue
(Binding<DemoState.User>) $R10 = {
  transaction = {
    plist = {
      elements = nil
    }
  }
  location = 0x0000600003c26a80 {
    SwiftUI.AnyLocationBase = {}
  }
  _value = (name = "", count = 2)
}
```

SwiftUI把`@State var user = User()`转换成三个属性

```swift
private var _user: State<User> = State(initialValue: User())
private var $user: Binding<User> { return _user.projectedValue }
private var user: User {
    get { return _user.wrappedValue }
    nonmutating set { _user.wrappedValue = newValue }
}
```
为什么$user是只读的？测试一下会发现修改失败
```swift
(lldb) expr $user = User(name:"",count:100)
error: <EXPR>:3:1: error: cannot assign to property: '$user' is immutable
$user = User(name:"",count:100)
^~~~~

error: <EXPR>:3:9: error: cannot assign value of type 'User' to type 'Binding<User>'
$user = User(name:"",count:100)
        ^~~~~~~~~~~~~~~~~~~~~~~
        
(lldb) expr $user.name = "Tim"
error: <EXPR>:3:7: error: cannot assign to property: '$user' is immutable
$user.name = "Tim"
~~~~~ ^

error: <EXPR>:3:14: error: cannot assign value of type 'String' to type 'Binding<String>'
$user.name = "Tim"
             ^~~~~ 
```
说明projectedValue只读属性

通过上面分析可以画出一张State内部实现属性的关系

```swift
_user:State<User>
	_value:User
		_name:String
		_count:Int
	_wrappedValue:User 
		get { _value }
		set { _value = newValue }
	_projectedValue:User 
		get { _value }
```
我们进一步可以大致写出State的部分可能实现逻辑
```swift
@propertyWrapper struct State<T> {
    var _value:T
    
    init(wrappedValue: T) {
        _value = wrappedValue
    }

    var wrappedValue: T {
        nonmutating set { _value = newValue }   
        get { _value.value }
    }

    var projectedValue: T { _value }
}
```



# 总结

* @State属性包装器针对值类型进行状态管理，用于在Struct中mutable值类型，它的所有相关操作和状态改变和当前 View 生命周期保持一致
* Binding将存储在别处的值语意的属性转换为引用语义，在使用时需要在变量名加$符号
* 添加了property wrapper的属性，变量本身并没有变化，而是修改了由SwiftUI维护的当前struct之外的变量


参考：
* https://forums.swift.org/t/why-i-can-mutate-state-var-how-does-state-property-wrapper-work-inside/27209
* https://medium.com/@kateinoigakukun/inside-swiftui-how-state-implemented-92a51c0cb5f6
* https://kateinoigakukun.hatenablog.com/entry/2019/03/22/184356


