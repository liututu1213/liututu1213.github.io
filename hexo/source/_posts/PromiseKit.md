---
title: PromiseKit
category: swift
abbrlink: 4809
date: 2019-12-31 01:01:25
tags:
---

####  Promise中的swift用法

##### 使用type inference推导泛型参数

```swift
public final class Promise<T>: Thenable, CatchMixin {
  public init<U: Thenable>(_ bridge: U) where U.T == T {
    box = EmptyBox()
    bridge.pipe(to: box.seal)
  }
}
```

```swift
class EmptyBox<T>: Box<T> {}
```

在Promise.swift中，`EmptyBox`是一个泛型类，`init`方法创建`EmptyBox`的实例的时候，没有指定类型参数。

对于`EmptyBox<T>`中的`T`具体是依据Promise中对于box这个属性的定义：

```swift
let box: Box<Result<T>>
```

有了这个定义，则可以通过`EmptyBox()`这样的表达式通过编译，type inference 就会自动把EmptyBox的泛型参数推导为Result\<T>。



在某些泛型代码里，面对type inference 推导出来的类型，可以用更为简洁的方式调用init，如[Box.swift](https://github.com/mxcl/PromiseKit/blob/master/Sources/Box.swift)  中的用法：

```swift
enum Sealant<R> {
    case pending(Handlers<R>)
    case resolved(R)
}

final class Handlers<R> {
    var bodies: [(R) -> Void] = []
    func append(_ item: @escaping(R) -> Void) { bodies.append(item) }
}

class EmptyBox<T>: Box<T> {
    private var sealant = Sealant<T>.pending(.init())
}
```

`Sealant<T>.pending`创建一个enum对象，case pending有一个关联值，`Handlers<R>`，由于Handlers有默认的init的方法，可以先依靠type inference推断出`.pending()`里需要的参数类型。

```swift
//等价
private var sealant = Sealant<T>.pending(.init())
private var sealant = Sealant<T>.pending(Handlers.init())
```

`Sealant`和`Handlers`在Promise中算是原子类型，其中`Handlers`用于保存处理Promise结果的`Handlers`，泛型参数R表示Promise的结果，在Handlers的定义中的`bodies`保存了所有对这个结果感兴趣的方法，在Promise的范式里，新的closure会经过一些列方法，最红由append方法加入到bodies。

`Sealant`可以理解为Promise状态的类型，它的泛型参数`R`和关联值`Handlers`中的`R`是表示相同的含义，都是表示Promise的结果。当一个Promise还在等待结果的时候状态则为`.pending`，它的关联值就是所有对这个结果有所期待的handlers，如果Promise被满足，状态则改变为`.resolved`，此时的关联值就包含了Promise期望的值。

#### Box

在Box的继承体系中，一共有三个类：

```swift
class Box<T> {}
final class SealedBox<T>: Box<T> {}
class EmptyBox<T>: Box<T> {}
```



```swift
class Box<T> {
    func inspect() -> Sealant<T> { fatalError() }
    func inspect(_: (Sealant<T>) -> Void) { fatalError() }
    func seal(_: T) {}
}
```



`Box<T>`更像是一个protocol

-  `inspect() -> Sealant<T>`,利用它查询当前Box的状态`.pending`or`.resolved`
- `inspect(_: (Sealant<T>) -> Void)`这个方法完成操作，会根据它的closure参数中`Sealant<T>`的值的不同而不同
- `seal(_: T`用于向Box中封装值，利用这个方法，可以把box的状态从`.pending`改为`.resolved`



#### SealedBox

SealedBox表示已经密封好的盒子，指的是状态已经为`.resolved`，其中已经包含了Promise的结果，且无法再对其进行修改。

```swift
final class SealedBox<T>: Box<T> {
    let value: T

    init(value: T) {
        self.value = value
    }

    override func inspect() -> Sealant<T> {
        return .resolved(value)
    }
}
```

创建SealedBox时会直接传递一个value，表示把value密封在Box里，它唯一重写的方法`inspect`就是读取Box的状态，这个方法直接返回`.resolved(value)`。



####  EmptyBox

EmptyBox空的盒子，表示等待往里面装入值，PromiseKit中大部分的功能的实现都是使用EmptyBox来实现的。

```swift
class EmptyBox<T>: Box<T> {
    private var sealant = Sealant<T>.pending(.init()) //可以理解为盒子的封条
    private let barrier = DispatchQueue(label: "org.promisekit.barrier", attributes: .concurrent)
}
```

  EmptyBox对Box的三个方法进行了重写:

```swift
override func inspect() -> Sealant<T> {
    var rv: Sealant<T>!
    barrier.sync {
        rv = self.sealant
    }
    return rv
}
```

第一个方法是获取盒子状态的inspect，用同步的方式获取当前的sealant。



```swift
override func inspect(_ body: (Sealant<T>) -> Void) {
    var sealed = false
    barrier.sync(flags: .barrier) {
        switch sealant {
        case .pending:
            // body will append to handlers, so we must stay barrier’d
            body(sealant)
        case .resolved:
            sealed = true
        }
    }
    if sealed {
        // we do this outside the barrier to prevent potential deadlocks
        // it's safe because we never transition away from this state
        body(sealant)
    }
}

```

第二个inspect方法是主要是把盒子的状态传递给body。



```swift
override func seal(_ value: T) {
    var handlers: Handlers<T>!
    barrier.sync(flags: .barrier) {
        guard case .pending(let _handlers) = self.sealant else {
            return  // already fulfilled!
        }
        handlers = _handlers
        self.sealant = .resolved(value)
    }
    
    if let handlers = handlers {
        handlers.bodies.forEach{ $0(value) }
    }
}
```

`seal`这个方法主要的作用：

- 将盒子的状态从`.pending`编程`.resolved`
- 把盒子中包含的值变成`value`
- 通知之前注册过的关注这个变化的handlers

在seal的实现里面`barrier.sync`的作用是只有当盒子的状态还是`.pending`，才可以执行密封的动作，如果盒子已经被密封上，则直接顺序调用所有的handlers。



####  Resolver

```swift
/// An object for resolving promises
public final class Resolver<T> {
    let box: Box<Result<T>>

    init(_ box: Box<Result<T>>) {
        self.box = box
    }

    deinit {
        if case .pending = box.inspect() {
            conf.logHandler(.pendingPromiseDeallocated)
        }
    }
}
```

`Resolver`包含了`Box`对象，而Box里面封装的是`Result<T>`，创建Promise的时候，我们用`fulfill`方法，向盒子里封装的，就是`case fulfill`。而使用`reject`，封装的就是`case rejected`。

```swift
public extension Resolver {
    /// Fulfills the promise with the provided value
    func fulfill(_ value: T) {
        box.seal(.fulfilled(value))
    }
    
    /// Rejects the promise with the provided error
    func reject(_ error: Error) {
        box.seal(.rejected(error))
    }
    
}
```

上面的`Resolver`的扩展，就是用不同的值给`Resolver`内置的盒子封箱，除了具体的值外，还可以封装一个`Result<T>`、`Optional`，值得注意的是当传递的是nil，`Resolver`会自动把它解析成`.rejected`。

```swift
/// Resolves the promise with the provided value or error
func resolve(_ obj: T?, _ error: Error?) {
    if let error = error {
        reject(error)
    } else if let obj = obj {
        fulfill(obj)
    } else {
        reject(PMKError.invalidCallingConvention)
    }
}

/// Fulfills the promise with the provided value unless the provided error is non-nil
func resolve(_ obj: T, _ error: Error?) {
    if let error = error {
        reject(error)
    } else {
        fulfill(obj)
    }
}
```



#### Promise.swift

```
public final class Promise<T>: Thenable, CatchMixin {
    let box: Box<Result<T>>
}
```

Promise遵守了两个协议`Thenable`和`CatchMixin`，`Thenable`提供了用于串联Promise的接口操作，跟`Resolver`一样，Promise也持有了`Box<Result<T>>`，这个属性会注入到`Resolver`，用于处理异步操作的结果。



Promise的resolver init方法允许我们指定一个动态生成“盒子值”的过程

```Swift
public init(resolver body: (Resolver<T>) throws -> Void) {
  box = EmptyBox()
  let resolver = Resolver(box)
  do {
    try body(resolver)
  } catch {
    resolver.reject(error)
  }
}
```

以下是 example：

```swift
public func load() -> Promise<UserSession> {
  return Promise<UserSession> {
    resolver in
    do {
      let userSession = try loadUserSession()
      resolver.fulfill(userSession)
    }
    catch {
      resolver.reject(error)
    }
  }
}
```

可以看到，这个根据loadUserSession的结果分别调用fullfill和reject的闭包，就是init的body参数。

在init(resolver:)的实现里，定义了一个EmptyBox对象，并用它创建了一个Resolver。然后用这个resolver对象作为body回调的参数，因此就可以load的中的闭包调用fulfill和reject的原因。



#### Thenable

Protocol Thenable，它约定了串联Promise对象的各种后续操作。

```swift
/// Thenable represents an asynchronous operation that can be chained.
public protocol Thenable: class {
    /// The type of the wrapped value
    associatedtype T

    /// `pipe` is immediately executed when this `Thenable` is resolved
    func pipe(to: @escaping(Result<T>) -> Void)

    /// The resolved result or nil if pending.
    var result: Result<T>? { get }
}
```

关联类型T,可以理解为Promise中包装的类型，例如Promise<UserSession>中，T的类型就是UserSession。

在Promise.swift中可以看到var result: Result<T>?的实现逻辑，在Promise里，box中封装的类型是Result<T>，result的作用就是当"盒子"已经密封好的时候，把它的值取出来，如果是pending(没有密封好的时候)，否则返回nil。

```swift
/// - See: `Thenable.result`
public var result: Result<T>? {
    switch box.inspect() {
    case .pending:
        return nil
    case .resolved(let result):
        return result
    }
}
```



#### Thenable的扩展

通过Thenable的一系列扩展接口来实现"Promise"串联操作。

```swift
public func then<U>(
    on: DispatchQueue? = conf.Q.map,
    flags: DispatchWorkItemFlags? = nil,
    _ body: @escaping (Self.T) throws -> U) -> PromiseKit.Promise<U.T> where U : PromiseKit.Thenable
```

**on**表示then串联的闭包执行的队列，它有一个默认值，是在主线程;

**flag**表示队列中任务的配置，默认是nil;

**body**就是在Promise对象之后，串联的closure，用Promise中封装的值作为闭包的参数，返回值U是一个新的遵从Thenable的类型，通常我们会返回一个新的Promise类型或者Guarantee对象。

then 函数返回的是一个新的Promise，这个Promise封装的值就是body参数中返回的U中的关联类型。



#### `then`的实现

```swift
func then<U: Thenable>(on: DispatchQueue? = conf.Q.map, flags: DispatchWorkItemFlags? = nil, _ body: @escaping(T) throws -> U) -> Promise<U.T> {
    let rp = Promise<U.T>(.pending)
    pipe {
        switch $0 {
        case .fulfilled(let value):
            on.async(flags: flags) {
                do {
                    let rv = try body(value)
                    guard rv !== rp else { throw PMKError.returnedSelf }
                    rv.pipe(to: rp.box.seal)
                } catch {
                    rp.box.seal(.rejected(error))
                }
            }
        case .rejected(let error):
            rp.box.seal(.rejected(error))
        }
    }
    return rp
}
```

首先创建了一个返回值为Promise<U.T>:

```
let rp = Promise<U.T>(.pending)  // Promise<U.T>.init(.pending)
```

.pending创建的不是一个Box，实际上是调用了以下的init方法

```swift
init(_: PMKUnambiguousInitializer) {
  box = EmptyBox()
}
```

以上方法创建了一个空的盒子，这个盒子将要封装的值类型就是Result<U.T>。



接下来就是pipe方法的调用，这个方法可以大概理解为当上游的Promise的期望达成之后，就调用then提供的闭包。

```swift
public func pipe(to: @escaping(Result<T>) -> Void) {
    switch box.inspect() { 
    case .pending:
        box.inspect {
            switch $0 {
            case .pending(let handlers):
                handlers.append(to)
            case .resolved(let value):
                to(value)
            }
        }
    case .resolved(let value):
        to(value)
    }
}

```

 `box.inspect`读取了当前Promise中盒子的状态，如果box是resolved状态则直接把这个值传递给pipe的closure方法，如果是还在等待异步操作的结果，则调用box的另外的一个inspect版本，这个版本的 `box.inspect`会把当前Box的状态，传递给`inspect`的closure参数，并执行这个closure。

如果此时Promise还未得到期望的结果，就会把pipe的closure加入到handlers里面，否则则直接执行`to(value)`，所以当Promise一旦得到期望的结果就立即回调用pipe提供的closure执行。

在了解pipe函数之后，再来看一下then函数中对pipe的使用

```swift
pipe {
    switch $0 {
    case .fulfilled(let value):
        on.async(flags: flags) {
            do {
                let rv = try body(value)
                guard rv !== rp else { throw PMKError.returnedSelf }
                rv.pipe(to: rp.box.seal)
            } catch {
                rp.box.seal(.rejected(error))
            }
        }
    case .rejected(let error):
        rp.box.seal(.rejected(error))
    }
}

```

在这个pipe的闭包里面，首先会检查Promise对象传递过来的Result\<T>对象，如果Result包含的是错误，则提取这个错误并封装到rp.box中，否则Result\<T>对象包含的是成功执行的结果。do里面的body(value)就是在执行提供给then的closure。

重点是在对`rv.pipe(to: rp.box.seal)`的理解上，`rv.pipe`的closure参数，会在rv持有的box被密封的时候调用，而调用时，传递给`rp.box.seal`的参数就是`rv`被resolve时所封装的结果，即一个`Result<U.T>`对象，这就是可以一直wrapper串联各种操作的原因。



#### Promise框架总结

#####  一.创建一个Promise\<T>的步骤

![promise](https://raw.githubusercontent.com/CathyLy/imageForSource/master/2020/promise.png)

- 首先调用了Promise的init方法

  ```swift
  public init(resolver body: (Resolver<T>) throws -> Void) {
      box = EmptyBox()
      let resolver = Resolver(box)
      do {
          try body(resolver)
      } catch {
          resolver.reject(error)
      }
  }
  ```

  

- 其次在`init`方法中创建了一个`.pending`状态的空盒子，这个盒子有两个属性：一个是用于同步盒子状态的队列，另一个是处理Promise值的Handlers数组：

  ```
  class EmptyBox<T>: Box<T> {
      private var sealant = Sealant<T>.pending(Handlers.init())
      private let barrier = DispatchQueue(label: "org.promisekit.barrier", attributes: .concurrent)
  }
  ```

- 第三，用上一步创建的盒子，创建一个`Resolver`对象，`Resolver`是PromiseKit为我们处理事件提供的接口。

- 第四，调用传递给`init`方法的闭包，这个闭包里面的代码有可能是同步执行也有可能是异步执行，具体取决于具体的逻辑，如果这个闭包抛出异常，PromiseKit会自动调用`resolver.reject(error)`，否则则需要在编写的闭包里面，根据代码执行的结果进行`resolver.fulfill(value)`或者`resolver.reject(error)`。这也是PromiseKit里面唯一有可能包含闭包嵌套的地方。

- 第五，无论是`fulfill`还是`reject`，它们背后的操作都是把特定的值封装在`Resolver`内部的盒子里。

- 最后，在`seal`的方法里面，如果盒子已经`resolved`，则直接返回，如果盒子的状态是`.pending`则把盒子设置为`.resolved`状态并一一调用每一个处理Promise期望值的handler。

  ```
  override func seal(_ value: T) {
      var handlers: Handlers<T>!
      barrier.sync(flags: .barrier) {
          guard case .pending(let _handlers) = self.sealant else {
              return  // already fulfilled!
          }
          handlers = _handlers
          self.sealant = .resolved(value)
      }
      if let handlers = handlers {
          handlers.bodies.forEach{ $0(value) }
      }
  }
  
  ```

  

##### 二.Promise的Wrapper操作

![image-20200121211615087](https://raw.githubusercontent.com/CathyLy/imageForSource/master/2020/promise-wrapper.png)

上面的图重点在于`then`方法实现里面

- 首先直接创建了它的返回值，一个`Promise<U.T>`对象，这个`Promise`里面包含的是一个等待密封的空盒子

- 创建好盒子之后，调用了`Promise.pipe`方法

  ```
  public func pipe(to: @escaping(Result<T>) -> Void) {
      switch box.inspect() {
      case .pending:
          box.inspect {
              switch $0 {
              case .pending(let handlers):
                  handlers.append(to)
              case .resolved(let value):
                  to(value)
              }
          }
      case .resolved(let value):
          to(value)
      }
  }
  
  ```

  这个方法里面有5个关键点，前两个是两次调用了`box.inspect`方法，这两次调用都会向上游的`Promise.box`自带的队列里添加两个任务。

  后三个是对`to`的引用，第一处是如果Promise是`.pending`则`pipe`的参数（闭包）添加到上游的`Promise.box`的`Handlers`的关联值中，后两处是如果Promise已经被`.resolved`则直接调用`pipe`的闭包参数

- then的`pipe`回调中，有4处关键点：第一，`try body(value)`这是是真正调用外部给`then`传递的闭包的地方；

  ```
  pipe {
    switch $0 {
    case .fulfilled(let value):
      on.async(flags: flags) {
        do {
          let rv = try body(value)
          guard rv !== rp else { throw PMKError.returnedSelf }
          rv.pipe(to: rp.box.seal)
        } catch {
          rp.box.seal(.rejected(error))
        }
      }
    case .rejected(let error):
      rp.box.seal(.rejected(error))
    }
  }
  ```

  后三处分别以下三行代码：

  ```
  rv.pipe(to: rp.box.seal)
  rp.box.seal(.rejected(error))
  rp.box.seal(.rejected(error))
  ```

  `rv`是`body(value)`的返回值，`rp`是`then`要返回的`Promise`对象，后两行代码主要功能是无论上游`Promise`发生了什么错误，还是说外部提供给`then`的闭包发生了异常，都要给`rp`的盒子密封上错误的value。

  如果没有发生任何异常，`rp`中的值则依赖`body`的实现：如果`body`中调用`fulfill`就封装成功的值，调用`reject`就封装错误的值。

  最后，把`rp`的封箱过程，交由`rv`来处理，等`rv`持有的盒子被密封了，就可以返回了。

  


#### when

```swift
let getUserAssets = getUserAssets(session: session)
let getUserMarginAssetsDetails = getMarginAccountDetails(session: session)
let getUserMarginCoeff = getUserMarginCoeff(session: session)
let getAccountProfit =getAccountProfit(session: session, period: time)

when(fulfilled: getUserAssets, getUserMarginAssetsDetails, getUserMarginCoeff, getAccountProfit)
.map({ (userAssets, userMarginAssetsDetails, userMarginCoeff, profits) -> MarginFundsState in
    return MarginFundsState(userAssets: userAssets, userMarginAssetsDetails: userMarginAssetsDetails, userMarginCoeff: userMarginCoeff, profits: profits)
})
```

`when`表示多个异步操作同时处理，当`when`中的操作都完成时再返回Promise结果。

```swift
public func when<U: Thenable>(fulfilled thenables: [U]) -> Promise<[U.T]> 
public func when<T>(resolved promises: Promise<T>...) -> Guarantee<[Result<T>]>
public func race<U: Thenable>(_ thenables: U...) -> Promise<U.T>
```

- `when(fulfilled:)`在所有异步操作都执行完成之后再回调，
- `when(resolved: )`是使一个或多个Promise失败也会等待，此`when`变体返回的值是一个`[Result<T>]`的数组，所有要保证是相同的泛型。
- `race()`表示的是只有有一个异步操作完成则立刻调用`then`的回调，其他没有执行完毕的异步操作会继续执行，不会停止。

```swift
/// Wait for all promises in a set to fulfill.
public func when<U: Thenable, V: Thenable, W: Thenable, X: Thenable, Y: Thenable>(
    fulfilled pu: U, _ pv: V, _ pw: W, _ px: X, _ py: Y) -> Promise<(U.T, V.T, W.T, X.T, Y.T)> {
    return _when([pu.asVoid(), pv.asVoid(), pw.asVoid(), px.asVoid(), py.asVoid()])
        .map(on: nil) { (pu.value!, pv.value!, pw.value!, px.value!, py.value!) }
}
```

从上面的when的函数可以看出，`when(fulfilled:)`PromiseKit目前最多支持同时五个异步不同Promise操作。

以上就是整个PromiseKit的源码分析。
