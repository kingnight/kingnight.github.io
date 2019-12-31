# 1.引言

本文将介绍Swift4.0开始引入的新特性`Codable`，它能够将程序内部的数据结构序列化成可交换数据，也能够将通用数据格式反序列化为内部使用的数据结构，大大提升对象和其表示之间互相转换的体验。

# 2.简介


`Codable`协议在Swift4.0开始被引入，目标是取代现有的`NSCoding`协议，它对结构体，枚举和类都支持，能够把JSON这种弱类型数据转换成代码中使用的强类型数据，同时由于编译器的帮助，可以使开发者少写很多重复代码。

`Codable`是一种混合类型，由`Decodable`和`Encodable`协议构成。
`Decodable`协议定义了一个初始化函数：
```swift
init(from decoder: Decoder) throws
```
遵从`Decodable`协议的类型可以使用任何`Decoder`对象进行初始化,完成一个解码过程。
`Encodable`协议定义了一个方法：

```swift
func encode(to encoder: Encoder) throws
```
任何`Encoder`对象都可以创建遵从了`Encodable`协议类型的表示，完成一个编码过程。
由于Swift标准库中的类型，比如`String`，`Int`，`Double`和 Foundation 框架中`Data`，`Date`，`URL`都是默认支持`Codable`协议的，所以只需声明支持协议即可。我们以常见的学生信息为例，id代表编号，name代表姓名，grade代表年级，构成一个最简单对象。

```swift
struct Student{
	var id:String
	var name:String
	var grade:Int
}
```
如果需要Student这个类型支持`Codable`协议，只需增加遵从`Codable`协议
```swift
struct Student:Codable{
	var id:String
	var name:String
	var grade:Int
}
```
那么现在Student类型就会默认支持`Codable`的`init(from:)` 和 `encode(to:)`方法，即使他们没有声明这些方法。

接下来我们将以最常用的JSON格式为例，介绍编码和解码过程。

本文的所有代码均在Xcode10，Swift4.2环境下测试通过。

## 2.1解码
```swift
{
	"id": "127182781278",
	"name": "小明",
	"grade": 1
}
```
这段JSON数据与Student结构的字段一一对应，我们现在使用系统提供的`JSONEncoder`来解码数据
```swift
let student = try JSONDecoder().decode(Student.self,from: json)
print(student)
//输出：Student(id: "127182781278", name: "小明", grade: 1)
```
整个过程非常简单，创建一个解码器，这个解码器的decode方法需要传入两个参数，第一个参数指定JSON转成的数据结构的类型，这个类型是将弱类型转换成强类型的关键，第二个参数传入原始的data数据。
## 2.2编码
编码过程与解码过程基本对应，系统提供了一个`JSONEncoder`对象用于编码。先创建一个对象并传入相同参数；
```swift
let student2 = Student(id: "127182781278", name: "小明", grade: 1)
```
创建编码器，然后传入值给它进行编码，编码器通过Data实例的方式返回一个字节的集合，这里我们为了方便显示，将它转为了字符串。
```swift
do {
   let jsonData = try JSONEncoder().encode(student2)
   let jsonString = String(decoding: jsonData, as: UTF8.self)
   print(jsonString)
}catch{
    print(error.localizedDescription)
}
//输出：{"id":"127182781278","name":"小明","grade":1}
```
解码和编码过程比起来，解码过程遇到的问题会多得多，比如数据缺失，类型错误，业务场景复杂带来的单个接口数据格式变化等等；所以下面的内容将更加深入的介绍解码过程；

# 3.应用场景

JSON格式是网络上最常用的数据传输格式，Web服务返回的数据一般都是JSON格式，而`Codable`能够很好的把JSON数据转换成应用内使用的数据格式，替代了传统的手工编写代码解码的方式，可以减少大量重复无意义代码的书写，提高效率；

