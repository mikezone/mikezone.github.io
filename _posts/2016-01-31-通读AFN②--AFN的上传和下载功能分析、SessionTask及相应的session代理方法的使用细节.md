---
layout: cnblog_post
title:  "通读AFN②--AFN的上传和下载功能分析、SessionTask及相应的session代理方法的使用细节"
date:   2016-01-31 13:50:39
categories: iOS
---
这一部分主要研究AFN的上传和下载功能，中间涉及到各种NSURLSessionTask的一些创建的解析和HTTPSessionManager对RESTful风格的web应用的支持，同时会穿插一点NSURLSession代理方法被调用的时机和对上传的数据的序列化的步骤。
本文主要讲解的是上传和下载的代码实现细节，不会考虑上传过程中的安全性问题。

文件的上传和下载同时也包括普通的数据请求说说到底都是使用了系统的NSURLSession类创建对应的Task，然后执行，为了更好得理解，我们先理清一下NSURLSessionTask类以及它的子类、NSURLSessionTaskDelegate协议和它的子协议之间的关系，以及各种代理方法调用的时机。
先看一张图：
<img src="http://qiniu.storage.mikezh.com/img%2FNSObjectx.jpg" width="800" alt="NSURLSessionTask类、NSURLSessionTaskDelegate协议"/>

其中的调用是指，task在resume之后会调用的Session对应的代理方法声明在的协议，例如：
当执行一个NSURLSessionDataTask类型的任务resume之后，负责创建它的session将会调用在NSURLSessionDataDelegate中定义的几个方法:

```objectivec
- URLSession: dataTask: didReceiveResponse: completionHandler:
- URLSession: dataTask: didBecomeDownloadTask:
- URLSession: dataTask: didBecomeStreamTask:
- URLSession: dataTask: didReceiveData:
- URLSession: dataTask: willCacheResponse: completionHandler:
```
由于NSURLSessionDataDelegate协议遵守了NSURLSessionTaskDelegate和NSURLSessionDelegate，所以也会调用这样几个方法：

```objectivec
// 在NSURLSessionDelegate中声明的
- URLSession: didBecomeInvalidWithError:
- URLSession: didReceiveChallenge: completionHandler:
- URLSessionDidFinishEventsForBackgroundURLSession:

// 在NSURLSessionTaskDelegate中声明的
- URLSession: task: willPerformHTTPRedirection: newRequest: completionHandler:
- URLSession: task: didReceiveChallenge: completionHandler:
- URLSession: task: needNewBodyStream:
- URLSession: task: didSendBodyData: totalBytesSent: totalBytesExpectedToSend:
- URLSession: task: didCompleteWithError:
```
实际上你无法通过session来创建NSURLSessionTask，只能创建它的子类来使用，iOS并没有提供可以直接创建它的方法:

1.不可能通过alloc init创建 ，因为创建之后 无法给request属性(readonly)赋值，网络请求无法进行。

2.NSURLSession、NSURLSession(NSURLSessionAsynchronousConvenience) 没有提供直接创建的方法。
或许apple本来就打算将这个类设计为抽象类，而只能使用继承它的类。

而只要是使用了session类进行创建任何一个dataTask、uploadTask或者是downloadTask就会调用在NSURLSessionDelegate中和NSURLSessionTaskDelegate中声明的代理方法,这些代理方法大都是进行网络请求的配置，少部分涉及到数据处理，而子协议NSURLSessionDataDelegate、NSURLSessionDownloadDelegate和NSURLSessionSteamDelegate都是具体的数据处理方法。


经过一些简单的测试：看看一些方法的调用顺序，使用dataTask进行一个普通的网络请求：
如果使用的是GET请求，或者使用的是POST请求、但是HTTPBody没有数据，主要调用两个代理方法：

```objectivec
- URLSession: dataTask: didReceiveData: // 当服务端有数据返回时调用，没有数据返回则不调用
- URLSession: task: didCompleteWithError: // 在请求完成之后必调用
```
如果是POST请求，并且HTTPBody中带有数据，那么主要调用以下几个方法(实际上不管创建的任务是dataTask或是uploadTask都是这样，毕竟uploadTask是继承自dataTask的)：

```objectivec
- URLSession: task: didSendBodyData: totalBytesSent: totalBytesExpectedToSend: // 当HTTPBody中有数据时调用
- URLSession: dataTask: didReceiveData: // 同上
- URLSession: task: didCompleteWithError: // 同上
```
当进行一些下载操作，使用downloadTask的时候：

