---
title: Moya && Alamofire
tags: liu
category: swift
abbrlink: 44783
date: 2020-02-26 00:05:39
---

对Moya和Alamofire的分析基于以下版本：Alamofire (4.9.1) 、Moya (13.0.1)。

## Moya

Moya的目录结构如下：

```
├── Moya
│   ├── AnyEncodable.swift
│   ├── Cancellable.swift
│   ├── Endpoint.swift
│   ├── Image.swift
│   ├── Moya+Alamofire.swift
│   ├── MoyaError.swift
│   ├── MoyaProvider+Defaults.swift
│   ├── MoyaProvider+Internal.swift
│   ├── MoyaProvider.swift
│   ├── MultiTarget.swift
│   ├── MultipartFormData.swift
│   ├── Plugin.swift
│   ├── Plugins
│   │   ├── AccessTokenPlugin.swift
│   │   ├── CredentialsPlugin.swift
│   │   ├── NetworkActivityPlugin.swift
│   │   └── NetworkLoggerPlugin.swift
│   ├── Response.swift
│   ├── TargetType.swift
│   ├── Task.swift
│   ├── URL+Moya.swift
│   ├── URLRequest+Encoding.swift
│   └── ValidationType.swift
├── ReactiveMoya
│   ├── MoyaProvider+Reactive.swift
│   └── SignalProducer+Response.swift
└── RxMoya
    ├── MoyaProvider+Rx.swift
    ├── Observable+Response.swift
    └── Single+Response.swift
```



### Moya介绍

Moya是基于Alamofire的更高一层的网络请求封装抽象层，在Moya中使用Enum来管理API，每一个网络请求接口都是枚举的一种case，下面是Moya的大概流程图：