另一方面，由于Codable协议被设计出来用于替代`NSCoding`协议，所以遵从`Codable`协议的对象就可以无缝的支持`NSKeyedArchiver`和`NSKeyedUnarchiver`对象进行Archive&UnArchive操作，把结构化的数据通过简单的方式持久化和反持久化，原有的解码过程和持久化过程需要单独处理，现在通过新的`Codable`协议一起搞定，大大提高了效率。

# 4.使用技巧


## 4.1嵌套对象，数组和字典
由于Swift4支持条件一致性，所有当数组中每个元素遵从`Codable`协议，字典中对应的`key`和`value`遵从`Codable`协议，整体对象就遵从`Codable`协议。

```swift
//swift/stdlib/public/core/Codable.swift.gyb
extension Array : Decodable where Element : Decodable {
    // ...
}

//swift/stdlib/public/core/Codable.swift.gyb
extension Dictionary : Decodable where Key : Decodable, Value : Decodable {
    // ...
}
```
下面有一个班级对象的数据，由班级编号和学生成员组成
```swift  
 {
    "classNumber":"101",
    "students":[
          {
             "id": "127182781278",
             "name": "小明",
             "grade": 1
          },
          {
             "id": "216776999999",
             "name": "小王",
             "grade": 1
          }
    ]
}
```
那么我们可以定义一个这样的对象Class，classNumber代表班级编号，students代表学生数组，根据上述提到的一致性规则，即可完成解码工作
```swift
struct Class:Codable{
   var classNumber:String
   var students:[Student]
}
```

## 4.2空对象或空值

### 4.2.1空对象
在复杂业务场景下，很可能我们需要处理的的数据结构比较复杂，不同的字段`key`对应相同的数据结构，但是可能有值，也可能只是返回空值，比如有这样两个字段`firstFeed`和`sourceFeed`具有相同的JSON结构，但是由于业务需要，在不同场景下`firstFeed`可能有值（结构与`sourceFeed`一致），也有可能没有值，返回空对象`{}`，这时应该如何处理呢？

```swift
{
	"firstFeed": {},
	"sourceFeed": {
		"feedId": "408255848708594304",
		"title": "“整个宇宙的星星都在俯身望你” 🌟🌟🌟\nphoto by overwater"
	 }
}
```
根据以往的经验，我们尝试使用下面的方式去解析：
```swift
class SourceFeed: Codable{
    public var feedId: String
    public var title: String
}

class Feed: Codable {
    public var firstFeed: SourceFeed? 
    public var sourceFeed: SourceFeed
}

var decoder = JSONDecoder()
let feed = try decoder.decode(Feed.self, from: json)

print(feed.firstFeed)
print(feed.sourceFeed.feedId)

```
如果你运行会发现错误，提示`firstFeed`没有指定的`key`存在
  ▿ keyNotFound : 2 elements
    - .0 : CodingKeys(stringValue: "feedId", intValue: nil)
    ▿ .1 : Context
      ▿ codingPath : 1 element
        - 0 : CodingKeys(stringValue: "firstFeed", intValue: nil)
      - debugDescription : "No value associated with key CodingKeys(stringValue: \"feedId\", intValue: nil) (\"feedId\")."
      - underlyingError : nil

这时我们需要调整`SourceFeed`的写法

```swift
class SourceFeed: Codable{
    public var feedId: String?
    public var title: String?
}
```
把`SourceFeed`的每个属性设置为可选值，这样由于`Feed`对象中的`firstFeed`也是可选值，就可以在`firstFeed`返回空对象`{}`时，自动解析为`nil`

### 4.2.2空值null
空值`null`经常会被服务器返回，如果使用`Objective-C`，`null`被默认转换为`NSNull`，如果没有经过检查类型而进行强制类型转换，很容易造成Crash，Swift语言引入`Codable`后，可以将可能为`null`的值的类型设置为可选，这样`Codable`可以自动将`null`映射为`nil`，很容易就解决了这个问题。