```objectivec
- URLSession: downloadTask: didWriteData: totalBytesWritten: totalBytesExpectedToWrite: // 间歇性调用
- URLSession: downloadTask: didFinishDownloadingToURL: // 下载完成时调用
- URLSession: task: didCompleteWithError: // 本次网络访问完成时调用 在上面的方法调用之后调用
```

可以发现，但凡是session进行的网络请求都会最终调用`- URLSession: task: didCompleteWithError:`,而在在之前调用的代理方法，会因request是否携带数据，访问完成的时候服务端是否有response的数据，还有使用的task的类型会有一些差别。下面会针对uploadTask的使用和downloadTask的使用细说这些差别，以及介绍一些实现上传和下载的具体方案。

### 第二部分 上传

使用上传归根结底都会使用apple的uploadTask，翻看AFN的源码(仅仅是session部分)也都是使用了苹果的三个创建uploadTask的方法完成的。
apple的三个方法都是一个思路：将要上传的文件的二进制写入到HTTPBody中，
按照有没有使用Form可以分为两类：

1.没有使用form

```objectivec
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL;
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData;
```
2.使用了form

```objectivec
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request;
```
下面分别介绍一下：

#### 不使用表单的情况

AFN没有使用html表单直接上传的方式比较简单，实现上是直接调用了apple的`- uploadTaskWithRequest: fromFile:`方法或者`- uploadTaskWithRequest: fromData:`，关于苹果的这两个方法，苹果给出这样的文档
创建一个任务，这个任务能对指定的URLRquest对象执行HTTP请求和上传提供的数据。

对于request的参数有一点需要注意的是：它的body stream和body data会被忽略，只使用fromData参数提供的数据。对于request对象还有一个要求，必须是包含了request body，因此HTTP方法可以是POST或者PUT，另外可以使用HTTP的RequestHeader提供一些上传的元数据，如文件名字等。其实这两个方法内部的实现中，是将要上传的数据覆盖写入到了HTTPBody中。

AFNURLSessionManager对上面两个苹果的方法进行了再一次的封装，这个封装就是将代理方法的处理交给了AFURLSessionManagerTaskDelegate类，同时将传入的进度NSProgress对象的指针指向了AFURLSessionManagerTaskDelegate对象的属性progress，将task处理完成的回调赋给它的属性AFURLSessionTaskCompletionHandler。

如果按照这种方式进行文件上传，可以按照如下方式使用AFN：

```objectivec
// 文件上传,不使用表单(只能上传单个文件),需要服务端的配合：
// 1.服务端从HTTPBody得到文件内容的二进制
// 2.将二进制存入文件中，并命名

// 这个方法内部直接使用apple的uploadTaskWithRequest创建任务， uploadTask具体的实现是：
// 将文件的二进制写入到HTTPBody中
- (void)uploadFileNoFormWithURLString:(NSString *)urlString fromFile:(NSURL *)fileURL orFromData:(NSData *)bodyData progress:(NSProgress * __autoreleasing *)progress success:(void(^)(id responseObject))success failure:(void(^)(NSError *error))failure {
    
    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
    NSURL *url = [NSURL URLWithString:urlString];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST"; // 必须要使用POST  否则会使用默认的GET， 这样服务器得到的input和HTTPBody内容不同，这是因为使用这种方式上传文件实际上是将文件的二进制写入到HTTPBody中。
    
    void (^completionBlock)(id responseObject, NSError *error) = ^(id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(error);
            }
        } else {
            if (success) {
                success(responseObject);
            }
            
        }
    };
    // 这里实际调用的URLSessionManager的方法，而不是HTTPSessionManager的方法
    if (fileURL) {
        [[manager uploadTaskWithRequest:request fromFile:fileURL progress:progress completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
            completionBlock(responseObject, error);
        }] resume];
        return;
    }
    if (bodyData) {
        [[manager uploadTaskWithRequest:request fromData:bodyData progress:progress completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
            completionBlock(responseObject, error);
        }] resume];
    }
    return;
}
```

#### 使用表单的情况

使用表单其实是对HTTPBody数据格式进行改造，类似于html中使用表单控件上传，同样的在底层上也是模拟html表单上传数据的格式。经过这样的模拟之后，服务端接收到的每个文件对应到一个表单域(field)的值，这样`方便了服务端的处理和前台的html页面的统一`。

