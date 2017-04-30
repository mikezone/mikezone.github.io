---
layout: cnblog_post
title:  "通读AFN①--从创建manager到数据解析完毕"
date:   2016-01-28 13:50:39
categories: iOS
---
### 流程梳理
今天开始会写几篇关于AFN源码解读的一些Blog，首先要梳理一下AFN的整体结构(主要是讨论2.x版本的Session访问模块)：
我们先看看我们最常用的一段代码：

```objectivec
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
[manager GET:@"https://www.baidu.com" parameters:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
    // ... successHandler
} failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
    // ... failureHandler
}];
```
在前面关于 <a href="http://www.cnblogs.com/Mike-zh/p/5152073.html" target='blank'>AFN URLEncode</a> 的文章说道，AFN将网络访问分为三个过程化的模块，下面我把第一部分再分为两个步骤：

1.访问前的准备：使用AFURLRequestSerialization类创建一个新的URLRequest对象(用于即将进行的网络访问)，对传递过来的URLrequest对象进行三步加工：

①配置默认网络配置，如(allowsCellularAccess,cachePolicy,HTTPShouldHandleCookies,HTTPShouldUsePipelining,networkServiceType,timeoutInterval)

②将request的HTTPHeader赋给新的request

③将parameter字典转为queryString，拼接在URLRequest的URL后面.如果是POST，PUT，PATCH方法，则放在HTTPBody中，并设置`Content-Type`头为表单类型：`application/x-www-form-urlencoded`

2.用1中所得的mutableRequest对象创建dataTask

3.访问过程中，将代理职责下放给AFURLSessionManagerTaskDelegate，通过代理方法接收数据。

4.完全接受到数据或失败之后的处理：失败回调、成功后解析然后回调。

上面四个步骤都是在`[manager GET: parameters: success: failure:]`这个方法中完成的，而在进行网络访问之前的`[AFHTTPSessionManager manager]`是对网络访问过程组件的初始化，也就是，在`AFHTTPSessionManager`的`+manager`方法中，完成了对自己和`requestSerializer`以及`responseSerializer`的初始化工作，`+manager`方法内部的代码:

```objectivec
self.baseURL = url;

self.requestSerializer = [AFHTTPRequestSerializer serializer];
self.responseSerializer = [AFJSONResponseSerializer serializer];
```
可以看出requestSerializer和responseSerializer对象都是按照默认的构造方法`serializer`创建的，同时可以看出responseSerializer默认使用了JSON的解析方式，着也是为什么当使用AFN进行网络请求时，JSON会自动进行解析的原因。看到这里我们也了解了如果想进行修改默认的request和response序列化方式修改，在何时添加这部分代码。就是在manager的默认设置完成之后，在开始进行网络访问三步走之前：

```objectivec
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
manager.requestSerializer = [AFJSONRequestSerializer serializer];
manager.responseSerializer = [AFXMLParserResponseSerializer serializer];
[manager GET:@"https://www.baidu.com" parameters:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
    // ... successHandler
} failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
    // ... failureHandler
}];
```
我们能改变的不仅仅是request和reponse按照什么格式序列化，还可以改变默认的session配置，进行创建Task的session对象在AFN中成为了AFHTTPSessionManager的属性，如果不使用构造方法`- (instancetype)initWithBaseURL:(NSURL *)url sessionConfiguration:(NSURLSessionConfiguration *)configuration`     传给它一个值，它会在`AFHTTPSessionManager`的父类`AFURLRequestSerialization`中默认配置的，不光如此，而且还配置了`AFHTTPSessionManager`的很多重要属性，在`AFURLRequestSerialization`的`-initWithBaseURL: sessionConfiguration:`中：

```objectivec
if (!configuration) {
    configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
}

self.sessionConfiguration = configuration;

self.operationQueue = [[NSOperationQueue alloc] init];
self.operationQueue.maxConcurrentOperationCount = 1;

self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];

self.responseSerializer = [AFJSONResponseSerializer serializer];

self.securityPolicy = [AFSecurityPolicy defaultPolicy];

self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];

self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

self.lock = [[NSLock alloc] init];
self.lock.name = AFURLSessionManagerLockName;

[self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
    for (NSURLSessionDataTask *task in dataTasks) {
        [self addDelegateForDataTask:task completionHandler:nil];
    }

    for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
        [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
    }

    for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
        [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
    }
}];

[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidResume:) name:AFNSURLSessionTaskDidResumeNotification object:nil];
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(taskDidSuspend:) name:AFNSURLSessionTaskDidSuspendNotification object:nil];

return self;
```
这些默认的配置大多是不可以在外部修改，因为大都为readonly属性，只是在实现文件中给了修改的接口。例如：