还以上面的SourceFeed为例
```
{
	"firstFeed": null,
	"sourceFeed": {
		"feedId": "408255848708594304",
		"title": "“整个宇宙的星星都在俯身望你” 🌟🌟🌟\nphoto by overwater"
	 }
}
```
```
class SourceFeed: Codable{
    public var feedId: String
    public var title: String
}

class Feed: Codable {
    public var firstFeed: SourceFeed? 
    public var sourceFeed: SourceFeed
}

var decoder = JSONDecoder()
let feed = try decoder.decode(Feed.self, from: json)

print(feed.firstFeed)
print(feed.sourceFeed.feedId)
//输出结果
//nil
//408255848708594304

```

## 4.3字段匹配

### 4.3.1处理键名和属性名不匹配

Web服务中使用JSON时一般使用蛇形命名法（snake_case），把名称转换为小写字符串，并用下划线（_）代替空格来连接这些字符，与此不同的是Swift API设计指南中预先把对类型的转换定义为UpperCamelCase，其他所有东西都定义为  lowerCamelCase。

由于这种需求十分普遍，在Swift4.1时`JSONDecoder`添加了`keyDecodingStrategy`属性，可以在不同的书写惯例之间方便地转换。
```swift
var decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
```
如果有这样的键值chat_message，就会转换成chatMessage。

但是还可能有特殊情况，Web服务的开发者可能某些时候大意了，也没有遵守蛇形命名法，而是很随意的处理了，那么如果我们想对键值进行校正，该如何处理？

解决办法是：使用 CodingKeys 指定一个明确的映射。Swift 会寻找符合` CodingKey` 协议的名为 CodingKeys 的子类型。这是一个标记为` private` 的枚举类型，对于名称不匹配的键指定明确的 `String` 原始值。

```swift
private enum CodingKeys: String, CodingKey {
      case template
      case chatMessage = "chat_message"
      case chatID
      case groupUserName = "group_user_name"
      case groupId
 }
```



### 4.3.2两端键值不匹配

很多时候Web服务下发的键值信息都是最基本的信息，客户端需要加工和处理这些信息使其方便可用。

```swift
{
     "createTime": "2018-08-23 11:11:56.659",
     "repostsCount": 0,
     "tag": “205"
}
```

上面这段JSON数据代表转发消息所需记录的时间，转发消息数量和标记；对应的数据结构如下：

```swift
struct Feed: Decodable {
      var createTime : String?
      var repostsCount :Int
      var tag: String?
}
```

两者之间一一对应，符合要求；但是，进一步考虑，在业务展示时候，我们需要根据createTime提供的数据进行进一步处理，展示成“刚刚”，“XX分钟之前”，“XX小时之前”等更易读的样式供用户查看，所以需要增加一个键值formatTime用于记录展示信息，但是这个信息不是Web服务提供的，所以如果加入后

```
struct Feed: Decodable {
      var createTime : String?
      var repostsCount :Int
      var tag: String?
      var formatTime:String?
}
```

使用`Codable`解码时就会失败，提示keyNotFound。这时刚刚提到的`CodingKeys`派上了用场，我们重写`CodingKeys`

```swift
private enum CodingKeys: String, CodingKey {
      case createTime
      case repostsCount 
      case tag
 }
```

枚举值中仅仅列出Web服务下发我们需要解码的字段，忽略formatTime，这样就饿可以正常解码

还有一种可能，Web服务下发的键值字段很多，我们只需要解析其中一部分，其他的都用不到，比如：

```swift
{
     "createTime": "2018-08-23 11:11:56.659",
     "repostsCount": 0,
     "tag": “205”
     “id” :  “23623263636633"
}
```

其中的id键值无需用到，但是Web服务下发了，这时处理起来很简单

```swift
struct Feed: Decodable {
      var createTime : String?
      var repostsCount :Int
      var tag: String?
}
```