AFN使用这种方式上传的实现依靠的是apple的`- uploadTaskWithStreamedRequest:`这个方法,对于这方法，文档中有这样的说明：
用一个指定的request创建upload task。之前的request的body stream数据会被忽略，如何需要上传数据调用URLSession:task:needNewBodyStream:方法。也就是说在这个方法中设置的request的HTTPBody和HTTPBodyStream会被忽略，而真正上传的数据是从代理方法`URLSession:task:needNewBodyStream:`中取得的。

AFN的做法是：在AFHTTPRequestSerializer对象的`multipartFormRequestWithMethod: URLString: parameters: constructingBodyWithBlock: error:`方法中将要上传的数据组装为NSInputStream对象(其实是NSInputStream的子类AFMultipartBodyStream)并设置为request的HTTPBodyStream属性，然后返回这个request，当真正执行到`URLSession:task:needNewBodyStream:`方法时，会从request中将这个InputStream取出，然后复制，最终传递给用来接收它的回调completionHandler。

```objectivec
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
 needNewBodyStream:(void (^)(NSInputStream *bodyStream))completionHandler
{
    NSInputStream *inputStream = nil;

    if (self.taskNeedNewBodyStream) {
        inputStream = self.taskNeedNewBodyStream(session, task);
    } else if (task.originalRequest.HTTPBodyStream && [task.originalRequest.HTTPBodyStream conformsToProtocol:@protocol(NSCopying)]) {
        inputStream = [task.originalRequest.HTTPBodyStream copy];
    }

    if (completionHandler) {
        completionHandler(inputStream);
    }
}
```
这里AFN拼接表单域的方式和html在浏览器中的行为一致，AFN的拼接方式完全按照浏览器的方式模拟了这个过程，主要通过两个类来实现，用于拼接的AFStreamingMultipartFormData类和用于Strea转换的AFMultipartBodyStream类。

1.首先是将parameter参数转为AFQueryStringPair数组，并将每个AFQueryStringPair对象的元素转为filed和value的二进制形式，然后使用AFStreamingMultipartFormData对象的`- appendPartWithFormData: name:`方法将它们拼接为下面格式（boundary生成之后的boundary）

```sh
--boundary
Content-Disposition: form-data; name="xx";


二进制data
```
2.将parameter的传递的参数拼接完成后拼接文件：
利用request中传递过来的block继续给上面的AFStreamingMultipartFormData兑现追加内容：使用`-appendPartWithFileURL: name: error:`方法拼接文件，拼接为如下格式：

```sh
--boundary
Content-Disposition: form-data; name="xxx"; filename="xxx"
Content-Type: xxx/xxx

二进制data
```
最后还得加上头部

```sh
Content-Type:multipart/form-data; boundary=生成后的boundary
```
还有尾部

```sh
--生成后的boundary--
```
这是最终拼接的结果，实际上AFN的拼接过程比这个要复杂，它并没有将最终形式的'串'直接拼接出来，而是将每一个部分转为一个AFHTTPBodyPart对象，存储到AFStreamingMultipartFormData对象的属性bodyStream中，bodyStream是一个AFMultipartBodyStream对象，使用的是的`- appendHTTPBodyPart:`方法将AFHTTPBodyPart存储到了它自己的可变数组属性HTTPBodyParts中，最后在AFStreamingMultipartFormData对象的以下方法完成拼接：

```objectivec
- (NSMutableURLRequest *)requestByFinalizingMultipartFormData {
    if ([self.bodyStream isEmpty]) {
        return self.request;
    }

    // Reset the initial and final boundaries to ensure correct Content-Length
    [self.bodyStream setInitialAndFinalBoundaries];
    [self.request setHTTPBodyStream:self.bodyStream];

    [self.request setValue:[NSString stringWithFormat:@"multipart/form-data; boundary=%@", self.boundary] forHTTPHeaderField:@"Content-Type"];
    [self.request setValue:[NSString stringWithFormat:@"%llu", [self.bodyStream contentLength]] forHTTPHeaderField:@"Content-Length"];

    return self.request;
}
```
可以看到bodyStream在加了头部和尾部之后赋值给了request，这里最关键的就是AFMultipartBodyStream(bodyStream的类型)已经重写了InputStream的`read:maxLength:`和`getBuffer:length:`连个方法，这样当bodyStream被读取的时候会按照这两个方法的实现，按照刚才介绍的那种形式将数据拼接起来。

