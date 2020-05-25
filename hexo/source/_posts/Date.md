---
title: Date
tags: liu
category: swift
abbrlink: 1442
date: 2020-01-05 00:05:39
---

####  1. UTC、GMT和时间戳介绍

##### GMT

格林尼治标准时间（Greenwich Mean Time）是指位于英国伦敦郊区的皇家格林尼治天文台的标准时间，即本初子午线时间。

由于地球在椭圆轨道里的运动速度不均匀，这个时刻可能与实际的太阳时有误差，所以有了UTC时间出现。



##### UTC

协调世界时间（又称世界标准时间），与GMT一样都是本初子午时间，只不过UTC是经过协调后的世界时间，比GMT更加准确。全球一共有24个时区。

![timezone](https://raw.githubusercontent.com/CathyLy/imageForSource/master/2020/timezone.png)

而北京时间位于东八区，与格林尼治时间相差八小时，也就是我们经常在开发中遇到8小时时差的问题。



#### 2. swift中Date与Calender关系

![](https://raw.githubusercontent.com/liututu1213/imageForSource/master/2020/date-calender.png)

首先Date可以通过DateFormatter进行互转，Date通过格式化可以转化成自定义的样式，反过来同样成立。在iOS中，没办法像其他语言一样直接time.year这样的api获取年份等信息，想要获取Date的年月日信息需要使用Calende+DateCompontents来完成，反之，也可以通过DateComponents对各个组件进行组装，通过Calender对象变成date。

Calender封装了有关计算时间的系统的信息，其中定义了年的开始、长度和分割，提供了关于日历的信息和日历计算的支持。



####  3. Date

iOS中的Date是用来表示公历的GMT时间，Date是没有存储时区等信息，所以不管你在哪个时区打印的Date()都是一样的，这一点非常重要。

```swift
let date = Date()
print(date)
```

打印结果：

```
2020-01-04 10:03:01 +0000
```

Date()获取的是零时区的时间(格林尼治的时间)，而如果你系统所在的时区是东八区即北京时间，则实际系统时间与打印出来的时间相差八小时。



Date的初始化方法：

```Swift
public init()
public init(timeIntervalSinceNow: TimeInterval) //以当前时间的偏移秒数来初始化
public init(timeIntervalSince1970: TimeInterval) //以GMT时间的偏移秒数来初始化
public init(timeInterval: TimeInterval, since date: Date) //以给定秒数返回相对于另一个给定日期初始化的“日期”
...
```



时间比较相关方法：

```Swift
public func compare(_ other: Date) -> ComparisonResult
public static func == (lhs: Date, rhs: Date) -> Bool
public static func < (lhs: Date, rhs: Date) -> Bool
public static func > (lhs: Date, rhs: Date) -> Bool
public static func + (lhs: Date, rhs: TimeInterval) -> Date
...
```



##### 时间戳

时间戳是指1970-01-01 00:00:00到当前时间的秒数。

```
date.timeIntervalSince1970 
date.timeIntervalSinceNow
date.timeIntervalSince(xx)
```



#### 4. DateFormatter: Date 与String 互相转换

DateFormatter 是Formatter的子类，Formatter是将数据在字符串与特定类型的对象之间转换，目前Formatter有两个子类NumberFormatter和DateFormatter。

DateFormatter用来在日期和字符串之间的转换，DateFormatter格式其实是遵守了Unicode Technical Standard。

```swift
open var dateFormat: String!
open var dateStyle: DateFormatter.Style
open var timeStyle: DateFormatter.Style
open var locale: Locale!
open var timeZone: TimeZone!
open var calendar: Calendar! //如果未指定则使用当前用户的逻辑日历
```

下面将会来一一介绍这些相关的属性。

DateFormatter提供了许多定义好的时间格式，如dateStyle和timeStyle。

```swift
public enum Style : UInt {
    case none
    case short
    case medium
    case long
    case full
}
```

| 格式类型 |         dateStyle         |           timeStyle            |
| :------: | :-----------------------: | :----------------------------: |
|   none   |             /             |               /                |
|  short   |          1/4/20           |            7:37 PM             |
|  medium  |        Jan 4, 2020        |           7:38:51 PM           |
|   long   |      January 4, 2020      |        7:39:50 PM GMT+8        |
|   full   | Saturday, January 4, 2020 | 7:40:40 PM China Standard Time |

DateFormatter有一个local的属性，Local是与国际化相关的基础类。

#### 5. Local

Local是是与国际化相关的基础类，在OS X中，可以到"系统偏好设置->语言与地区"中设置当前的local。Local是包含所有地区的语言与文化习俗的基础类，一个Local实例包含了针对这个地区内特定一群人的所有语言文化基准，其中包括：

- 语言
- 键盘
- 数字、日期和时间格式
- 货币  如USD
- 排序和分类
- 符号、颜色与头像的使用

每一个Local的实例对应着一个地区标识符，如en_US、zh_CN等，这些标识符包含 语言码(en)和地区码(US)。

```swift
Locale.preferredLanguages //返回用户偏好设置的语言列表
```



#### 6. TimeZone

TimeZone表示时区信息，上面提到过世界上有24是个时区，任何时区都是GMT为基准，TimeZone对象所代表的时区都是相对于GMT的。

```swift
public static var current: TimeZone { get }
public init?(identifier: String) //用给定的标识符初始化timeZone对象
public init?(secondsFromGMT seconds: Int) //通过相对于GMT时间偏移量(时差)来获取时区
```



一般应用程序的默认时区，是和手机系统设置的时区一致的。

```
TimeZone.current
```

打印结果如下：

```
▿ Asia/Shanghai (current)
  - identifier : "Asia/Shanghai"
  - kind : "current"
  ▿ abbreviation : Optional<String>
    - some : "GMT+8"
  - secondsFromGMT : 28800 //相对于GMT的时差 秒
  - isDaylightSavingTime : false
```



#### 7. DateComponents

这个类可用于计算未来或过去的日期，Calender用它来实现Date和DateComponents相互转换

####  8. Calendar

在iOS中，没办法像其他语言一样直接time.year这样的api获取年份等信息，想要获取Date的年月日需要使用Calender的日历对象。Calender封装了有关计算时间的系统的信息，其中定义了年的开始、长度和分割，提供了关于日历的信息和日历计算的支持。

```swift
public static var current: Calendar { get }
public static var autoupdatingCurrent: Calendar { get } //当前系统设置的日历的值
public init(identifier: Calendar.Identifier) //Identifier 是一个枚举，提供了各种类型的日历
public func component(_ component: Calendar.Component, from date: Date) -> Int //返回一个给定date的日期组件
```



先来介绍Calender的相关的api

##### 8.1  Calendar.Identifier

```swift
public enum Identifier {
    case gregorian //公历 Calender的默认值
    case buddhist
    case chinese //农历
    case coptic
    case ethiopicAmeteMihret
    case ethiopicAmeteAlem
    case hebrew
    case iso8601
    case indian
    case islamic
    case islamicCivil
    case japanese
    case persian
    case republicOfChina
}
```



##### 8.2 Calendar.Component

```swift
public enum Component {
    case era // 特定时期的时代，取决于日期的日历系统
    case year
    case month
    case day
    case hour
    case minute
    case second
    case weekday //星期几，默认情况下周日为1，周一为2，以此类推
    case weekdayOrdinal //表示指定日期是当前这个月的第几个星期几，如2020-02-13，星期四，是这个月的第2个星期四
    case quarter //返回季度单位的数字，但不同地区季度的起始时间不同，因此此属性默认是0，除非赋值。
    case weekOfMonth //Identifier for the week of the month calendar unit.
    case weekOfYear// Identifier for the week of the year unit.
    case yearForWeekOfYear //Identifier for the week-counting year unit.
    case nanosecond
    case calendar
    case timeZone
}
```

Date类是无法单独来获取每一个元素的信息，必须使用component方法获取给定日期的日期组件即Calendar.Component，获取某个时间点对应的"年"、"月"、"日"、"第几周"、"周几"等信息。



##### 8.3 Calender相关函数

`public func minimumRange(of component: Calendar.Component) -> Range<Int>?`

```swift
let calendar = Calendar.current
let range = calendar.minimumRange(of: .day)
print(range) //Range(1..<29)
```

返回指定组件最小的限制范围，如Day Components 最小的范围是1-28



`public func maximumRange(of component: Calendar.Component) -> Range<Int>?`

```swift
let calendar = Calendar.current
let range = calendar.maximumRange(of: .day)
print(range) // Range(1..<32)
```

date所在的小组件在 大组件里的范围。



`public func range(of smaller: Calendar.Component, in larger: Calendar.Component, for date: Date) -> Range<Int>?`

```swift
let date = Date() ///"Feb 14, 2020 at 1:10 AM"
let calendar = Calendar.current
let range = calendar.range(of: .day, in: .month, for: date)
print(range) /// Range(1..<30)
```

返回指定日期所在的小组件在大组件的一个时间范围。



`public func dateInterval(of component: Calendar.Component, start: inout Date, interval: inout TimeInterval, for date: Date) -> Bool`

```swift
let date = Date() ///Feb 14, 2020 at 1:14 AM
let calendar = Calendar.current
var d: Date = Date()
var time: Double = 0

/// 当前时间所在的一周的开始时间，周日为每周的开始时间
calendar.dateInterval(of: .weekOfMonth, start: &d, interval: &time, for: date)
print(d)
print(time)
```

打印结果：

```
2020-02-08 16:00:00 +0000
604800.0
```

通过两个inout参数返回包含给定日期、组件的开始时间和时间长。

`public func ordinality(of smaller: Calendar.Component, in larger: Calendar.Component, for date: Date) -> Int?`

 返回指定的日期 指定的较小日历组件(Day)在较大日历组件(weekOfMonth) 的序号。



`public func date(byAdding components: DateComponents, to date: Date, wrappingComponents: Bool = false) -> Date?`

```swift
let date = Date() ///2020-02-13 17:29:35 +0000
let calendar = Calendar.current
var dateComponets = DateComponents()
dateComponets.day = 18
let day = calendar.date(byAdding: dateComponets, to: date)!
print(day) 2020-03-02 17:29:35 +0000

let day2 = calendar.date(byAdding: dateComponets, to: date, wrappingComponents: true)!
print(day2) ///2020-02-02 17:29:35 +0000
```

向给定日期添加组件计算出新的日期，其中`wrappingComponents`为true表示组件在增加的时候溢出时，不是向更高的组件"进位"



`public func date(from components: DateComponents) -> Date?`

用指定的DateComponents创建一个新的日期，如果匹配不到组件的日期，则返回nil。



```swift
public func dateComponents(_ components: Set<Calendar.Component>, from date: Date) -> DateComponents
public func dateComponents(in timeZone: TimeZone, from date: Date) -> DateComponent
```

返回这个日历时区所有的日历组件，第一个是传入指定的Calendar.Component数组，第二个是传入TimeZone



```swift
public func dateComponents(_ components: Set<Calendar.Component>, from start: Date, to end: Date) -> DateComponents

public func dateComponents(_ components: Set<Calendar.Component>, from start: DateComponents, to end: DateComponents) -> DateComponents
```



```
public func component(_ component: Calendar.Component, from date: Date) -> Int
```

```swift
let date = Date() //2020-02-15 07:14:06 +0000
let calendar = Calendar.current
let startOfDay = calendar.startOfDay(for: date)
print(startOfDay) //2020-02-14 16:00:00 +0000
```

返回给定日期的第一个时刻。



```swift
public func compare(_ date1: Date, to date2: Date, toGranularity component: Calendar.Component) -> ComparisonResult
public func isDate(_ date1: Date, equalTo date2: Date, toGranularity component: Calendar.Component) -> Bool 
```

component用于比较的粒度，如.hour 判断两个日期是否在相同的小时内。



```swift
public func isDate(_ date1: Date, inSameDayAs date2: Date) -> Bool
public func isDateInToday(_ date: Date) -> Bool
public func isDateInYesterday(_ date: Date) -> Bool
public func isDateInTomorrow(_ date: Date) -> Bool
public func isDateInWeekend(_ date: Date) -> Bool
```



```
public func dateIntervalOfWeekend(containing date: Date) -> DateInterval?
```



使用Calender和Components对Date进行的扩展：

```swift
extension Date {
     var startOfWeek: Date {
        var calendar = Calendar.current
        let component = calendar.dateComponents([.yearForWeekOfYear, .weekOfYear, .hour], from: self)
        calendar.firstWeekday = 2
        return calendar.date(from: component)!
    }

    var startOfCurrentMonth: Date {
         let calendar = NSCalendar.current
         let components = calendar.dateComponents([.year, .month, .hour], from: self)
         let startOfMonth = calendar.date(from: components)!
         return startOfMonth
     }
}
```

startOfWeek获取一周开始的日期，`calendar.firstWeekday = 2`用于设置一周从周一开始算起；startOfCurrentMonth获取当前日期所在的月开始时间。