```objectivec
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
// 设置最大并发操作数
manager.operationQueue.maxConcurrentOperationCount = 3; // error
[manager GET: parameter: success: failure:];
```
AFN网络访问的默认设置大多都是不希望用户修改的，对外提供的接口也仅仅局限于request和response的序列化方式的修改。
看完了这些，我们来着重看一下网络访问三步走的过程：

### 1.访问前的准备：使用AFURLRequestSerialization类创建一个新的URLRequest对象
这一部分的很多知识点，在这篇文章  <a href="http://www.cnblogs.com/Mike-zh/p/5152073.html" target='blank'>iOS. PercentEscape是错用的URLEncode，看看AFN和Facebook吧</a>中有介绍，这里说一些补充的内容：
首先是requestSerializer的创建细节：这个虽然不属于这部分内容(它是在+manager方法中就创建了)，但有些问题还需注意：

这个创建过程主要是设置默认编码为UTF8，对Accept-Language、User-Agent两个头的初始化，设置允许queryString放在URL中的HTTP请求方法为&#64;"GET", &#64;"HEAD", &#64;"DELETE"、添加对&#64;[&#64;"allowsCellularAccess", &#64;"cachePolicy", &#64;"HTTPShouldHandleCookies", &#64;"HTTPShouldUsePipelining", &#64;"networkServiceType", &#64;"timeoutInterval"]属性值(这些key通过一个静态数组获得)的观察者为本身。

需要注意的是请求头本来是Request的属性，这里设置请求头是用requestSerilizer对象的一个字典属性mutableHTTPRequestHeaders将它们先存储起来，以备在修改传递过来的request对象过程中使用。

为什么要KVO以上6个属性？

字典属性mutableObservedChangedKeyPaths用来存储这6个属性值中非空的值，如果这6个属性中的任何一个被赋了新值，就会在observeValueForKeyPath:中检查新值是否为空，如果为空，就从mutableObservedChangedKeyPaths中移出这个对象，表示不再需要考虑这个值对配置的影响。
而这些非空的值会在进行网络访问前创建新的mutableRequest对象的时候一一赋给它（这些属性本来就是URLRequest对象的属性）。

这个过程我们可以换一个思路实现，就是非空给属性赋值，空时赋给属性NSNull，在将这些属性赋给mutableRequest的时候判断是否为NSNull，如果是，就不赋值了。相比之下AFN的做法对扩展性更好一些。而这种方法的使用在AFN是非常常见的。

下面我们就看一下mutableRequest创建的细节吧：

```objectivec
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(method);
    NSParameterAssert(URLString);

    NSURL *url = [NSURL URLWithString:URLString];

    NSParameterAssert(url);

    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    mutableRequest.HTTPMethod = method;

    // 给mutableRequest赋值刚才在AFHTTPRequestSerializerObservedKeyPaths存储的属性，已经去掉了空值。
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }
    // 将HTTPRequestHeaders字典属性中的Header传给mutableRequest， 将格式化好的queryString传给mutableRequest
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}
```
刚才的大费口舌就是刚好是对这段代码的解释。
准备好了request，我们就来看一下如何使用request创建dataTask

### 2.使用准备好的mutableRequest对象创建dataTask
在`- (NSURLSessionDataTask *)dataTaskWithHTTPMethod: URLString: parameters: failure:`方法中的的后半段：

```objectivec
__block NSURLSessionDataTask *dataTask = nil;
dataTask = [self dataTaskWithRequest:request completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) { // 下面会解读这一句
    if (error) {
        if (failure) {
            failure(dataTask, error);
        }
    } else {
        if (success) {
            success(dataTask, responseObject);
        }
    }
}];

return dataTask;
```
其中的failure和success实际上是由我们使用者传递过来，这段非常简单的代码同样是有点机关的，这其中包含了AFN设计中使用的`将代理职责转移`的思想,尽管我们平常也使用过类似的代码，但还是研读一下AFN如何实现的吧：

上面的dataTask的创建的核心代码实现是这样的：

```objectivec
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                            completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    __block NSURLSessionDataTask *dataTask = nil;
    dispatch_sync(url_session_manager_creation_queue(), ^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    [self addDelegateForDataTask:dataTask completionHandler:completionHandler]; // 下面有解析

    return dataTask;
}
```
如上，AFN会选择在它自定义的串行队列`url_session_manager_creation_queue`(这个队列标记了label："com.alamofire.networking.session.manager.creation")中采用同步的方式创建dataTask。
在dataTask被创建之后将代理职责下方给了AFURLSessionManagerTaskDelegate对象，我们可以通过查看`[self addDelegateForDataTask:dataTask completionHandler:completionHandler];`得出：