介绍完了这些，我们看一下使用这种方案进行上传文件的常用代码：

```objectivec
// 多文件上传，使用POST方法，使用的是表单的方式，需要服务端的脚本支持
// 使用表单上传，将文件作为表单的中的一个field
- (void)uploadFileUseFormWithURLString:(NSString *)urlString parameter:(id)parameter constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block progress:(NSProgress * __autoreleasing *)progress success:(void(^)(id responseObject))success failure:(void(^)(NSError *error))failure {
    
    AFHTTPSessionManager *mgr = [AFHTTPSessionManager manager];
    [mgr POST:urlString parameters:parameter constructingBodyWithBlock:block success:^(NSURLSessionDataTask *task, id responseObject) {
        if (success) {
            success(responseObject);
        }
    } failure:^(NSURLSessionDataTask *task, NSError *error) {
        if (failure) {
            failure(error);
        }
    }];
}

// 也可以使用提前组装好request的方法
- (void)uploadFileUseFormWithStreamedRequest:(NSURLRequest *)urlRequest progress:(NSProgress * __autoreleasing *)progress success:(void(^)(id responseObject))success failure:(void(^)(NSError *error))failure {
    AFHTTPSessionManager *mgr = [AFHTTPSessionManager manager];
    [[mgr uploadTaskWithStreamedRequest:urlRequest progress:progress completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(error);
            }
        } else {
            if (success) {
                success(responseObject);
            }
        }
    }] resume];
}
```
这里提供了两种方案，底层代码完全一样，第一种是在使用时拼接表单，第二种是将表单和parameter组装到request之后直接调用，调用方法如下：

```objectivec
// 对第一种方式的调用
NSString *uploadURLString = @"http://127.0.0.1/post/upload-multipart.php";
NSDictionary *parameter = @{@"username": @"Mike"};
NSProgress *progress1 = nil;

[self uploadFileUseFormWithURLString:uploadURLString parameter:parameter constructingBodyWithBlock:^(id<AFMultipartFormData> formData) {
    NSString *documentFolder = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject];
    NSURL *fileURL = [NSURL fileURLWithPath:[documentFolder stringByAppendingPathComponent:@"1.txt"]];
    [formData appendPartWithFileURL:fileURL name:@"userfile[]" error:NULL];
    NSURL *fileURL1 = [NSURL fileURLWithPath:[documentFolder stringByAppendingPathComponent:@"2.jpg"]];
    [formData appendPartWithFileURL:fileURL1 name:@"userfile[]" fileName:@"aaa.jpg" mimeType:@"image/jpeg" error:NULL];
} progress:&progress1 success:^(id responseObject) {
    NSLog(@"%@", responseObject);
} failure:nil];

// 对第二种方式(预先组装request)的调用
NSString *documentFolder = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject];
NSURL *fileURL1 = [NSURL fileURLWithPath:[documentFolder stringByAppendingPathComponent:@"1.txt"]];
NSURL *fileURL2 = [NSURL fileURLWithPath:[documentFolder stringByAppendingPathComponent:@"2.jpg"]];

NSMutableURLRequest *request = [[AFHTTPRequestSerializer serializer] multipartFormRequestWithMethod:@"POST" URLString:uploadURLString parameters:parameter constructingBodyWithBlock:^(id<AFMultipartFormData> formData) {
    [formData appendPartWithFileURL:fileURL1 name:@"userfile[]" fileName:@"1.txt" mimeType:@"text/plain" error:nil];
    [formData appendPartWithFileURL:fileURL2 name:@"userfile[]" fileName:@"aaa.jpg" mimeType:@"image/jpeg" error:nil];
} error:nil];

NSProgress *progress2 = nil;
[self uploadFileUseFormWithStreamedRequest:request progress:&progress2 success:^(id responseObject) {
    NSLog(@"%@", responseObject);
} failure:nil];
```
两种方法的底层实现和结果是完全一致的。

#### 针对RESTful Web应用的文件上传
使用RESTful风格的应用，默认对网络资源增、删、改、查对应于HTTP的PUT、DELETE、POST、GET方法。RESTful更多的是强调服务端的技术，而对于iOS客户端而言，网络访问的代码变动不是特别大，例如针对文件上传，服务器要具有处理文件上传的能力，而且`不用像html表单那样针对特定的页面定制具有专一功能的处理脚本`，对于这种需求，也许webDav是一个很好的工具，当然web服务端也可能会针对各自的语言和平台使用各自的文件处理中间件，但服务端的处理不是本文的重点，不再多说。另外，针对文件上传服务端可能要做一些用户验证，iOS客户端就要做一些配合了。