![](https://raw.githubusercontent.com/liututu1213/imageForSource/master/2020/moya.png)

### Request

request部分主要是包含请求相关的代码，定义了TargetType协议，通过TargetType来定义一个请求。

- TargetType:请求相关信息
- Endpoint：Moya进行请求的最终数据结构
- Task：定义了请求的发起方式，请求参数设置等
- Cancellable：请求取消相关协议



### MoyaProvider

#### Provider初始化

MoyaProvider是Moya的核心处理类，Moya中所有的API请求都是通过Provider来发起的。Provider真正做的事情可以用一个流来表示：Target->Endpoint->Request,这个过程中Provider会一步步生层一个NSURLRequest来交给Alamofire发起请求。

```swift
public init(endpointClosure: @escaping EndpointClosure = MoyaProvider.defaultEndpointMapping,
            requestClosure: @escaping RequestClosure = MoyaProvider.defaultRequestMapping,
            stubClosure: @escaping StubClosure = MoyaProvider.neverStub,
            callbackQueue: DispatchQueue? = nil,
            manager: Manager = MoyaProvider<Target>.defaultAlamofireManager(),
            plugins: [PluginType] = [],
            trackInflights: Bool = false) {}

```

- endpointClosure：负责从TargetType转成Endpoint，Moya对endpointClosure提供了默认的实现。

  ```swift
  final class func defaultEndpointMapping(for target: Target) -> Endpoint {
      return Endpoint(
          url: URL(target: target).absoluteString,
          sampleResponseClosure: { .networkResponse(200, target.sampleData) },
          method: target.method,
          task: target.task,
          httpHeaderFields: target.headers
      )
  }
  ```

  在实际的项目中，可以自定义这个闭包来实现复杂的功能。

  

- requestClosure：负责把Endpoint转换成NSURLRequest，同样也有默认的实现`MoyaProvider.defaultRequestMappin`

  ```swift
  final class func defaultRequestMapping(for endpoint: Endpoint, closure: RequestResultClosure) {
      do {
          let urlRequest = try endpoint.urlRequest()
          closure(.success(urlRequest))
      } catch MoyaError.requestMapping(let url) {
          closure(.failure(MoyaError.requestMapping(url)))
      } catch MoyaError.parameterEncoding(let error) {
          closure(.failure(MoyaError.parameterEncoding(error)))
      } catch {
          closure(.failure(MoyaError.underlying(error, nil)))
      }
  }
  ```

  

- stubClosure：返回一个枚举StubBehavior，默认会返回never，如果需要假数据测试可以重写这个。

  ```swift
  public enum StubBehavior {
      case never
      case immediate
      case delayed(seconds: TimeInterval)
  }
  ```

  

- callbackQueue： 回调的线程，默认在主线程。

- manager：真正用来请求的类，由Alamofire桥接过来的，Moya提供了一个默认的Manager实现。

  ```swift
  final class func defaultAlamofireManager() -> Manager {
      let configuration = URLSessionConfiguration.default
      configuration.httpAdditionalHeaders = Manager.defaultHTTPHeaders
      
      let manager = Manager(configuration: configuration)
      manager.startRequestsImmediately = false
      return manager
  }
  ```

  

- plugins：插件，能够在不修改源码的情况下，来扩展其他功能，而且能够保证Provider的职责单一，满足开闭原则。
- trackInflights：是否要追踪请求，默认是false。



#### Provider 发起请求实现

![](https://raw.githubusercontent.com/liututu1213/imageForSource/master/2020/Provider.png)

在初始化Provider实例对象之后，对外暴露`request`方法以便发起请求：

```swift
@discardableResult
open func request(_ target: Target,
                  callbackQueue: DispatchQueue? = .none,
                  progress: ProgressBlock? = .none,
                  completion: @escaping Completion) -> Cancellable {
    
    let callbackQueue = callbackQueue ?? self.callbackQueue
    return requestNormal(target, callbackQueue: callbackQueue, progress: progress, completion: completion)
}
```

在swift里面，如果没有接收某方法的返回值，Xcode会报出警告，使用`@discardableResult`关键字在不想接收方法返回值的时候，用来消除Xcode的警告。

在MoyaProvider+Internal.swift文件中主要定义了十个方法,主要做的事如下图：

![](https://raw.githubusercontent.com/liututu1213/imageForSource/master/2020/provider-rquest.png)



### Moya+Alamofire

Moya+Alamofire.swift文件对Alamofire进行了简单的封装，这一层包装隔离了Alamofire的代码，如果需要更换请求框架的话，可以进行简单的修改来替换。

- 使用**typealias**对Alamofire相关的的API进行封装：Manager、Request、Method等

- 让Alamofire相关的类实现Moya定义的一些protocol

  ```swift
  extension Request: RequestType { }
  extension DataRequest: Requestable {
     ...
  }
  extension DownloadRequest: Requestable {
     ...
  }
  ```



### Plugin

Moya提供了非常好用的插件机制，可以通过插件在不修改源码的基础上，充分扩展Moya的功能。

```swift
public protocol PluginType {
    /// Called to modify a request before sending.
    func prepare(_ request: URLRequest, target: TargetType) -> URLRequest
    /// Called immediately before a request is sent over the network (or stubbed).
    func willSend(_ request: RequestType, target: TargetType)
    /// Called after a response has been received, but before the MoyaProvider has invoked its completion handler.
    func didReceive(_ result: Result<Moya.Response, MoyaError>, target: TargetType)
    /// Called to modify a result before completion.
    func process(_ result: Result<Moya.Response, MoyaError>, target: TargetType) -> Result<Moya.Response, MoyaError>
}

public extension PluginType {
    func prepare(_ request: URLRequest, target: TargetType) -> URLRequest { return request }
    func willSend(_ request: RequestType, target: TargetType) { }
    func didReceive(_ result: Result<Moya.Response, MoyaError>, target: TargetType) { }
    func process(_ result: Result<Moya.Response, MoyaError>, target: TargetType) -> Result<Moya.Response, MoyaError> { return result }
}
```

调用方可以实现上面的协议，来创建各种插件处理不同事情，从而保证Provider的单一职责。

- prepare：这个方法会在endpoint生成Request之后且请求发送前调用

  ```swift
  let performNetworking = { (requestResult: Result<URLRequest, MoyaError>) in
      // Allow plugins to modify request
      let preparedRequest = self.plugins.reduce(request) { $1.prepare($0, target: target) }
  }
  ```

- willSend：会在发送Alamofire请求前调用

  ```swift
  func sendAlamofireRequest<T>(_ alamoRequest: T, target: Target, callbackQueue: DispatchQueue?, progress progressCompletion: Moya.ProgressBlock?, completion: @escaping Moya.Completion) -> CancellableToken where T: Requestable, T: Request {
      // Give plugins the chance to alter the outgoing request
      let plugins = self.plugins
      plugins.forEach { $0.willSend(alamoRequest, target: target) }
  }
  ```

  

- didReceive：在收到Response的时候调用

  ```swift
  let completionHandler: RequestableCompletion = { response, request, data, error in
      let result = convertResponseToResult(response, request: request, data: data, error: error)
      // Inform all plugins about the response
      plugins.forEach { $0.didReceive(result, target: target) }
      completion(result)
  }
  ```

- process：在请求完成后修改Result。



### Response

响应结果相关方法，以及字典，JSON等相关转换方法。

```swift
/// Represents a response to a `MoyaProvider.request`.
public final class Response: CustomDebugStringConvertible, Equatable {
    public let statusCode: Int
    public let data: Data
    public let request: URLRequest?
    public let response: HTTPURLResponse?
}
```



#### Cancel

网络API请求是 可以被取消的，Moya返回协议Cancellable给客户端。

```swift
internal class CancellableWrapper: Cancellable {
    internal var innerCancellable: Cancellable = SimpleCancellable()

    var isCancelled: Bool { return innerCancellable.isCancelled }

    internal func cancel() {
        innerCancellable.cancel()
    }
}
```

在内部实现中，引入了一个CancellableWrapper来进行实际的Cancel包装，返回的实际实现协议的类型就是它。

使用一个CancellableWrapper包装的原因主要是对于没有实际发出的请求(参数错误)，cancel动作直接用SimpleCancellable即可，如果已经发出的请求，cancel则需要取消实际的网络请求。

在`CancellableToken`中，取消网络请求：

```swift
public final class CancellableToken: Cancellable, CustomDebugStringConvertible {
    fileprivate var lock: DispatchSemaphore = DispatchSemaphore(value: 1)
    
    public func cancel() {
        _ = lock.wait(timeout: DispatchTime.distantFuture)
        defer { lock.signal() }
        guard !isCancelled else { return }
        isCancelled = true
        cancelAction()
    }
    init(request: Request) {
        self.request = request
        self.cancelAction = {
            request.cancel()
        }
    }
}
```

这里用到了信号量，为了防止两个线程同时执行cancel操作。



### Moya的设计原则

1.面向协议

swift是一个面向协议的语言，协议建立的是一个抽象的依赖关系，同时Swift协议支持扩展，可以通过扩展为协议中的方法提供默认的实现。

```
public protocol TargetType {} //表示一个API请求
public protocol Cancellable {} // 唯一确定请求，只有一个接口用力爱取消
public protocol PluginType {} // 插件类型
public protocol RequestType {} // 对外提供的请求类型，隐藏Alamofire的细节
```



2.不可变状态

不可变状态是函数式编程的一个核心概念，在Moya中，很多状态都是不可变的。

```
public protocol TargetType {
    var baseURL: URL { get } // 只读
    ...
}
```

3.高阶函数

在swift中，函数是一等公民，可以作为函数的参数和返回值。高阶函数配合函数默认值，是swift开发中进行接口暴露的常用技巧。

4.插件

通过各个节点暴露出插件的接口，让Moya的日志、授权等功能无须耦合到核心代码里，让代码足够灵活性。

5.类型安全

使用枚举来保证类型安全是swift中常用的技巧。

6.错误处理

通过Result类型来处理异步错误。

```swift
/// A public function responsible for converting the result of a `URLRequest` to a Result<Moya.Response, MoyaError>.
public func convertResponseToResult(_ response: HTTPURLResponse?, request: URLRequest?, data: Data?, error: Swift.Error?) ->
    Result<Moya.Response, MoyaError> {
        switch (response, data, error) {
        case let (.some(response), data, .none):
            let response = Moya.Response(statusCode: response.statusCode, data: data ?? Data(), request: request, response: response)
            return .success(response)
        case let (.some(response), _, .some(error)):
            let response = Moya.Response(statusCode: response.statusCode, data: data ?? Data(), request: request, response: response)
            let error = MoyaError.underlying(error, response)
            return .failure(error)
        case let (_, _, .some(error)):
            let error = MoyaError.underlying(error, nil)
            return .failure(error)
        default:
            let error = MoyaError.underlying(NSError(domain: NSURLErrorDomain, code: NSURLErrorUnknown, userInfo: nil), nil)
            return .failure(error)
        }
}

```

用Result类型最大的好处是可以不用每一步都处理错误，通过Result类型，可以在最后统一处理错误。

```
provider.request(...).filter().mapJSON.filter().{ result in
    switch result {
        case let .success(moyaResponse):
    
        case let .failure(error):
    }
 }
```





## Alamofire

Alamofire的目录结构：

```
├── AFError.swift
├── Alamofire.swift
├── DispatchQueue+Alamofire.swift
├── MultipartFormData.swift
├── NetworkReachabilityManager.swift
├── Notifications.swift
├── ParameterEncoding.swift
├── Request.swift
├── Response.swift
├── ResponseSerialization.swift
├── Result.swift
├── ServerTrustPolicy.swift
├── SessionDelegate.swift
├── SessionManager.swift
├── TaskDelegate.swift
├── Timeline.swift
└── Validation.swift
```

![](https://raw.githubusercontent.com/liututu1213/imageForSource/master/2020/Alamofire.png)

### Alamofire.swift

该文件主要是给用户提供一些便利的调用方法`request`、`download`、`upload`、`stream`，以及定义了两个protocol接口，提供将其他类型转换成URL和URLRequest。

```swift
@discardableResult
public func request(
    _ url: URLConvertible,
    method: HTTPMethod = .get,
    parameters: Parameters? = nil,
    encoding: ParameterEncoding = URLEncoding.default,
    headers: HTTPHeaders? = nil)
    -> DataRequest
{
    return SessionManager.default.request(
        url,
        method: method,
        parameters: parameters,
        encoding: encoding,
        headers: headers
    )
}

@discardableResult
public func request(_ urlRequest: URLRequestConvertible) -> DataRequest {
    return SessionManager.default.request(urlRequest)
}
//....
```

上面是提供给外部发起网络请求的api。

```swift
public protocol URLConvertible {
    func asURL() throws -> URL
}

public protocol URLRequestConvertible {
    func asURLRequest() throws -> URLRequest
}
```

这个两个protocol主要是负责类型转换的接口。

### SessionManager

#### 1.SessionManager的初始化

上面的便利方法主要是调用Manager类的单利，Manager类中主要负责Session和Request的初始化，并提供SessionDelegate代理方法的默认实现。在实现代理方法时提供相应的闭包以提供给用户使用该闭包来回调相应的代理方法。

```swift
public static let `default`: SessionManager = {
    let configuration = URLSessionConfiguration.default
    configuration.httpAdditionalHeaders = SessionManager.defaultHTTPHeaders

    return SessionManager(configuration: configuration)
}()
```

Manager类是以单利的形式对外使用的，创建SessionManager对象时指定了一个默认的`URLSessionConfiguration`和`defaultHTTPHeaders`。

```swift
public init(
    configuration: URLSessionConfiguration = URLSessionConfiguration.default,
    delegate: SessionDelegate = SessionDelegate(),
    serverTrustPolicyManager: ServerTrustPolicyManager? = nil) {
    self.delegate = delegate
    self.session = URLSession(configuration: configuration, delegate: delegate, delegateQueue: nil)

    commonInit(serverTrustPolicyManager: serverTrustPolicyManager)
}
private func commonInit(serverTrustPolicyManager: ServerTrustPolicyManager?) {
    session.serverTrustPolicyManager = serverTrustPolicyManager

    delegate.sessionManager = self

    delegate.sessionDidFinishEventsForBackgroundURLSession = { [weak self] session in
        guard let strongSelf = self else { return }
        DispatchQueue.main.async { strongSelf.backgroundCompletionHandler?() }
    }
}
```

SessionManager的`init`方法中，初始化了SessionDelegate和URLSession，并并SessionDelegate对象作为URLSession的delegate。

#### 2.Manager的request方法

```swift
@discardableResult
open func request(
    _ url: URLConvertible,
    method: HTTPMethod = .get,
    parameters: Parameters? = nil,
    encoding: ParameterEncoding = URLEncoding.default,
    headers: HTTPHeaders? = nil)
    -> DataRequest {
    var originalRequest: URLRequest?

    do {
        originalRequest = try URLRequest(url: url, method: method, headers: headers)
        /// 对参数进行编码, 其中encoding方法会根据ParameterEncoding的类型不同返回不同的request
        let encodedURLRequest = try encoding.encode(originalRequest!, with: parameters)
        return request(encodedURLRequest)
    } catch {
        return request(originalRequest, failedWith: error)
    }
}
```

`url`是一个实现了`URLConvertible`的对象，`method`参数表示请求方式（GET、POST、PUT等），`parameters`就是请求参数，`encoding:ParameterEncoding`是请求参数的编码方式，在Alamofire的源码里面有URLEncoding、JSONEncoding、PropertyListEncoding，这里默认的是URL 编码。

```swift
@discardableResult
open func request(_ urlRequest: URLRequestConvertible) -> DataRequest {
    var originalRequest: URLRequest?
    do {
        originalRequest = try urlRequest.asURLRequest()
        let originalTask = DataRequest.Requestable(urlRequest: originalRequest!)//return TaskConvertible

        let task = try originalTask.task(session: session, adapter: adapter, queue: queue)
        let request = DataRequest(session: session, requestTask: .data(originalTask, task))
        delegate[task] = request
        if startRequestsImmediately { request.resume() }
        return request
    } catch {
        return request(originalRequest, failedWith: error)
    }
}
```

上面的`request`的方法主要是负责通过Session创建DataTask，并用task对象初始化DataRequest类的对象，并将request存入ManagerDelegate类的属性中`delegate[task] = request`，因为在delegate属性中的代理方法是调用相应的TaskDelegate类的方法。

然后默认调用`request.resume`发起网络请求。



### SessionDelegate

这个类主要是URLSessionDelegate以及相关子协议的代理类，其中给出了各个代理方法的默认实现，并将URLSessionDelegate的相关代理封装成闭包的形式提供给外部。如果闭包变量不为空则会执行闭包块的内容，而不会执行默认的实现。

```swift
// MARK: URLSessionDelegate Overrides
open var sessionDidReceiveChallenge: ((URLSession, URLAuthenticationChallenge) -> (URLSession.AuthChallengeDisposition, URLCredential?))?
open var sessionDidReceiveChallengeWithCompletion: ((URLSession, URLAuthenticationChallenge, @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) -> Void)?

// MARK: URLSessionDataDelegate Overrides
open var dataTaskDidReceiveResponse: ((URLSession, URLSessionDataTask, URLResponse) -> URLSession.ResponseDisposition)?

// MARK: URLSessionDownloadDelegate Overrides
// ....

var retrier: RequestRetrier?
weak var sessionManager: SessionManager?

var requests: [Int: Request] = [:]
private let lock = NSLock()
/// Access the task delegate for the specified task in a thread-safe manner.
open subscript(task: URLSessionTask) -> Request? {
    get {
        lock.lock() ; defer { lock.unlock() }
        return requests[task.taskIdentifier]
    }
    set {
        lock.lock() ; defer { lock.unlock() }
        requests[task.taskIdentifier] = newValue
    }
}
extension SessionDelegate: URLSessionDelegate {}
extension SessionDelegate: URLSessionTaskDelegate {}
extension SessionDelegate: URLSessionDataDelegate {}
extension SessionDelegate: URLSessionDownloadDelegate {}
extension SessionDelegate: URLSessionStreamDelegate {}
```

- `subscript`表示自定义下标，可以通过`delegate[task]`的形式进行访问；

- 使用到NSLock来保证线程安全以便在访问指定的task时；

- `requests`负责储存request对象，以便SessionDelegate在URLSessionDelegate相应的代理方法中通过`self[task]?.delegate`，获取到TaskDelegate对象来调用其方法，例如以下的代理方法中的实现：

  ```swift
  open func urlSession(
      _ session: URLSession,
      task: URLSessionTask,
      didReceive challenge: URLAuthenticationChallenge,
      completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void)
  {
      guard taskDidReceiveChallengeWithCompletion == nil else {
          taskDidReceiveChallengeWithCompletion?(session, task, challenge, completionHandler)
          return
      }
  
      if let taskDidReceiveChallenge = taskDidReceiveChallenge {
          let result = taskDidReceiveChallenge(session, task, challenge)
          completionHandler(result.0, result.1)
      } else if let delegate = self[task]?.delegate {
          delegate.urlSession(
              session,
              task: task,
              didReceive: challenge,
              completionHandler: completionHandler
          )
      } else {
          urlSession(session, didReceive: challenge, completionHandler: completionHandler)
      }
  }
  ```

### Request相关类

#### 1.Request初始化

Request就是负责通过会话创建各种类型的Task，并给出相应的Task Delegate。

```swift
init(session: URLSession, requestTask: RequestTask, error: Error? = nil) {
    self.session = session

    switch requestTask {
    case .data(let originalTask, let task):
        taskDelegate = DataTaskDelegate(task: task)
        self.originalTask = originalTask
    case .download(let originalTask, let task):
        taskDelegate = DownloadTaskDelegate(task: task)
        self.originalTask = originalTask
    case .upload(let originalTask, let task):
        taskDelegate = UploadTaskDelegate(task: task)
        self.originalTask = originalTask
    case .stream(let originalTask, let task):
        taskDelegate = TaskDelegate(task: task)
        self.originalTask = originalTask
    }

    delegate.error = error
    delegate.queue.addOperation { self.endTime = CFAbsoluteTimeGetCurrent() }
}
```

初始化方法需要两个参数：第一个参数是URLSession对象，改对象时SessionManager单例持有的session，第二个参数同样是Manager中传过来的`RequestTask`枚举类型，根据RequestTask的类型来设置代理对象DataDelegate，例如如果是data类型则创建的是DataDelegate的实例。



#### Request的ProgressHandler闭包

在使用Request类的对象时，可以链式地调用Request中的方法，最常用的是获取相应任务执行的进度：

```swift
Alamofire.request(URL(string: "https://httpbin.org/get")!).downloadProgress { (progress) in
    /// progress
    
}.response { (response) in
    /// response
}
```

其中DataRequest和DownloadRequest定义的是downloadProgress，UploadRequest定义的是uploadProgress闭包，可以通过这个闭包监听下载或上传的进度。

#### resume方法

```swift
open func resume() {
    guard let task = task else { delegate.queue.isSuspended = false ; return }
    
    if startTime == nil { startTime = CFAbsoluteTimeGetCurrent() }
    
    task.resume()
    
    NotificationCenter.default.post(
        name: Notification.Name.Task.DidResume,
        object: self,
        userInfo: [Notification.Key.Task: task]
    )
}

```

`resume`方法主要是用来记录startTime，然后调用task的resume方法开始执行任务，并发出相应的通知。



### ResponseSerialization

 ResponseSerialization是用来对Response返回的值进行序列化。在文件中定义了两个protocol：`DataResponseSerializerProtocol`和`DownloadResponseSerializerProtocol`

```swift
public protocol DataResponseSerializerProtocol {
    /// The type of serialized object to be created by this `DataResponseSerializerType`.
    associatedtype SerializedObject
    var serializeResponse: (URLRequest?, HTTPURLResponse?, Data?, Error?) -> Result<SerializedObject> { get }
}

public protocol DownloadResponseSerializerProtocol {
    associatedtype SerializedObject
    var serializeResponse: (URLRequest?, HTTPURLResponse?, URL?, Error?) -> Result<SerializedObject> { get }
}
```

这两个protocol分别由`DataResponseSerializer<Value>`和`DownloadResponseSerializer<Value>`来实现。

在Alamofire中调用DataRequest的的`response`或者`responseJSON`等其他方法，就会在request结束时，添加一个操作`OperationQueue`来处理服务器的response。

```swift
@discardableResult
public func response<T: DataResponseSerializerProtocol>(
    queue: DispatchQueue? = nil,
    responseSerializer: T,
    completionHandler: @escaping (DataResponse<T.SerializedObject>) -> Void)
    -> Self
{
    delegate.queue.addOperation {
        let result = responseSerializer.serializeResponse(
            self.request,
            self.response,
            self.delegate.data,
            self.delegate.error
        )

        var dataResponse = DataResponse<T.SerializedObject>(
            request: self.request,
            response: self.response,
            data: self.delegate.data,
            result: result,
            timeline: self.timeline
        )

        dataResponse.add(self.delegate.metrics)

        (queue ?? DispatchQueue.main).async { completionHandler(dataResponse) }
    }

    return self
}
```



总结：

在Alamofire框架中使用了协议、扩展、闭包以及枚举关联值，在解析网络数据的时，将闭包类型作为函数的参数，通过闭包变量来提供相应的解析方案。

- Alamofire对泛型的使用：

  ```
  public enum Result<Value> {}
  public func validate<S: Sequence>(contentType acceptableContentTypes: S) -> Self where S.Iterator.Element == String {}
  public struct DataResponse<Value> {}
  ```

- 链式调用

  对于Request类。其中很多方法都使用了返回`->Self`方法

  ```
  /// 鉴权
  open func authenticate(usingCredential credential: URLCredential) -> Self {}
  /// 进度
  open func downloadProgress(queue: DispatchQueue = DispatchQueue.main, closure: @escaping ProgressHandler) -> Self {}
  ///响应
  public func response(queue: DispatchQueue? = nil, completionHandler: @escaping (DefaultDataResponse) -> Void) -> Self {}
  ```

- 顺序创建Task，并保存所有的requests。

  ```
  delegate[task] = request
  open subscript(task: URLSessionTask) -> Request? {
      get {
          lock.lock() ; defer { lock.unlock() }
          return requests[task.taskIdentifier]
      }
      set {
          lock.lock() ; defer { lock.unlock() }
          requests[task.taskIdentifier] = newValue
      }
  }
  ```

- 在URLSession的回调中根据task把回调分发给对应的TaskDelegate