```objectivec
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self; // AFURLSessionManagerTaskDelegate弱引用它的管理者(AFHTTPSessionManager对象)
    delegate.completionHandler = completionHandler; // 将完成的回调(failure和success的处理)传递给AFURLSessionManagerTaskDelegate

    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask];
}
```
而在的实现中：

```objectivec
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [self.lock unlock];
}
```
这里AFNHTTPSessionManager将单个dataTask的代理职责下放给了一个`AFURLSessionManagerTaskDelegate`对象，但是这个对象仍然受manager的控制，manager会用一个可变字典类型的属性mutableTaskDelegatesKeyedByTaskIdentifier存储它管理的所有的dataTask和这个dataTask对应的`AFURLSessionManagerTaskDelegate`对象的关系，而具体的任务下放就是通过这种关系来实现的。

下面就边介绍数据请求与接收的过程阶段边解释如何通过这种关系将代理职责下放。

### 3.网络访问过程中
这一过程是由dataTask的resume方法开始的。AFHTTPSessionManager的成员session会使用上面的request进行网络请求，当接收到数据之后进入回调，AFN已将session在AFHTTPSessionManager的父类AFURLSessionManager中默认设置了`self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];`并且session的代理方法也已在AFURLSessionManager类中实现。而AFN在这个实现的过程中将每次接收到的数据都交给了当前dataTask对应的`AFURLSessionManagerTaskDelegate`对象处理，在这里实现了职责下放：
在AFURLSessionManager.m中：

```objectivec
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask]; // 找到dataTask对应的AFURLSessionManagerTaskDelegate对象
    [delegate URLSession:session dataTask:dataTask didReceiveData:data]; // 代理职责下放

    if (self.dataTaskDidReceiveData) {
        self.dataTaskDidReceiveData(session, dataTask, data);
    }
}

// 如何找到dataTask对应的AFURLSessionManagerTaskDelegate对象
- (AFURLSessionManagerTaskDelegate *)delegateForTask:(NSURLSessionTask *)task {
    NSParameterAssert(task);

    AFURLSessionManagerTaskDelegate *delegate = nil;
    [self.lock lock];
    delegate = self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)]; // 根据taskId，之前以key为taskId、value为AFURLSessionManagerTaskDelegate对象的形式存入字典中。
    [self.lock unlock];

    return delegate;
}
```
<span style="color: #0000ff;font-size: 11pt; font-family:Comic Sans MS">而真正处理网络请求的类是`AFURLSessionManagerTaskDelegate`，它从未被设置为session的delegate，而是在AFHTTPSessionManager(AFURLSessionManager)对session的代理方法的实现中主动调用。</span>

这个数据最后被这样处理，在AFURLSessionManagerTaskDelegate中

```objectivec
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    [self.mutableData appendData:data];
}
```
我们可以看到AFURLSessionManagerTaskDelegate类有一个mutableData属性用来拼接接收的数据。
看一下接收完毕之后是如何处理的，先是在AFURLSessionManager中：

```objectivec
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];

    // delegate may be nil when completing a task in the background
    if (delegate) {
        [delegate URLSession:session task:task didCompleteWithError:error];

        [self removeDelegateForTask:task];
    }

    if (self.taskDidComplete) {
        self.taskDidComplete(session, task, error);
    }
}
```
这里先找到task对应的AFURLSessionManagerTaskDelegate对象，同样是通过dataTask的Id，然后将处理任务交给这个delegate对象，等它处理之后，sessionManager会将这个delegate对象从字典中移除：

```objectivec
- (void)removeDelegateForTask:(NSURLSessionTask *)task {
    NSParameterAssert(task);

    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];
    [self.lock lock];
    [delegate cleanUpProgressForTask:task];
    [self removeNotificationObserverForTask:task];
    [self.mutableTaskDelegatesKeyedByTaskIdentifier removeObjectForKey:@(task.taskIdentifier)];
    [self.lock unlock];
}
```
这样manager管理的session进行的一次dataTask就完毕了。

再看一下在AFURLSessionManagerTaskDelegate中，如何具体处理的