以'我要在http://127.0.0.1/uploads这个url映射的目录下放置一个文件名为12345.png的图片文件'为例，当我要进行的操作已经能用自然语言表达时，便可以以REST的思路来写代码了，那么：
我要使用的HTTP方法是PUT
我要操作的url为http://127.0.0.1/uploads
我会写一个方法将文件名传入，将文件的URL或者二进制传入,如下：

```objectivec
- (void)uploadFileUseRESTWithURLString:(NSString *)urlString rename:(NSString *)rename fromFile:(NSURL *)fileURL orFromData:(NSData *)bodyData progress:(NSProgress * __autoreleasing *)progress success:(void(^)(id responseObject))success failure:(void(^)(NSError *error))failure {
    
    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
    
    NSString *urlStringPath = [urlString stringByAppendingPathComponent:rename];
    NSURL *url = [NSURL URLWithString:urlStringPath];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    
    request.HTTPMethod = @"PUT";
    // 身份验证 BASIC 方式
    NSString *usernameAndPassword = @"admin:123456";
    NSData *data = [usernameAndPassword dataUsingEncoding:NSUTF8StringEncoding];
    NSString *authString = [@"BASIC " stringByAppendingString:[data base64EncodedStringWithOptions:0]];
    [request setValue:authString forHTTPHeaderField:@"Authorization"];
    
    
    void (^completionBlock)(id responseObject, NSError *error) = ^(id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(error);
            }
        } else {
            if (success) {
                success(responseObject);
            }
            
        }
    };
    if (fileURL) {
        [manager uploadTaskWithRequest:request fromFile:fileURL progress:progress completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
            completionBlock(responseObject, error);
        }];
        return;
    }
    if (bodyData) {
        [manager uploadTaskWithRequest:request fromData:bodyData progress:progress completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
            completionBlock(responseObject, error);
        }];
    }
    return;
}
```
调用也是非常的简单:

```objectivec
NSString *filePath = [[NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject] stringByAppendingPathComponent:@"1.png"];
NSURL *fileURL = [NSURL fileURLWithPath:filePath];
NSProgress *progress = nil;
[self uploadFileUseRESTWithURLString:@"http://127.0.0.1/uploads" rename:@"12345.png" fromFile:fileURL orFromData:nil progress:&progress success:^(id responseObject) {
    NSLog(@"%@", responseObject);
} failure:nil];
```

### 第三部分 下载
AFN使用的下载同样是调用了apple提供的方法:

```objectivec
- downloadTaskWithRequest:
- downloadTaskWithURL:
- downloadTaskWithResumeData:
```
毫无疑问的是使用url最简单了，毕竟大部分的下载是无需构建request的。AFN使用名字相似的方法对系统方法进行的一层包装，通过sessionManager统一管理task。
例如完成一个简单的下载任务：

```objectivec
- (void)downloadWithURLString:(NSString *)urlString progress:(NSProgress * __autoreleasing *)progress completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler {
    AFURLSessionManager *manager = [[AFURLSessionManager alloc] init];
    
    NSMutableCharacterSet *mutableCharacterSet = [[NSMutableCharacterSet alloc] init];
    [mutableCharacterSet formUnionWithCharacterSet:[NSCharacterSet URLHostAllowedCharacterSet]];
    [mutableCharacterSet formUnionWithCharacterSet:[NSCharacterSet URLPathAllowedCharacterSet]];
    NSString *escapedURLString = [urlString stringByAddingPercentEncodingWithAllowedCharacters:mutableCharacterSet];
    
    NSURL *url = [NSURL URLWithString:escapedURLString];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];

    [[manager downloadTaskWithRequest:request progress:progress destination:^NSURL *(NSURL *targetPath, NSURLResponse *response) {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
        NSString *fileName = httpResponse.suggestedFilename ?: urlString;
        NSString *filePath = [[NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject] stringByAppendingPathComponent:fileName];
        NSURL *destinationURL = [NSURL fileURLWithPath:filePath];
        return destinationURL;
    } completionHandler:^(NSURLResponse *response, NSURL *filePath, NSError *error) {
        completionHandler(response, filePath, error);
    }] resume];
}
```
好吧，那段进行urlEncode的代码确实碍眼。看一下下载的功能非常简单，只要使用几行代码就完成了，这里有一个调用并获取进度的示例：