只需Feed对象中列出需要解析的键值，并且无需重写`CodingKeys`，`Codable`解码时会自动忽略掉多余的键值。



## 4.4定制日期格式处理
我们经常需要需要跟日期打交道，你上次发帖的日期，浏览文章的日期，倒计时的天数等等，这些数据可能以不同形式展现下发，最常见的日期标准是[ISO8601](https://zh.wikipedia.org/wiki/ISO_8601)和[RFC3339](https://tools.ietf.org/html/rfc3339)，举例来说，

```swift
1985-04-12T23:20:50.52Z          //1
1996-12-19T16:39:57-08:00        //2
1996-12-20T00:39:57Z             //3
1990-12-31T23:59:60Z             //4
1990-12-31T15:59:60-08:00        //5
1937-01-01T12:00:27.87+00:20     //6
```

上面这些都是日期表示格式，但是只有第二个和第三个示例是Swift中`Codable`可以解码的，我们首先来看如何解码
```swift
{
	"updated":"2018-04-20T14:15:00-0700"
}
```

```swift
struct Feed:Codable{
     var updated:Date
}
    
let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .iso8601
let feed = try! decoder.decode(Feed.self, from: json)
print(feed.updated)
//输出：2018-04-20 21:15:00 +0000
```

`JSONDecoder` 提供了一个方便的机制可以解析日期类型，根据你的需求设置一下` dateDecodingStrategy `属性为iso8601就可以解码符合标准（ISO8601DateFormatter）的日期格式了。

另一种常用的日期格式是时间戳(timestamp)，时间戳是指格林威治时间1970年01月01日00时00分00秒起至现在的总秒数。

```swift
{
	"updated":1540650536
}
```
```swift
struct Feed:Codable{
	var updated:Date
}
    
let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .secondsSince1970  //如果是毫秒，使用.millisecondsSince1970
let feed = try! decoder.decode(Feed.self, from: json)
print(feed.updated)
//输出：2018-10-27 14:28:56 +0000
```

解码时间戳格式日期需要将`JSONDecode`r的`dateDecodingStrategy`设置为secondsSince1970（秒为单位）或millisecondsSince1970（毫秒为单位）。

那么如果不是刚才提到的可以默认支持的解码格式怎么办？`JSONDecoder`对象也提供了定制化方式：我们以前面提到的第一种格式为例，1985-04-12T23:20:50.52Z ，通过扩展`DateFormatter`定义一个新的iso8601Full，把这个作为参数传入`dateDecodingStrategy`

```swift
extension DateFormatter {
    static let iso8601Full: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss.SSSZZZZZ"
        formatter.calendar = Calendar(identifier: .iso8601)
        formatter.timeZone = TimeZone(secondsFromGMT: 0)
        formatter.locale = Locale(identifier: "en_US_POSIX")
        return formatter
    }()
}

let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .formatted(DateFormatter.iso8601Full)
let feed = try! decoder.decode(Feed.self, from: json)
print(feed.updated)
```
我们可以翻看官方文档是这样描述的：

```swift
JSONDecoder.DateDecodingStrategy.formatted(_:)

The strategy that defers formatting settings to a supplied date formatter.
```

提供一个定制化的日期格式工具，你可以根据需求定制解码格式。

## 4.5枚举值

在信息流类型的移动端产品中，一定会有一个模版类型的字段，用以区别展现的卡片样式，是图片，视频，链接还是广告等等，我们在代码中使用时，一般更希望将模版类型转换成枚举类型，方便使用。下面举两个简单的例子来说明如何从字符串或者整型数据转换成枚举类型。
###4.5.1从字符串解析枚举类型
```swift
{
	"feedId":"100000",
	"template": "video"
}
```
其中template代表模版类型，是字符串类型
```swift
    struct Feed:Codable {
        var feedId:String
        var template:FeedTemplate
    }
    
    enum FeedTemplate:String,Codable{
        case FEED_VIDEO = "video"
        case FEED_PIC = "pic"
        case FEED_LINK = "link"
    }
    
    let feed = try! JSONDecoder().decode(Feed.self, from: json)
    print(feed.feedId)
    print(feed.template)
    //输出
    //100000
	//FEED_VIDEO
```
首先创建FeedTemplate枚举类型，它的原始值是`String`类型，并且遵从`Codable`协议，列举出所有可能的类型和对应的字符串值，然后在Feed类型中定义template字段的类型为FeedTemplate，就可以完成从数据到枚举类型的转换。

### 4.5.2从整型解析枚举类型
从整型数据解码枚举类型与字符串非常类似，不同在于指定枚举类型的原始类型。
```swift
{
	"feedId":"100000",
	"template": 1
}
```

```swift
    struct Feed:Codable {
        var feedId:String
        var template:FeedTemplate
    }
    
    enum FeedTemplate:Int,Codable{
        case FEED_VIDEO = 1
        case FEED_PIC = 2
        case FEED_LINK = 3
    }
    
    let feed = try! JSONDecoder().decode(Feed.self, from: json)
    print(feed.feedId)
    print(feed.template)
```

## 4.6动态键值结构

很多时候由于产品功能的需要，Web服务通常会下发动态结构数据，比如下面这段简化的JSON结构

```swift
{
	"template":"video",
 	"videoFeed":{
 		"vid":"1234",
        "url":"http://www.baidu.com",
        "coverPic":"http://www.baidu.com/pic.png"
     },
     "picFeed":{
         "content":"今天天气不错哦",
         "pics":{
                "width":100,
                "height":200
          }
      },
      "linkFeed":{
          "title":"四季沐歌",
          "url":"http://www.google.com"
      }
}
```

其中，template代表模版类型，有三种可能video，pic，link；同一个时刻，Web服务只会下发一种数据。

比如视频模式时

```swift
{
	"template":"video",
 	"videoFeed":{
 		"vid":"1234",
        "url":"http://www.baidu.com",
        "coverPic":"http://www.baidu.com/pic.png"
     }
}
```

图文模式时

```swift
{
	"template":"pic",
     "picFeed":{
         "content":"今天天气不错哦",
         "pics":{
                "width":100,
                "height":200
          }
      }
}
```

如果想要处理好这种动态数据结构，那么就必须要重写`init`方法和`encode`方法了。

为了简化问题，这里只实现`init`方法：

```swift
struct Feed:Codable {
    var template:FeedTemplate
    var videoFeed:VideoFeed?
    var picFeed:PicFeed?
    var linkFeed:LinkFeed?
        
    private enum CodingKeys:String,CodingKey{
        case template
        case videoFeed
        case picFeed
        case linkFeed
     }
        
     init(from decoder: Decoder) throws {
         let container = try decoder.container(keyedBy: CodingKeys.self)
         template = try container.decode(FeedTemplate.self, forKey: .template)
         do {
             videoFeed = try container.decodeIfPresent(VideoFeed.self, forKey: .videoFeed)
         } catch {
             videoFeed = nil
         }
         do {
             picFeed = try container.decodeIfPresent(PicFeed.self, forKey: .picFeed)
         } catch {
             picFeed = nil
         }
         do {
             linkFeed = try container.decodeIfPresent(LinkFeed.self, forKey: .linkFeed)
         } catch {
             linkFeed = nil
         }
     }
}
    
struct VideoFeed:Codable {
     var vid:String
     var url:String
     var coverPic:String
}
    
struct PicFeed:Codable {
     var content:String
     var pics:PicFeedImage
}
    
struct PicFeedImage:Codable{
     var width:Int
     var height:Int
}
    
struct LinkFeed:Codable{
     var title:String
     var url:String
}
    
enum FeedTemplate:String,Codable{
     case FEED_VIDEO = "video"
     case FEED_PIC = "pic"
     case FEED_LINK = "link"
}
```

其中 出现了我们之前没有提到的`decodeIfPresent`方法。当不确定键值是否会存在时，在设置属性时把这个属性设置为可选，然后使用`decodeIfPresent`这个方法会查找该键值是否存在，如果存在就decode，如果不存在就会返回`nil`，这样就可以简单处理动态数据结构造成的问题。

## 4.7特殊类型
很多时候我们希望一个结构体或者对象遵循`Codable`协议，但是其中的某个属性不支持`Codable`协议，这时我们就需要特殊处理。例如`CLLocationCoordinate2D`是我们常用的定义经纬度的结构体，但是它并不遵循`Codable`协议。当我们想要定义一个目的地的描述，包含两个参数：目的地名称和经纬度坐标，你可能会想通过实现`encode`和`init`方法让`CLLocationCoordinate2D`遵循`Codable`协议，比如这样：

```swift
struct Destination:Codable {
    var  location : CLLocationCoordinate2D
    var  name : String
    
    private enum CodingKeys:String,CodingKey{
        case latitude
        case longitude
        case name
    }
    
    public func encode(to encoder: Encoder) throws {
        var container = try! encoder.container(keyedBy:CodingKeys.self)
        try container.encode(name,forKey:.name)
	    //编码纬度
        try container.encode(location.latitude,forKey:.latitude)
		//编码经度
        try container.encode(location.longitude,forKey:.longitude)
    }
    
    public init(from decoder: Decoder) throws {
        var latitude: CLLocationDegrees
        var longitude: CLLocationDegrees
        let container = try decoder.container(keyedBy: CodingKeys.self)
        latitude = try container.decode(CLLocationDegrees.self,forKey:.latitude)
        longitude = try container.decode(CLLocationDegrees.self,forKey:.longitude)
        self.location = CLLocationCoordinate2D(latitude:latitude,longitude:longitude)
        self.name = try container.decode(String.self,forKey:.name)
    }
}
```

这样做看起来好像没什么问题，但是如果将来苹果升级了系统，使得`CLLocationCoordinate2D`也遵从`Codable`协议，那么很可能与我们的实现产生冲突。

所以我们不妨来换一种方式：因为`CLLocationCoordinate2D`其实只有两个属性，我们不妨重新定一个Coordinate对象，具有这两个相同的属性，我们可以使用这个新Coordinate替代原来的`CLLocationCoordinate2D`，同时实现了两者之间的互相转换，很好的解决了问题，当未来`CLLocationCoordinate2D`也遵循`Codable`协议时不会造成任何影响。

```swift
struct Destination:Codable {
    var  location : Coordinate
    var  name : String
}

struct Coordinate: Codable {
    let latitude, longitude: Double
}

extension CLLocationCoordinate2D {
    init(_ coordinate: Coordinate) {
        self = .init(latitude: coordinate.latitude, longitude: coordinate.longitude)
    }
}

extension Coordinate {
    init(_ coordinate: CLLocationCoordinate2D) {
        self = .init(latitude: coordinate.latitude, longitude: coordinate.longitude)
    }
}
```

# 5总结

在Swift中引入`Codable`，就可以使用很少的代码为你程序中的内部数据结构和通用数据之间架起桥梁，实现无缝转换。它可以将JSON松散的结构与强类型之间建立关系，简化你的开发成本。在这篇文章中，我们从使用者的角度介绍了各种场景下可能出现的问题和对应的解决方案；由于篇幅的限制，很多问题无法展开详细的分析，希望没有使用过的小伙伴能够尝试去使用`Codable`，已经使用过的小伙伴能够通过我们分享的经验能有所收获。


# 参考文献

【1】https://developer.apple.com/documentation/swift/codable

【2】<https://developer.apple.com/documentation/foundation/archives_and_serialization/using_json_with_custom_types>

【3】https://github.com/xitu/gold-miner/blob/master/TODO/ultimate-guide-to-json-parsing-with-swift-4.md