```objectivec
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
    __strong AFURLSessionManager *manager = self.manager;

    __block id responseObject = nil;

    __block NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
    userInfo[AFNetworkingTaskDidCompleteResponseSerializerKey] = manager.responseSerializer;

    //Performance Improvement from #2672
    NSData *data = nil;
    if (self.mutableData) {
        data = [self.mutableData copy];
        //We no longer need the reference, so nil it out to gain back some memory.
        self.mutableData = nil;
    }

    if (self.downloadFileURL) {
        userInfo[AFNetworkingTaskDidCompleteAssetPathKey] = self.downloadFileURL;
    } else if (data) {
        userInfo[AFNetworkingTaskDidCompleteResponseDataKey] = data;
    }

    if (error) {
        userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;

        dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
            if (self.completionHandler) {
                self.completionHandler(task.response, responseObject, error);
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
            });
        });
    } else {
        dispatch_async(url_session_manager_processing_queue(), ^{
            NSError *serializationError = nil;
            responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];

            if (self.downloadFileURL) {
                responseObject = self.downloadFileURL;
            }

            if (responseObject) {
                userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
            }

            if (serializationError) {
                userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
            }

            dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
                if (self.completionHandler) {
                    self.completionHandler(task.response, responseObject, serializationError);
                }

                dispatch_async(dispatch_get_main_queue(), ^{
                    [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
                });
            });
        });
    }
#pragma clang diagnostic pop
}
```
取出SessionManager，和sessionManager的responseSerializer属性，创建userInfo字典，存放数据解析的组件对象和返回的数据等

如果有错误：
1.userInfo存入error，key为完成错误的标记，

2.创建队列任务：在主队列中完成回调(由最开始传入的success和failure处理)、然后向主线程发送附带userInfo的任务完成的通知，

3.将2创建的任务放在静态的队列组url_session_manager_completion_group()中执行。

没有错误：
在异步的静态队列url_session_manager_processing_queue(label是"com.alamofire.networking.session.manager.processing")中处理：

1.用manager的responseSerializer属性进行数据解析，将data解析为responseObject
	1.1.解析正确，将responseObject存入userInfo中，
	1.2.解析失败，将错误信息serializationError存入userInfo，

2.创建队列任务：在主队列中完成回调(由最开始传入的success和failure处理)、然后向主线程发送附带userInfo的任务完成的通知，

3.将2创建的任务放在静态的队列组url_session_manager_completion_group()中执行。

要说明的一点是：AFN只负责发送通知，而没有对通知进行接收的处理，这部分需要使用者自己完成。
现在就只剩下数据解析的过程了还没有介绍了。

### 4.数据解析
这里主要体现的是面向对象`多态`的特性。
在无论我们使用AFHTTPSessionManager对象或是使用AFURLSessionManager对象创建的dataTask在数据解析阶段，都会调用上面刚刚分析完的代码中的`responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];`这一句进行数据解析，而在HTTPSessionManager的manager方法中默认为我们创建了JSON类型的解析器`self.responseSerializer = [AFJSONResponseSerializer serializer];`,这样在执行过程中，就会动态地调用AFJSONResponseSerializer的`-responseObjectForResponse: data: error:`方法，它的实现是这样的：

```objectivec
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
    }

    id responseObject = nil;
    NSError *serializationError = nil;
    // Workaround for behavior of Rails to return a single space for `head :ok` (a workaround for a bug in Safari), which is not interpreted as valid input by NSJSONSerialization.
    // See https://github.com/rails/rails/issues/1742
    BOOL isSpace = [data isEqualToData:[NSData dataWithBytes:" " length:1]];
    if (data.length > 0 && !isSpace) {
        responseObject = [NSJSONSerialization JSONObjectWithData:data options:self.readingOptions error:&serializationError];
    } else {
        return nil;
    }

    if (self.removesKeysWithNullValues && responseObject) {
        responseObject = AFJSONObjectByRemovingKeysWithNullValues(responseObject, self.readingOptions);
    }

    if (error) {
        *error = AFErrorWithUnderlyingError(serializationError, *error);
    }

    return responseObject;
}
```
这是一个非常简单的算法，

1.先调用了从父类(AFHTTPResponseSerializer)集成而来的数据验证方法，如果验证失败了，并且确认错误是由AFN解析引起的，返回nil，

2.检验data是否为空或者一个空格这样的无效数据，失败返回nil，否则将data解析为JSONObject

3.如果removesKeysWithNullValues属性设置为YES，那么要去掉2中的JSONObject中的value等于[NSNull null]的元素。

AFJSONResponseSerializer类是AFHTTPResponseSerializer的子类，一些初始化的设置，还有验证数据的方法都是在AFJSONResponseSerializer中完成的。

看一下AFJSONResponseSerializer类：