```objectivec
NSString *urlString = @"http://127.0.0.1/static/功夫熊猫.mp4";
NSProgress *progress = nil;
[self downloadWithURLString:urlString progress:&progress completionHandler:^(NSURLResponse *response, NSURL *filePath, NSError *error) {
    NSLog(@"%@", error);
}];
[progress addObserver:self forKeyPath:@"completedUnitCount" options:NSKeyValueObservingOptionNew context:nil];}


- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    if ([object isKindOfClass:[NSProgress class]]) {
        NSProgress *p = object;
        NSLog(@"已完成大小:%lld  总大小:%lld", p.completedUnitCount, p.totalUnitCount);
        NSLog(@"进度:%0.2f%%", p.fractionCompleted * 100);
    }
}
```
可以利用这种方式来观察进度,它的原理是将外部传来的NSProgress指针指向AFURLSessionManagerTaskDelegate对象的progress属性，AFURLSessionManagerTaskDelegate对象会在- URLSession: downloadTask: didWriteData: totalBytesWritten: 

totalBytesExpectedToWrite:方法中改变自身的progress属性值，这是外部的Progress的值也就改变了。
也可以使用URLSessionManager的downloadProgressForTask:方法获取指定task的完成进度，不过这要进行对task的统一管理。

在开发中经常会遇到下载管理，UI要获取到下载的进度，通常有两种方法：

1.下载器管理进度，进度改变发送通知，监听进度改变的UI控件更新(非常消耗性能)

2.UI控件主动获取任务管理器中的任务，间隔性地获取进度。

说到NSURLSessionDownloadDelegate中声明的三个方法：

```objectivec
- URLSession:session downloadTask:downloadTask didFinishDownloadingToURL: // 在下载完成的时候调用
- URLSession: downloadTask: didWriteData: totalBytesWritten: totalBytesExpectedToWrite: // 在下载的过程中间歇性调用
- URLSession: downloadTask: didResumeAtOffset: expectedTotalBytes:
```
对于第三个方法到底什么时候调用，就不得不说一下使用NSURLSession的`- downloadTaskWithResumeData:`方法创建downloadTask了。
这个方法用resumeData创建一个downloadTask，如果downloadTask不能被成功地恢复, URLSession:task:didCompleteWithError: 会被调用。
说白了它被用来恢复已经暂停的downloadTask，那么downloadTask如何暂停呢，毕竟都没有被暂停何来恢复。可以主动调用downloadTask的一个对象方法：

```objectivec
- (void)cancelByProducingResumeData:(void (^)(NSData * __nullable resumeData))completionHandler;
```
这个方法会让downloadTask暂停，它接收一个回调，在任务暂停之后调用，一般在这个回调内部记录一下恢复点的数据resumeData，resumeData参数是将来用来继续的参数。resumeData只是一个chunk，而不是已经下载的全部数据，因此无法通过它实现断点续传，只能实现简单的暂停和继续，并且要保证通过resume创建downloadTask时使用的session和创建被取消的downloadTask时使用的session是同一个，也就是所谓的session没有离线 。
这是一些实现暂停和继续的示例代码：

```objectivec
// 暂停下载
- (IBAction)pauseButtonDidClicked:(UIButton *)sender {
    [self.downloadTask cancelByProducingResumeData:^(NSData *resumeData) {
        self.resumeData = resumeData; 
        self.downloadTask = nil;
    }]; // 这是一个异步的方法
}

// 继续
- (IBAction)resumeButtonDidClicked:(UIButton *)sender {
    if (self.resumeData == nil) {
        return;
    }
    self.downloadTask = [self.session downloadTaskWithResumeData:self.resumeData];    
    self.resumeData = nil;    
    [self.downloadTask resume];
}
```
那么以AFN的方式，就得保存manager，因为只要manager没有改变session就没有改变，同时应该将下载的任务添加到一个任务管理器中，对任务进行统一的调配，这样就可以使用使用下面的步骤进行任务的暂停和继续：

1.先取得downloadTask调用它的cancelByProducingResumeData:将resumeData保存起来

2.需要继续的时候使用创建了上面的dataTask的manager和保存的resumeData进行恢复,manager调用downloadTaskWithResumeData:就可以。