```objectivec
- (instancetype)init {
 	// ...
    self.stringEncoding = NSUTF8StringEncoding;

    self.acceptableStatusCodes = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(200, 100)]; // 只接受statusCode为2xx
    self.acceptableContentTypes = nil; // 接收的Content-Type,需要子类的init中重写

    return self;
}

- (BOOL)validateResponse:(NSHTTPURLResponse *)response
                    data:(NSData *)data
                   error:(NSError * __autoreleasing *)error
{
    BOOL responseIsValid = YES;
    NSError *validationError = nil;

    if (response && [response isKindOfClass:[NSHTTPURLResponse class]]) {
        if (self.acceptableContentTypes && ![self.acceptableContentTypes containsObject:[response MIMEType]]) {
            if ([data length] > 0 && [response URL]) {
                NSMutableDictionary *mutableUserInfo = [@{
                                                          NSLocalizedDescriptionKey: [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: unacceptable content-type: %@", @"AFNetworking", nil), [response MIMEType]],
                                                          NSURLErrorFailingURLErrorKey:[response URL],
                                                          AFNetworkingOperationFailingURLResponseErrorKey: response,
                                                        } mutableCopy];
                if (data) {
                    mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
                }

                validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorCannotDecodeContentData userInfo:mutableUserInfo], validationError);
            }

            responseIsValid = NO;
        }

        if (self.acceptableStatusCodes && ![self.acceptableStatusCodes containsIndex:(NSUInteger)response.statusCode] && [response URL]) {
            NSMutableDictionary *mutableUserInfo = [@{
                                               NSLocalizedDescriptionKey: [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: %@ (%ld)", @"AFNetworking", nil), [NSHTTPURLResponse localizedStringForStatusCode:response.statusCode], (long)response.statusCode],
                                               NSURLErrorFailingURLErrorKey:[response URL],
                                               AFNetworkingOperationFailingURLResponseErrorKey: response,
                                       } mutableCopy];

            if (data) {
                mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
            }

            validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorBadServerResponse userInfo:mutableUserInfo], validationError);

            responseIsValid = NO;
        }
    }

    if (error && !responseIsValid) {
        *error = validationError;
    }

    return responseIsValid;
}
```
init不再多说，主要是验证方法`- (BOOL)validateResponse: data: error:`，在这个方法内部完成了这些工作：

1.设置验证通过responseIsValid的默认值YES，错误validationError为nil

2.验证
 &emsp;<br/>2.1对response的MIME类型验证：如果acceptableContentTypes属性中不包含response的MIME类型，则认为验证失败，responseIsValid设为NO，本地化错误描述，并将描述、response的URL、response对象存入userInfo字典，用这个userInfo字典创建Domain为AFURLResponseSerializationErrorDomain的NSError对象
 &emsp;<br/>2.2对response.statusCode验证：如果acceptableStatusCodes属性中不包含response.statusCode，则认为失败，处理同2.1,

3.将错误赋给参数error，返回responseIsValid。
<br/>
对于其他类型的解析与JSON类似，这里列举一下经过解析后的的id responseObject对应的类型:

| manager的responseSerializer属性类型 | 解析后的responseObject类型 |
| ------ | ----- |
| AFHTTPResponseSerializer | NSData |
| AFJSONResponseSerializer | JSONObject(NSDictionary或NSArray) |
| AFXMLParserResponseSerializer | NSXMLParser |
| AFXMLDocumentResponseSerializer | NSXMLDocument |
| AFPropertyListResponseSerializer | propertyList(NSDictionary或NSArray) |
| AFImageResponseSerializer | iOS、TV、Watch:UIImage &emsp;&emsp; Mac:NSImage|
| AFCompoundResponseSerializer | 用responseSerializers数组中对象依次解析，<br/>第一个失败，则用第二个解析，依次类推,返回第一个成功的结果|

当获取responseObject对象后，直接按类型使用即可，例如如果设置了manager.responseSerializer = [AFXMLParserResponseSerializer serializer],就要这样解析：

```objectivec
success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
    NSXMLParser *saxParser = (NSXMLParser *)responseObject;
    saxParser.delegate = self;
    [saxParser parse];
}
```
不过若要使用相同的manager对象进行下一次网络访问,如果不知道response的Content-Type，就要将manager的responseSerializer复原，重新设置为：

```objectivec
manager.responseSerializer = [AFJSONRequestSerializer serializer]； // 如果manager为AFHTTPSessionManager
manager.responseSerializer = [AFHTTPRequestSerializer serializer]； // 如果manager为AFURLSessionManager
```