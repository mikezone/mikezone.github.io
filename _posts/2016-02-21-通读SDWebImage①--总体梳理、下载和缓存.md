---
layout: cnblog_post
title:  "通读SDWebImage①--总体梳理、下载和缓存"
date:   2016-02-21 13:50:39
categories: iOS
---
<div><a name="labelTop"></a></div>
<!--Category-->
<div id="navCategory">
	<b>本文目录</b>
	<ul>
		<li><a href="#anchor1_0">下载操作SDWebImageDownloaderOptions和下载过程实现</a></li>
                <li><a href="#anchor2_0">下载管理SDWebImageDownloader</a></li>
                <li><a href="#anchor3_0">缓存SDImageCache</a></li>
                <li><a href="#anchor4_0">SDWebImageManager：按需下载->完成缓存->缓存管理等一系列完整的流程线</a></li>
	</ul>
</div>
<!--Category结束-->

要写点关于SDWebImage的文章了，这段时间看的不少，总体的感受是SDWebImage的代码不如AFN那么规整、有条理，并没有吐槽的意思，稍微细细看一下就会有这样的感受。本篇文章不会用大量的篇幅来介绍SDWebImage如何使用，而是更多地介绍SDWebImage的整体思路和一些实现细节，还有介绍一些不是特别常用的一些功能(因为有不少iOS开发人员还只是会使用sd_setImageWithURL)。首先我们要看一下SDWebImage的整体结构：
<img src="http://qiniu.storage.mikezh.com/img%2FSnip20160218_9.png" width="900" alt="SDWebImage整体结构图"/>
这里我要说明的一点是我当前使用SD的git提交版本是e41af47e2f5de9317d55083e23168e076b550e34(Sat Jan 30 02:54:23 2016 +0100)。让我们看一下这张图的内容。
可以将SDWebImage的框架分为三个部分：

1.适配<br/>
SDWebImage<br/>
iOS版本、编译指令的适配、线程切换的宏、还有一个导出的内联函数，用于根据image的命名key 将image转换成相应的scale的UIImage类型以完成对Image的缩放，如文件名带&#64;2x，将按照2倍缩放。

2.Util工具<br/>
核心的类就是SDWebImageManager它负责创建和管理下载任务、对缓存操作进行管理，我们通常使用的UIImageView的WebCache分类下的sd_setImageWithURL方法的实现就依赖于这个类，其他View分类的设置图片的方法也实现也类似。<br/>
SDWebImageManager实现下载依赖于下载器：SDWebImageDownloader，下载器负责管理下载任务，而执行下载任务是由SDWebImageDownloaderOperation操作完成。<br/>
SDWebImageManager实现缓存依赖于缓存管理：SDImageCache，能够完成图片的内存缓存和磁盘缓存，还可以查询指定url的图片是否进行了缓存、取出缓存等操作。<br/>
下载和缓存的过程中会调用适配模块进行将图片转为合适的尺寸，使用解压模块将被压缩的图片解压后完成缓存。

3.分类<br/>
包括两部分：①.视图分类、②.用于图片格式处理和格式转换的模块。

①.视图分类<br/>
视图分类中有一个基本的分类：<br/>
UIView+WebCacheOperation这个分类用于完成将组合操作(SD定义了能够实现下载和缓存的组合操作类SDWebImageCombinedOperation)与View绑定、取消绑定和移除绑定等功能。其他视图分类的实现都依赖于这个分类。<br/>
MKAnnotationView+WebCache、UIImageView+WebCache、UIImageView+HighlightedWebCache对view中的图片的加载过程的实现比较相似(后面会介绍),UIButton+WebCache分类中针对UIButton的不同的State可以设置不同的image。

②.用于图片格式处理和格式转换的模块<br/>
NSData+ImageContentType这个分类只有一个方法sd_contentTypeForImageData:，是根据图片的二进制data的第一个字节的数据，得到图片相应的MIME类型。<br/>
UIImage+MultiFormat也只有一个方法sd_imageWithData:，根据传入的NSData，读取到MIME类型然后转换成对应的UIImage。<br/>
UIImage+GIF根据传入的值如文件名或者NSData，得到对应的GIF图的UIImage对象，实际上是一个animatedImage。<br/>
UIImage+WebP根据传入的NSData，得到对应的WebP图的UIImage对象，这个方法的实现依赖于WebP库，需要到google下载libwebp。<br/>

以上是从代码的角度分析了SD可以完成的工作，而在github上SD的主页可以看到，它的自我介绍中的主打功能：

```sh
提供UIImageView的一个分类，以支持网络图片的加载与缓存管理
一个异步的图片加载器
一个异步的内存+磁盘图片缓存
支持GIF图片
支持WebP图片
后台图片解压缩处理
确保同一个URL的图片不被下载多次
确保虚假的URL不会被反复加载
确保下载及缓存时，主线程不被阻塞
```
本篇文章的内容主要涉及到4个类：`SDWebImageDownloaderOptions`、`SDWebImageDownloader`、`SDImageCache`、`SDWebImageManager`，详细介绍如何实现下载和缓存的以及如何在这个过程中做到上面提到的‘三个确保’。至于其他内容(如GIF和WebP图片的加载)以后会一一介绍。
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor1_0">回到顶部</a></div>

### 下载操作SDWebImageDownloaderOptions和下载过程实现
SDWebImage下载图片使用的是`SDWebImageDownloaderOperation`，它是一个`NSOperation`的子类，同时遵守了`<SDWebImageOperation>`协议（其实这个协议只声明了一个方法cancel用于取消操作）。这个操作负责管理下载的选项，进行网络访问时的request，设置网络处理质询的凭据，进行网络连接接收数据，管理网络访问的response和是否解压的选项等。总之，它的任务就是网络访问配置、进行网络访问以及处理数据。

每一个NSOperation都是为了完成一项任务而诞生的，而`SDWebImageDownloaderOperation`的任务就是负责依照指定的下载选项，使用将指定的urlRequest创建NSURLConnection对象进行网络连接(NSURLConnection对象的代理就是SDWebImageDownloaderOperation自己)，进行对图片的下载。在下载的过程中对图片数据进行拼接，可以实现对进度progress的跟踪，在下载之后可以将接收到的图片数据转换、解压等，并完成一个下载完成的回调。如果网路访问过程中接收到质询，则使用服务端凭据或者本地存储的凭据处理质询；如果下载失败了，则发送错误通知，执行完成回调，并结束下载任务。

`SDWebImageDownloaderOperation`类主要有以下几个属性：<br/>
1.`NSURLRequest *request`:下载时进行网络请求的request，由构造方法传入。<br/>
2.`BOOL shouldDecompressImages`:下载后是否需要解压图片。<br/>
3.`BOOL shouldUseCredentialStorage`:URLConnection是否需要咨询凭据仓库来对连接进行授权，默认是YES。<br/>
这是NSURLConnectionDelegate的-connectionShouldUseCredentialStorage:方法的返回值<br/>
4.`NSURLCredential *credential`:在`-connection:didReceiveAuthenticationChallenge:`方法中验证质询时使用的凭据
 已经存在的request，URL的用户名或密码构成的凭据会覆盖这个值，具体解释参见SDWebImageDownloader部分。<br/>
5.`SDWebImageDownloaderOptions options`:readonly下载选项,由构造方法传入。<br/>
6.`NSInteger expectedSize`:预期的文件长度，使用NSInteger完全够用。<br/>
7.`NSURLResponse *response`:connection对象进行网络访问，接收到的的response
要注意的是：下载选项是在`SDWebImageDownloader`中定义的，`SDWebImageDownloader`是下载器负责管理下载队列和控制下载过程(通过调用`SDWebImageDownloaderOperation`的方法)。下载选项`SDWebImageDownloaderOptions`的定义如下：

```objectivec
typedef NS_OPTIONS(NSUInteger, SDWebImageDownloaderOptions) {
    SDWebImageDownloaderLowPriority = 1 << 0,
    /// 渐进式下载，如果设置了这个选项，会在下载过程中，每次接收到一段chunk数据就调用一次完成回调(注意是完成回调)回调中的image参数为未下载完成的部分图像
    SDWebImageDownloaderProgressiveDownload = 1 << 1,

    /// 通常情况下request阻止使用NSURLCache. 这个选项会用默认策略使用NSURLCache 
    SDWebImageDownloaderUseNSURLCache = 1 << 2,

    /// 如果从NSURLCache中读取图片，会在调用完成block时，传递空的image或imageData \
     * (to be combined with `SDWebImageDownloaderUseNSURLCache`).
    SDWebImageDownloaderIgnoreCachedResponse = 1 << 3,

    /// 系统为iOS 4+时，如果应用进入后台，继续下载。这个选项是为了实现在后台申请额外的时间来完成请求。如果后台任务到期，操作会被取消。
    SDWebImageDownloaderContinueInBackground = 1 << 4,

    /// 通过设置NSMutableURLRequest.HTTPShouldHandleCookies = YES的方式来处理存储在NSHTTPCookieStore的cookies
    SDWebImageDownloaderHandleCookies = 1 << 5,

   	/// 允许不受信任的SSL证书，在测试环境中很有用，在生产环境中要谨慎使用
    SDWebImageDownloaderAllowInvalidSSLCertificates = 1 << 6,

    /// 将图片下载放到高优先级队列中
    SDWebImageDownloaderHighPriority = 1 << 7,
};
```
这些选项主要涉及到下载的优先级、缓存、后台任务执行、cookie处理以及证书认证几个方面，在创建下载操作的时候可以使用组合的选项以完成一些特殊的需求。

`SDWebImageDownloaderOperation`只对外提供了一个对象方法`- initWithRequest: options: progress: completed: cancelled:`，它使用默认的属性值初始化一个`SDWebImageDownloaderOperation`对象。

下面我们看一下`SDWebImageDownloaderOperation`对NSOperation的`-start`方法的重写，毕竟这是完成下载任务的核心代码。以下是将-start提取出来的部分代码

```objectivec
@synchronized (self) {
        if (self.isCancelled) {
            self.finished = YES;
            [self reset]; // 将各个属性置空。包括取消回调、完成回调、进度回调，用于网络连接的connection，用于拼接数据的imageData、记录当前线程的属性thread。
            return;
        }

#if TARGET_OS_IPHONE && __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_4_0
 // 使用UIApplication的beginBackgroundTaskWithExpirationHandler方法向系统借用一点时间，继续执行下面的代码来完成connection的创建和进行下载任务。
 // 在后台任务执行时间超过最大时间时，也就是后台任务过期执行过期回调。在回调主动将这个后台任务结束。
 /*
 		^{
                __strong __typeof (wself) sself = wself;

                if (sself) {
                    [sself cancel];

                    [app endBackgroundTask:sself.backgroundTaskId];
                    sself.backgroundTaskId = UIBackgroundTaskInvalid;
                }
            }
 */
#endif
        self.executing = YES; // 标记状态
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO]; // 创建用于下载的connection
        self.thread = [NSThread currentThread]; // 记录当前线程
    }

    [self.connection start]; 

    if (self.connection) {
        if (self.progressBlock) { // 任务开始立刻执行一次进度回调
            self.progressBlock(0, NSURLResponseUnknownLength);
        }
        dispatch_async(dispatch_get_main_queue(), ^{ // 发送开始下载的通知，object为operation本身
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStartNotification object:self];
        });
        
        if (floor(NSFoundationVersionNumber) <= NSFoundationVersionNumber_iOS_5_1) {
            CFRunLoopRunInMode(kCFRunLoopDefaultMode, 10, false);
        }
        else {
            CFRunLoopRun();
        }
        // 当runloop开启之后，线程切换到runloop中的任务，开始下载图片，所以下面的代码是经过一段时间的延迟执行的，也就是当connection的网络访问进行之后，才会执行下面的代码。
        // 这个时候可以进行一些判断，如图片是否被正确地下载完成。
        if (!self.isFinished) {
            [self.connection cancel];

            // NSURLConnectionDelegate代理方法
            // 主动调用 并制造一个错误，这样做的目的是因为这个方法一旦调用，代理就不会再接收connection的消息，也就是不在调用其他的任何代理方法了，connection彻底结束。
            [self connection:self.connection didFailWithError:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorTimedOut userInfo:@{NSURLErrorFailingURLErrorKey : self.request.URL}]];
        }
    } else { // connectin 创建失败，这里直接执行完成回调，并传递一个connection没有初始化的错误
        if (self.completedBlock) {
            self.completedBlock(nil, nil, [NSError errorWithDomain:NSURLErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Connection can't be initialized"}], YES);
        }
    }

    // 运行到这里说明下载操作已经完成(无论成功还是失败)，因此没有必要在后台运行。使用endBackgroundTask:
#if TARGET_OS_IPHONE && __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_4_0
    Class UIApplicationClass = NSClassFromString(@"UIApplication");
    if(!UIApplicationClass || ![UIApplicationClass respondsToSelector:@selector(sharedApplication)]) {
        return;
    }
    if (self.backgroundTaskId != UIBackgroundTaskInvalid) {
        UIApplication * app = [UIApplication performSelector:@selector(sharedApplication)];
        [app endBackgroundTask:self.backgroundTaskId];
        self.backgroundTaskId = UIBackgroundTaskInvalid;
    }
#endif
```
这些就是一次下载操作要执行的任务，但是数据处理是下载任务的关键，`SDWebImageDownloaderOperation`通过NSURLConnection的代理方法完成对下载的图片的数据处理，主要用到以下几个方法：

```objectivec
// NSURLConnectionDataDelegate中声明
connection: didReceiveResponse: // 接收到服务端的response时，执行一次
connection: didReceiveData: // 每次接收到chunk数据都会调用
connectionDidFinishLoading: // 当连接结束的时候调用一次
connection: willCacheResponse: // 要进行缓存之前调用

// NSURLConnectionDelegate中声明 
connection: didFailWithError: // 连接失败，或者没有成功下载完成调用
connectionShouldUseCredentialStorage: // 指定是否需要使用本地凭据进行验证
connection: willSendRequestForAuthenticationChallenge: // 处理服务端过来的质询
```
下面我们看一下除了上面标注的一些基本的功能外，`SDWebImageDownloaderOperation`在每个方法内部还有哪些细节性的工作。

1.`connection: didReceiveResponse:`<br/>
主要完成以下工作：

```objectivec
if (statusCode<400并且不等于304) {
	// 设置预期文件长度属性的值
	// 立刻完成一次进度回调，传递的参数为0，
	// 初始化用于拼接图片二进制数据的属性imageData
	// 设置response属性为服务端返回的response值
	// 向主队列同步发送一个接收到response的通知
} else {
	// 如果statusCode为304，也就是服务端Not Modified并且拿到了本地的HTTP缓存，取消操作,发送操作停止的通知，执行完成回调，停止当前的runloop，设置下载完成标记为YES，正在执行标记为NO，将属性置空。
}
```
2.`connection: didReceiveData:`

```objectivec
使用自身属性imageData拼接接收的数据
if (下载选项设置了SDWebImageDownloaderProgressiveDownload) {
	取得已经拼接完的imageData，创建一个CGImageSourceRef类型的imageSouce，使用imageSouce创建CGImageRef类型的对象partialImageRef，代表着要下载的图片的一部分，调整方向并将使用`UIImage imageWithCGImage:partialImageRef`将其导出为UIImage，释放掉partialImageRef，并在主线程同步执行一次完成回调，指定第一个参数为刚才到处的UIImage，最后释放掉imageSource占用的空间。
}
执行一次进度回调progressBlock，第一个参数传递已经拼接的imageData的长度
```
3.`connectionDidFinishLoading:`

执行完这个方法之后，代理不会再接收任何connection发送的消息，意味着下载完成。通常情况下，下载任务正常结束之后，就会执行一次这个方法。

```objectivec
@synchronized(self) {
	停止当前的RunLoop，将connection属性和thread属性置空，发送下载停止的通知。
}
检查sharedURLCache是否缓存了这次下载response，如果没有就将responseFromCached设置为NO
执行完成回调completionBlock，并根据是否读取了缓存、图片尺寸是否为(0,0)等条件向完成回调传递不同的值。
将完成状态、执行状态的标记复位、将属性置空
```
4.`connection: didFailWithError:`

执行完这个方法之后，代理不会再接收任何connection发送的消息，意味着下载失败。通常情况下，下载任务非正常结束，就会执行一次这个方法。

```objectivec
@synchronized(self) {
        停止当前的RunLoop，将connection属性和thread属性置空，发送下载停止的通知。
}

if (self.completedBlock) { // 只使用这一种参数传递的方式完成回调
    self.completedBlock(nil, nil, error, YES);
}
将完成状态、执行状态的标记复位、将属性置空
```
5.`connection: willCacheResponse:`

缓存response之前调用一次这个方法，给connection的代理一次机会改变它。可以返回一个修改之后的response，或者返回nil不存储缓存。

`SDWebImageDownloaderOperation`在这个方法内部完成了以下工作：

```objectivec
responseFromCached = NO; // 标记这次下载的图片不是从缓存中读取出来的
if (self.request.cachePolicy == NSURLRequestReloadIgnoringLocalCacheData) { 
    return nil;  // 如果request的缓存策略(实际上Downloader在使用操作进行下载的时候，会根据下载选项修改request的缓存策略)是忽略本地缓存，不进行不进行缓存
} else {
    return cachedResponse; // 其他情况，正常进行缓存
}
```

6.`connectionShouldUseCredentialStorage: `

这个代理方法的返回值决定URL加载器是否需要使用存储的凭据对网络进行授权验证。
`SDWebImageDownloaderOperation`中这样实现：

```objectivec
- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection __unused *)connection {
    return self.shouldUseCredentialStorage; // shouldUseCredentialStorage属性的init方法中赋初值YES，提供了对外的setter，可以在外部修改这个值。
}
```
7.`connection: willSendRequestForAuthenticationChallenge:`

服务端发起一个质询，需要在这个方法中解决。`SDWebImageDownloaderOperation`对这个方法的实现比较复杂:

```objectivec
if (服务端要求的认证方式是信任认证) {
	如果下载选项没有设置允许无效的SSL证书这个下载选项，那么按照默认的方式处理质询
	其他情况，就直接使用服务端发过来的凭据继续访问
} else {
	如果这个质询之前没有授权失败过且self.credential存在(也就是想操作赋值了一个本地的凭据)，使用self.credential作为凭据处理质询

	其他情况直接使用没有凭据的方式处理质询。
}
```
以上就是所有的`SDWebImageDownloaderOperation`内部实现的NSURLConnection的代理方法，这些方法已经能够很好地完成网络访问、图片下载和数据处理。
`SDWebImageDownloaderOperation`中还定义了一些取消操作的方法，用于暂停下载任务，这些方法比较简单，这里不再一一赘述。
在上面的数据处理全部的过程中，我们发现时刻都在使用者`下载选项`，可见熟悉每个下载选项和使用时机的重要性。接下来看一下负责管理下载操作的`SDWebImageDownloader`类，同时下载选项枚举也是在这个类的头文件中声明的。
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor2_0">回到顶部</a></div>

### 下载管理SDWebImageDownloader
如果说`SDWebImageDownloaderOperation`实现了下载图片的细节，那么`SDWebImageDownloader`就负责控制operation来触发下载任务，并管理所有的下载任务，包括改变他们的状态，`SDWebImageDownloader`是进行下载控制的接口，在实际应用中，我们几乎很少直接使用`SDWebImageDownloaderOperation`，而几乎都是使用`SDWebImageDownloader`来进行下载任务。

`SDWebImageDownloader`有一个重要的属性executionOrder代表着下载操作执行的顺序，它是一个SDWebImageDownloaderExecutionOrder枚举类型：

```objectivec
typedef NS_ENUM(NSInteger, SDWebImageDownloaderExecutionOrder) {
    // 默认值，所有的下载操作以队列类型 (先进先出)执行.
    SDWebImageDownloaderFIFOExecutionOrder,

    // 所有的下载操作以栈类型 (后进先出)执行.
    SDWebImageDownloaderLIFOExecutionOrder
};
```
默认是`SDWebImageDownloaderFIFOExecutionOrder`，是在init方法中设置的。如果设置了后进先出，在下载操作添加到下载队列中时，会依据这个值添加依赖关系，使得最后添加操作出在依赖关系链条中的第一项，因而会优先下载最后添加的操作任务。

`SDWebImageDownloader`还提供了其他几个重要的对外接口(包括属性和方法):

1.`BOOL shouldDecompressImages`<br/>
是否需要解压，在init中设置默认值为YES，在下载操作创建之后将值传递给操作的同名属性。
解压下载或缓存的图片可以提升性能，但是会消耗很多内存
默认是YES，如果你会遇到因为过高的内存消耗引起的崩溃将它设置为NO。

2.`NSInteger maxConcurrentDownloads`<br/>
放到下载队列中的下载操作的总数，是一个瞬间值，因为下载操作一旦执行完成，就会从队列中移除。

3.`NSUInteger currentDownloadCount`<br/>
下载操作的超时时长默认是15.0，即request的超时时长，若设置为0，在创建request的时候依然使用15.0。
只读。

4.`NSURLCredential *urlCredential`<br/>
为request操作设置默认的URL凭据，具体实施为:在将操作添加到队列之前，将操作的credential属性值设置为urlCredential

5.`NSString *username`和`NSString *passwords`<br/>
如果设置了用户名和密码:在将操作添加到队列之前，会将操作的credential属性值设置为`[NSURLCredential credentialWithUser:wself.username password:wself.password persistence:NSURLCredentialPersistenceForSession]`，而忽略了属性值urlCredential。、

6.`- (void)setValue:(NSString *)value forHTTPHeaderField:(NSString *)field;`<br/>
 为HTTP header设置value，用来追加到每个下载对应的HTTP request, 若传递的value为nil，则将对应的field移除。
 扩展里面定义了一个HTTPHeaders属性(NSMutableDictionary类型)用来存储所有设置好的header和对应value。
 在创建request之后紧接着会将HTTPHeaders赋给request，request.allHTTPHeaderFields = self.HTTPHeaders;

7.`- (NSString *)valueForHTTPHeaderField:(NSString *)field;`<br/>
 返回指定的HTTP header field对应的value

8.`SDWebImageDownloaderHeadersFilterBlock headersFilter`<br/>
设置一个过滤器，为下载图片的HTTP request选取header.意味着最终使用的headers是经过这个block过滤之后的返回值。

9.`- (void)setOperationClass:(Class)operationClass;`<br/>
设置一个`SDWebImageDownloaderOperation`的子类 ，在每次 SDWebImage 构建一个下载图片的请求操作的时候作为默认的`NSOperation`使用.
参数operationClass为要设置的默认下载操作的`SDWebImageDownloaderOperation`的子类。 传递 `nil` 会恢复为 `SDWebImageDownloaderOperation`。

以下两个方法是下载控制方法了

`- (id <SDWebImageOperation>)downloadImageWithURL: options: progress: completed:`
这个方法用指定的URL创建一个异步下载实例。
有关completedBlock回调的一些解释：下载完成的时候block会调用一次.
没有使用SDWebImageDownloaderProgressiveDownload选项的情况下，如果下载成功会设置image参数,如果出错，会根据错误设置error参数. 最后一个参数总是YES. 如果使用了SDWebImageDownloaderProgressiveDownload选项，这个block会使用部分image的对象有间隔地重复调用，同时finished参数设置为NO，直到使用完整的image对象和值为YES的finished参数进行最后一次调用.如果出错,finished参数总是YES.

`- (void)setSuspended:(BOOL)suspended;`
设置下载队列的挂起(暂停)状态。若为YES，队列不再开启新的下载操作，再向队列里面添加的操作也不会被开启，但是正在执行的操作依然继续执行。

下面我们就来看一下下载方法的实现细节：

```objectivec
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url options:(SDWebImageDownloaderOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageDownloaderCompletedBlock)completedBlock {
	__block SDWebImageDownloaderOperation *operation;
    __weak __typeof(self) wself = self;

    [self addProgressCallback:progressBlock completedBlock:completedBlock forURL:url createCallback:^{
    	// 创建下载的回调
    }];

    return operation;
}
```
重点就是`addProgressCallback: completedBlock: forURL: createCallback:`的执行了，`SDWebImageDownloader`将外部传来的进度回调、完成回调、url直接传递给这个方法，并实现创建下载操作的代码块作为这个方法的createCallback参数值。下面就看一下这个方法的实现细节：

```objectivec
- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {
	// 对URL判空，如果为空，直接执行完成回调。
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return;
    }
    /*
    对dispatch_barrier_sync函数的解释：
     向分配队列提交一个同步执行的barrier block。与dispatch_barrier_async不同，这个函数直到barrier block执行完毕才会返回，在当前队列调用这个函数会导致死锁。当barrier block被放进一个私有的并行队列后，它不会被立刻执行。实际为，队列会等待直到当前正在执行的blocks执行完毕。到那个时刻，队列才会自己执行barrier block。而任何放到 barrier block之后的block直到 barrier block执行完毕才会执行。
     传递的队列参数应该是你自己用dispatch_queue_create函数创建的一个并行队列。如果你传递一个串行队列或者全局并行队列，这个函数的行为和 dispatch_sync相同。
     与dispatch_barrier_async不同，它不会对目标队列进行强引用(retain操作)。因为调用这个方法是同步的，它“借用”了调用者的引用。而且，没有对block进行Block_copy操作。
     作为对其优化，这个函数会在可能的情况下在当前线程唤起barrier block。
     */
    
    // 为确保不会死锁，当前队列是另一个队列，而不能是self.barrierQueue。
    dispatch_barrier_sync(self.barrierQueue, ^{
        BOOL first = NO;
        if (!self.URLCallbacks[url]) {
            self.URLCallbacks[url] = [NSMutableArray new];
            first = YES;
        }
        /*
        URLCallbacks字典类型key为NSURL类型，value为NSMutableArray类型，value只包含着一个元素，这个元素是一个NSMutableDictionary类型，它的key为NSString代表着回调类型，value为block，是对应的回调
        */
        // 同一时刻对相同url的多个下载请求只进行一次下载
        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;

        if (first) {
            createCallback();
            /* 解释
            若url第一次绑定它的回调，也就是第一次使用这个url创建下载任务，则执行一次创建回调。
            在创建回调中创建下载操作，dispatch_barrier_sync执行确保同一时间只有一个线程操作URLCallbacks属性，也就是确保了下面创建过程中在给operation传递回调的时候能取到正确的self.URLCallbacks[url]值。同时保证后面有相同的url再次创建时，if (!self.URLCallbacks[url])分支不再进入，first==NO,也就不再继续调用创建回调。这样就确保了同一个url对应的图片不会被重复下载。

            而下载器的完成回调中会将url从self.URLCallbacks中remove，虽然remove掉了，但是再次使用这个url进行下载图片的时候,Manager会向缓存中读取下载成功的图片了，而不是无脑地直接添加下载任务；即使之前的下载是失败的(也就是说没有缓存)，这样继续添加下载任务也是合情合理的。
            // 因此准确地说，将这个block放到并行队列dispatch_barrier_sync执行确保了，同一个url的图片不会同一时刻进行多次下载.
            
            // 这样做还使得下载操作的创建同步进行，因为一个新的下载操作还没有创建完成，self.barrierQueue会继续等待它完成，然后才能执行下一个添加下载任务的block。所以说SD添加下载任务是同步的，而且都是在self.barrierQueue这个并行队列中，同步添加任务。这样也保证了根据executionOrder设置依赖关是正确的。换句话说如果创建下载任务不是使用dispatch_barrier_sync完成的，而是使用异步方法 ，虽然依次添加创建下载操作A、B、C的任务，但实际创建顺序可能为A、C、B，这样当executionOrder的值是SDWebImageDownloaderLIFOExecutionOrder，设置的操作依赖关系就变成了A依赖C，C依赖B
            // 但是添加之后的下载依然是在下载队列downloadQueue中异步执行，丝毫不会影响到下载效率。

            // 以上就是说了SD下载的关键点：创建下载任务在barrierQueue队列中，执行下载在downloadQueue队列中。
            */
        }
    });
}
```
说完这些，我们再看一下SD如何给`addProgressCallback: completedBlock: forURL: createCallback:`方法设置创建回调的，毕竟这个才是创建下载操作并放入队列的一些细节：

```objectivec
[self addProgressCallback:progressBlock completedBlock:completedBlock forURL:url createCallback:^{
    NSTimeInterval timeoutInterval = wself.downloadTimeout;
    if (timeoutInterval == 0.0) {
        timeoutInterval = 15.0;
    }
    // 创建请求对象，并根据options参数设置其属性
    // 为了避免潜在的重复缓存(NSURLCache + SDImageCache)，如果没有明确告知需要缓存，则禁用图片请求的缓存操作, 这样就只有SDImageCache进行了缓存
    NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];
    
    request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
    request.HTTPShouldUsePipelining = YES;
    
    if (wself.headersFilter) {
        request.allHTTPHeaderFields = wself.headersFilter(url, [wself.HTTPHeaders copy]);
    }
    else {
        request.allHTTPHeaderFields = wself.HTTPHeaders;
    }
    // 创建SDWebImageDownloaderOperation操作对象，传入进度回调、完成回调、取消回调
    operation = [[wself.operationClass alloc] initWithRequest:request
                                                      options:options
                                                     progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                         // 从callbacksForURL中取出进度回调
                                                         SDWebImageDownloader *sself = wself;
                                                         if (!sself) return;
                                                         __block NSArray *callbacksForURL;
                                                         dispatch_sync(sself.barrierQueue, ^{
                                                             callbacksForURL = [sself.URLCallbacks[url] copy];
                                                         });
                                                         for (NSDictionary *callbacks in callbacksForURL) {
                                                             dispatch_async(dispatch_get_main_queue(), ^{ // 切换到主队列完成异步回调
                                                                 SDWebImageDownloaderProgressBlock callback = callbacks[kProgressCallbackKey];
                                                                 if (callback) callback(receivedSize, expectedSize);
                                                             });
                                                         }
                                                     }
                                                    completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
                                                        // 从callbacksForURL中取出完成回调。
                                                        // 将删除所有回调的block放到队列barrierQueue中使用barrier_sync方式执行，确保了在进行调用完成回调之前所有的使用url对应的回调的地方都是正确的数据。
                                                        SDWebImageDownloader *sself = wself;
                                                        if (!sself) return;
                                                        __block NSArray *callbacksForURL;
                                                        dispatch_barrier_sync(sself.barrierQueue, ^{
                                                            callbacksForURL = [sself.URLCallbacks[url] copy];
                                                            if (finished) {
                                                                [sself.URLCallbacks removeObjectForKey:url];
                                                            }
                                                        });
                                                        for (NSDictionary *callbacks in callbacksForURL) {
                                                            SDWebImageDownloaderCompletedBlock callback = callbacks[kCompletedCallbackKey];
                                                            if (callback) callback(image, data, error, finished);
                                                        }
                                                    }
                                                    cancelled:^{
                                                        // 将url对应的所有回调移除
                                                        SDWebImageDownloader *sself = wself;
                                                        if (!sself) return;
                                                        dispatch_barrier_async(sself.barrierQueue, ^{
                                                            [sself.URLCallbacks removeObjectForKey:url];
                                                        });
                                                    }];
    // 设置是否需要解压
    operation.shouldDecompressImages = wself.shouldDecompressImages;
    // 设置进行网络访问验证的凭据
    if (wself.urlCredential) {
        operation.credential = wself.urlCredential;
    } else if (wself.username && wself.password) {
        operation.credential = [NSURLCredential credentialWithUser:wself.username password:wself.password persistence:NSURLCredentialPersistenceForSession];
    }
    // 根据下载选项SDWebImageDownloaderHighPriority设置优先级
    if (options & SDWebImageDownloaderHighPriority) {
        operation.queuePriority = NSOperationQueuePriorityHigh;
    } else if (options & SDWebImageDownloaderLowPriority) {
        operation.queuePriority = NSOperationQueuePriorityLow;
    }
    // 将操作添加到队列中
    [wself.downloadQueue addOperation:operation];
    // 根据executionOrder设置操作的依赖关系
    if (wself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
        [wself.lastAddedOperation addDependency:operation];
        wself.lastAddedOperation = operation;
    }
}];
```
有关NSOperation的优先级还有一个小细节：

`queuePriority` <br/>
这个属性包含着操作的相对优先级。这个值会影响到操作出队列和执行的顺序，这个值总是符合一个系统预定义的常量。如果没有明确设置优先级，则使用默认值NSOperationQueuePriorityNormal。<br/>
当且仅当需要对没有依赖关系的操作之间设置优先级时候使用它。优先级值不应该用来实现对不同的操作对象之间的依赖管理。如果你想在操作之间建立依赖关系，应该使用addDependency:方法。

如果你尝试指定一个和定义好的常量不同的优先级值,操作对象自动调整你指定的值以适应NSOperationQueuePriorityNormal优先级，直到找到有效的常量值。例如，如果你指定了这个值为-10，操作会调整这个值来匹配NSOperationQueuePriorityVeryLow；相似的，如果你指定了这个值为+10，操作会调整这个值来匹配NSOperationQueuePriorityVeryHigh常量。

另外，我们可以观察到如果没有给operationClass传递值的情况下，SDWebImageDownloader的`- (id <SDWebImageOperation>)downloadImageWithURL: options: progress: completed:`方法实际上返回的是SDWebImageDownloaderOperation类型实例，并且是已经经过各种下载选项设置之后的放入到下载队列中的操作实例。<br/>
我们还需要关注一下方法`- (void)setSuspended:(BOOL)suspended;`，这个方法的实现只有一句：

```objectivec
- (void)setSuspended:(BOOL)suspended { 
    [self.downloadQueue setSuspended:suspended]; // 实际上是对下载队列调用了setSuspended方法
}
```
有关NSOperationQueue对象的setSuspended，不得不看一下文档的一些解释：

>当这个属性值是NO,队列主动开启在队列中的操作，并准备执行。将这个属性设置为YES，阻止队列开启任何队列式的操作，但已经开始且正在执行的操作会继续执行。你可以继续向暂停的队列添加操作，但是如果不改变这个属性为NO，这些操作不会计划执行。

>Operation当且仅当执行完成之后才从队列中移除。但是，为了结束执行，操作必须得先开启执行。因为一个暂停的队里不能开启任何一个新的操作，它不会移除任何一个在当前队列中且不是正在执行的操作(包括已经取消的操作)。
>你可以通过KVO监控这个属性值的改变。配置一个观察者来监控操作队列的suspended key path。
>这个属性的默认值是NO。

可见setSuspended方法传递YES，并不能暂停队列中的所有操作，而是让队列不再开启新的任务。

以上就是关于SD下载图片的全部内容。
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor3_0">回到顶部</a></div>

### 缓存SDImageCache
`SDImageCache`类是一个功能无比强大的缓存管理器。它可以实现内存和磁盘缓存功能的实现和管理，主要包括以下几个方面:<br/>
1.对内存或磁盘缓存进行单个图片增、删、查等操作<br/>
2.还提供使用命名空间的方式对图片分类管理，管理应用启动前的放入app中的预缓存图<br/>
3.同时还可以对所有的缓存整体操作，如查询总缓存文件个数，查询总缓存大小，一次性清理内存缓存，一次性清理磁盘缓存。<br/>
而且刚才所说的所有功能实现之后可以添加完成回调，以便在主线程更新UI或者给出提示信息。

`SDImageCache`的主要属性有以下几个：<br/>
1.`BOOL shouldDecompressImages`<br/>
是否进行解压<br/>
2.`BOOL shouldDisableiCloud`<br/>
不启用iCloud备份 默认是YES<br/>
3.`BOOL shouldCacheImagesInMemory`<br/>
使用内存缓存 默认是YES<br/>
4.`NSUInteger maxMemoryCost`<br/>
内存缓存NSCache能够承受的最大总开销，超过这个值NSCache会剔除对象。是内存缓存(NSCache类型)的属性值。<br/>
5.`NSUInteger maxMemoryCountLimit`<br/>
内存缓存NSCache能承受的最多对象个数<br/>
6.`NSInteger maxCacheAge`<br/>
最大缓存时长 以秒为单位， 默认值为kDefaultCacheMaxCacheAge，一周时间<br/>
7.`NSUInteger maxCacheSize`<br/>
最大缓存大小 以字节为单位。默认没有设置，也就是为0，而清理磁盘缓存的先决条件为self.maxCacheSize > 0，所以0表示无限制。<br/>
在看看它的主要的方法，这里将它们分为几个组分别说明：

有关命名空间，SD会根据命名空间，对内存缓存创建不同的NSCache对象并对name属性赋值，创建不同的磁盘缓存写文件队列等等。

```objectivec
/**
 * 用指定的命名空间初始化一个新的缓存仓库
 */
- (id)initWithNamespace:(NSString *)ns;

/**
 * 用指定的命名空间和目录初始化一个新的缓存仓库
 */
- (id)initWithNamespace:(NSString *)ns diskCacheDirectory:(NSString *)directory;

- (NSString *)makeDiskCachePath:(NSString*)fullNamespace; // 获取指定ns的完整路径,这里传递的是完整ns

/**
通过SDImageCache添加一个只读的缓存文件夹路径，用来搜索预缓存的图片
 如果你想在你的app中捆绑预加载图片，就非常有用。
 */
- (void)addReadOnlyCachePath:(NSString *)path; 
```
什么是完整的ns路径，按照SD的规则(可以在`initWithNamespace: diskCacheDirectory:`方法中查看)，fullNameSpace是在ns前面添加了前缀`com.hackemist.SDWebImageCache.`内存缓存的memCache.name直接设置为fullNamespace，若传入的ns为&#64;"xyz"磁盘缓存的的路径则变为Library/Caches/xyz/com.hackemist.SDWebImageCache.xyz。

这里还有两个用于查询指定key图片在磁盘缓存中的路径的方法

```objectivec
// 获取指定key的缓存路径，需要传入root文件夹
- (NSString *)cachePathForKey:(NSString *)key inPath:(NSString *)path;

// 获取指定key的默认文件路径，也就是root文件夹使用self.diskCachePath
- (NSString *)defaultCachePathForKey:(NSString *)key;
```
现在介绍一些对单个图片的缓存操作的方法：

增：

```objectivec
// 将指定的image缓存起来，key一般传入urlString，默认进行磁盘缓存,实际实现为下面方法toDisk传入YES
- (void)storeImage:(UIImage *)image forKey:(NSString *)key; 

// toDisk指定为是否进行磁盘缓存，实际实现为调用了下面的方法
- (void)storeImage:(UIImage *)image forKey:(NSString *)key toDisk:(BOOL)toDisk;

/**
 * 将image做内存缓存，磁盘缓存为可选
 
 * @param recalculate BOOL 代表着是否imageData可以使用或者一个新的data会根据UIImage构建
 * @param imageData   imageData作为服务器返回的数据, 这个值会用来做磁盘存储来代替将给定的image转换成可存储的/压缩的图片格式的方案，以便节约性能和CPU。(实际上是节约了计算能力，而多使用了一点磁盘的存储能力)
 内存缓存都是使用image
 image和imageData都非空 若recalculate为YES会忽略imageData，而使用image进行磁盘缓存
 两者有一个为空的，使用非空的进行磁盘缓存
 两者都为空，则没有磁盘缓存。
 */
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk;
```
查：

```objectivec

- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock; // 异步查询磁盘缓存，其实内部实现先查询了内存缓存，然后查询磁盘缓存,返回的是一个空的操作(稍后会解释为什么是空的操作)


- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key; // 异步查询内存缓存

- (UIImage *)imageFromDiskCacheForKey:(NSString *)key; // 查询完内存缓存之后再异步查询磁盘缓存
```
删：

```objectivec
// 异步地移除内存和磁盘缓存,实际是下面方法的withCompletion传入nil
- (void)removeImageForKey:(NSString *)key;

// 异步地移除内存和磁盘缓存,带完成回调
- (void)removeImageForKey:(NSString *)key withCompletion:(SDWebImageNoParamsBlock)completion; 

// 异步地移除内存缓存，可以选择是否移除磁盘缓存,实际是下面方法的withCompletion传入nil
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk;

// 异步地移除内存缓存，可以选择是否移除磁盘缓存，完成之后执行回调
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(SDWebImageNoParamsBlock)completion;
```
对于所有的缓存内容的整体操作，有如下一些方法:

删：

```objectivec
// 清除所有的内存缓存
- (void)clearMemory;

// 清除所有的磁盘缓存，无阻塞的方法，立刻返回.
- (void)clearDiskOnCompletion:(SDWebImageNoParamsBlock)completion;

// 上面的方法回调传入nil
- (void)clearDisk;

// 移除磁盘中所有的过期缓存。无阻塞的方法
// clean和clear的区别是：clear是全部移除，clean只清除过期的缓存
- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock;

// 上面的方法回调传入nil
- (void)cleanDisk;
```
查：

```objectivec
// 获取磁盘缓存的大小
- (NSUInteger)getSize; // 是在当前线程的串行队列中同步执行的，思路是遍历目录中的所有文件，累加大小

// 获取磁盘缓存文件的个数
- (NSUInteger)getDiskCount;

// 异步计算磁盘缓存的大小，然后执行回调，回调参数为文件个数和总大小
- (void)calculateSizeWithCompletionBlock:(SDWebImageCalculateSizeBlock)completionBlock;

// 异步检查图片是否在磁盘缓存中存在(没有加载图片)
- (void)diskImageExistsWithKey:(NSString *)key completion:(SDWebImageCheckCacheCompletionBlock)completionBlock;

// 上面方法回调参数传递nil
- (BOOL)diskImageExistsWithKey:(NSString *)key;
```
`SDImageCache`中的方法实现都比较简单：

内存缓存<br/>
添加都是对memCache属性添加元素，key为urlString<br/>
删除都是对memCache属性移除元素<br/>
查询都是按key取元素。<br/>

磁盘缓存<br/>
添加都是将UIImage的二进制写入文件，并以url的MD5为文件名(下面会具体分析)<br/>
删除都是将缓存文件删除<br/>
查询都是读取文件的二进制转为UIImage

对磁盘缓存整体的操作则是遍历文件夹进行对单个文件操作来实现，在执行清理操作的时候，会一一对比缓存文件的上次修改(存储)的时间到当前时间是否超过了过期时长，进行删除操作(下面会具体分析)。而对于readonly的预先缓存好的图片所在的路径会存储在私有属性customPaths中，查询图片的时候也会遍历这个属性中所有的文件夹。

另外内存缓存的memCache是自定义的NSCache子类AutoPurgeCache，会接收内存警告的通知，当收到通知，会调用removeAllObjects方法清除所有的内存缓存。

对于实现一张图片缓存的具体实现：

```objectivec
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk {
    if (!image || !key) {
        return;
    }
    // 内存缓存 前提是设置了需要进行
    if (self.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(image);
        [self.memCache setObject:image forKey:key cost:cost];
    }
    // 磁盘缓存
    if (toDisk) {
    	// 将缓存操作作为一个任务放入ioQueue中异步执行
        dispatch_async(self.ioQueue, ^{
            NSData *data = imageData;

            if (image && (recalculate || !data)) {
#if TARGET_OS_IPHONE
				// 需要确定图片是PNG还是JPEG。PNG图片容易检测，因为有一个唯一签名。PNG图像的前8个字节总是包含以下值：137 80 78 71 13 10 26 10
                // 在imageData为nil的情况下假定图像为PNG。我们将其当作PNG以避免丢失透明度。
                int alphaInfo = CGImageGetAlphaInfo(image.CGImage);
                BOOL hasAlpha = !(alphaInfo == kCGImageAlphaNone ||
                                  alphaInfo == kCGImageAlphaNoneSkipFirst ||
                                  alphaInfo == kCGImageAlphaNoneSkipLast);
                BOOL imageIsPng = hasAlpha;
                // 而当有图片数据时，我们检测其前缀，确定图片的类型
                if ([imageData length] >= [kPNGSignatureData length]) {
                    imageIsPng = ImageDataHasPNGPreffix(imageData);
                }

                if (imageIsPng) {
                    data = UIImagePNGRepresentation(image);
                }
                else {
                    data = UIImageJPEGRepresentation(image, (CGFloat)1.0);
                }
#else
                data = [NSBitmapImageRep representationOfImageRepsInArray:image.representations usingType: NSJPEGFileType properties:nil];
#endif
            }

            if (data) {
                if (![_fileManager fileExistsAtPath:_diskCachePath]) {
                    [_fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
                }

                // 根据image的key获取缓存路径
                NSString *cachePathForKey = [self defaultCachePathForKey:key];
                NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];

                [_fileManager createFileAtPath:cachePathForKey contents:data attributes:nil];

                // 不适用iCloud备份
                if (self.shouldDisableiCloud) {
                    [fileURL setResourceValue:[NSNumber numberWithBool:YES] forKey:NSURLIsExcludedFromBackupKey error:nil];
                }
            }
        });
    }
}
```
对于清理方法`cleanDiskWithCompletionBlock:`，有两个指标：文件的缓存有效期及最大缓存空间大小。文件的缓存有效期可以通过maxCacheAge属性来设置，默认是1周的时间。如果文件的缓存时间超过这个时间值，则将其移除。而最大缓存空间大小是通过maxCacheSize属性来设置的，如果所有缓存文件的总大小超过这一大小，则会按照文件最后修改时间的逆序，以每次一半的递归来移除那些过早的文件，直到缓存的实际大小小于我们设置的最大使用空间。清理的操作在-cleanDiskWithCompletionBlock:方法中，其实现如下：

```objectivec
- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock {
    dispatch_async(self.ioQueue, ^{
        NSURL *diskCacheURL = [NSURL fileURLWithPath:self.diskCachePath isDirectory:YES];
        NSArray *resourceKeys = @[NSURLIsDirectoryKey, NSURLContentModificationDateKey, NSURLTotalFileAllocatedSizeKey];

        // 枚举器预先获取缓存文件的有用的属性
        NSDirectoryEnumerator *fileEnumerator = [_fileManager enumeratorAtURL:diskCacheURL
                                                   includingPropertiesForKeys:resourceKeys
                                                                      options:NSDirectoryEnumerationSkipsHiddenFiles
                                                                 errorHandler:NULL];

        NSDate *expirationDate = [NSDate dateWithTimeIntervalSinceNow:-self.maxCacheAge];
        NSMutableDictionary *cacheFiles = [NSMutableDictionary dictionary];
        NSUInteger currentCacheSize = 0;

        // 枚举缓存文件夹中所有文件，该迭代有两个目的：移除比过期日期更老的文件；存储文件属性以备后面执行基于缓存大小的清理操作
        NSMutableArray *urlsToDelete = [[NSMutableArray alloc] init];
        for (NSURL *fileURL in fileEnumerator) {
            NSDictionary *resourceValues = [fileURL resourceValuesForKeys:resourceKeys error:NULL];

            if ([resourceValues[NSURLIsDirectoryKey] boolValue]) {
                continue;
            }

            // 移除早于有效期的老文件
            NSDate *modificationDate = resourceValues[NSURLContentModificationDateKey];
            if ([[modificationDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
                [urlsToDelete addObject:fileURL];
                continue;
            }

            // 存储文件的引用并计算所有文件的总大小
            NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
            currentCacheSize += [totalAllocatedSize unsignedIntegerValue];
            [cacheFiles setObject:resourceValues forKey:fileURL];
        }
        
        for (NSURL *fileURL in urlsToDelete) {
            [_fileManager removeItemAtURL:fileURL error:nil];
        }

        // 如果磁盘缓存的大小超过我们配置的最大大小，则执行基于文件大小的清理，我们首先删除最老的文件
        if (self.maxCacheSize > 0 && currentCacheSize > self.maxCacheSize) {
            // 以设置的最大缓存大小的一半值作为清理目标
            const NSUInteger desiredCacheSize = self.maxCacheSize / 2;

            // 按照最后修改时间来排序剩下的缓存文件
            NSArray *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                            usingComparator:^NSComparisonResult(id obj1, id obj2) {
                                                                return [obj1[NSURLContentModificationDateKey] compare:obj2[NSURLContentModificationDateKey]];
                                                            }];

            // 删除文件，直到缓存总大小降到我们期望的大小
            for (NSURL *fileURL in sortedFiles) {
                if ([_fileManager removeItemAtURL:fileURL error:nil]) {
                    NSDictionary *resourceValues = cacheFiles[fileURL];
                    NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
                    currentCacheSize -= [totalAllocatedSize unsignedIntegerValue];

                    if (currentCacheSize < desiredCacheSize) {
                        break;
                    }
                }
            }
        }
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}
```
我们看一下刚才遗留的一个问题，为什么使用`- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock`方法查询指定key的缓存时，返回的是一个空的NSOperation，我们先看一下这个方法的实现：

```objectivec
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock {
    // 对doneBlock、key判空 查找内存缓存
    // ...

    // 查找内存缓存
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        doneBlock(image, SDImageCacheTypeMemory);
        return nil;
    }

    NSOperation *operation = [NSOperation new];
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) { // isCancelled初始默认值为NO
            return;
        }

        @autoreleasepool {
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage && self.shouldCacheImagesInMemory) {
                NSUInteger cost = SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });

    return operation;
}
```
通过代码可以看到operation虽然没有具体的内容，但是我们可以在外部调用operation的cancel方法来改变isCancelled的值。这样做对从内存缓存中查找到图片的本次操作查询过程没有影响，但是如果本次查询过程是在磁盘缓存中进行的，就会受到影响，autoreleasepool{}代码块不再执行。而在这段代码块完成了这样的工作：将磁盘缓存取出进行内存缓存，在线程执行完成回调。因此可以看到这个返回的NSOpeation值可以帮助我们在外部控制不再进行磁盘缓存查询和内存缓存备份的操作，归根结底就是向外部暴漏了取消操作的接口。
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor4_0">回到顶部</a></div>

### SDWebImageManager：按需下载->完成缓存->缓存管理等一系列完整的流程线
在实际的运用中，我们并不直接使用SDWebImageDownloader类及SDImageCache类来执行图片的下载及缓存。为了方便用户的使用，SDWebImage提供了SDWebImageManager对象来管理图片的下载与缓存。而且我们经常用到的诸如UIImageView+WebCache等控件的分类都是基于SDWebImageManager对象的。该对象将一个下载器和一个图片缓存绑定在一起，并对外提供两个只读属性来获取它们，如下代码所示：

```objectivec
@interface SDWebImageManager : NSObject

@property (weak, nonatomic) id <SDWebImageManagerDelegate> delegate;

@property (strong, nonatomic, readonly) SDImageCache *imageCache;
@property (strong, nonatomic, readonly) SDWebImageDownloader *imageDownloader;

@property (nonatomic, copy) SDWebImageCacheKeyFilterBlock cacheKeyFilter;
// ...
@end
```
从上面的代码中我们还可以看到有一个delegate属性，其是一个`id<SDWebImageManagerDelegate>`对象。`SDWebImageManagerDelegate`声明了两个可选实现的方法，如下所示：

```objectivec
// 控制当图片在缓存中没有找到时，应该下载哪个图片
- (BOOL)imageManager:(SDWebImageManager *)imageManager shouldDownloadImageForURL:(NSURL *)imageURL;

// 允许在图片已经被下载完成且被缓存到磁盘或内存前立即转换
- (UIImage *)imageManager:(SDWebImageManager *)imageManager transformDownloadedImage:(UIImage *)image withURL:(NSURL *)imageURL;
```
这两个代理方法会在`SDWebImageManager`的`-downloadImageWithURL:options:progress:completed:`方法中调用，而这个方法是SDWebImageManager类的核心所在。我们来看看它的具体实现：

为了能够更好地理解这个方法的实现，再次必须强调一个SDWebImageOptions选项值`SDWebImageRefreshCached`,如果设置了这个值：

即使SD对图片缓存了，也期望HTTP响应cache control，并在需要的情况下从远程刷新图片。也就是说如果在磁盘中找到了这张图片，但设置了这个选项，仍然需要进行网络请求，查看服务器端的这张图片有没有被改变，并决定进行下载，然后使用新的图片，同时完成新的缓存。

但是这个下载并不是自己决定要不要进行的，还需要如果代理通过方法`[self.delegate imageManager:self shouldDownloadImageForURL:url]`返回NO，那就是代理要求这个url对应的图片不需要下载。这种情况下就不再下载，而是使用在缓存中查找到的图片

```objectivec
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock {
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");
    // 判断URL合法性 
    // ...

    __block SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new]; // 创建一个组合操作,主要用于将查询缓存、下载操作、进行缓存等工作联系在一起
    __weak SDWebImageCombinedOperation *weakOperation = operation;
    
    // 检查这个url是否在失败列表中，也就是是否曾经下载失败过。
    // ...

    // 如果没有设置失败重试选项(SDWebImageRetryFailed)，并且是一个失败过的url，则直接执行完成回调。
    // ...

    @synchronized (self.runningOperations) { // (self.runningOperations是一个数组，元素为正在进行的组合操作)
        [self.runningOperations addObject:operation];
    }
    NSString *key = [self cacheKeyForURL:url];

    operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:^(UIImage *image, SDImageCacheType cacheType) {
        // 如果操作被取消了，从正在进行的操作列表中将它移出.
        // ...
        
        // 条件A:在缓存中没有找到图片 或者 options选项包含SDWebImageRefreshCached (这两种情况都需要进行请求网络图片的)
        // 且
        // 条件B:代理允许下载
        /*
         条件B的实现为：代理不能响应imageManager:shouldDownloadImageForURL:方法 或者 能响应且方法返回值为YES。也就是说没有实现这个方法就是允许的，而如果实现了的话，返回为YES才是允许的。
         */
        if ((!image || options & SDWebImageRefreshCached) && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url])) {
            // 分支一：缓存中找到了图片 且 options选项包含SDWebImageRefreshCached， 先在主线程完成一次回调,使用的是缓存中找到的图片
            if (image && options & SDWebImageRefreshCached) {
                dispatch_main_sync_safe(^{
                    // 如果在缓存中找到了image但是设置了SDWebImageRefreshCached选项，传递缓存的image，同时尝试重新下载它来让NSURLCache有机会接收服务器端的更新
                    completedBlock(image, nil, cacheType, YES, url);
                });
            }
            
            // 如果没在缓存中找到image 或者 设置了需要请求服务器刷新的选项，则仍需要下载.
            SDWebImageDownloaderOptions downloaderOptions = 0;
            // ...
            if (image && options & SDWebImageRefreshCached) {
                // 如果image已经被缓存但是设置了需要请求服务器刷新的选项，强制关闭渐进式选项
                downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                // 如果image已经被缓存但是设置了需要请求服务器刷新的选项，忽略从NSURLCache读取的image
                downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
            }
            
            // 创建下载操作，先使用self.imageDownloader下载
            id <SDWebImageOperation> subOperation = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *data, NSError *error, BOOL finished) { // 使用self.imageDownloader进行下载
                __strong __typeof(weakOperation) strongOperation = weakOperation;
                if (!strongOperation || strongOperation.isCancelled) {
                    // 如果操作取消了，不做任何事情
                    //如果调用completedBlock, 这个block会和另一个completedBlock争夺同一个对象。因此，如果这个block后被调用，会覆盖新的数据。
                }
                else if (error) {
                    // 进行完成回调
                    // 将url添加到失败列表中
                    // ...
                }
                else {
                    // 如果设置了失败重试，将url从失败列表中去掉
                    // ...
                    
                    // 设置了SDWebImageRefreshCached选项 且 缓存中找到了image 且 没有下载成功
                    if (options & SDWebImageRefreshCached && image && !downloadedImage) {
                        // 这个分支的进入的条件:既没有error、downloadedImage又是nil，这种回调在SDWebImageDownloaderOperation进行下载的时候只有读取了URL的缓存才会发生，即下载正常完成，但是没有数据。
                        // 图片刷新遇到了NSSURLCache中有缓存的状况，不调用完成回调。
                        // Image refresh hit the NSURLCache cache, do not call the completion block
                    }
                    // 下载成功 且 设置了需要变形Image的选项 且变形的代理方法已经实现
                    else if (downloadedImage && (!downloadedImage.images || (options & SDWebImageTransformAnimatedImage)) && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) {
                        /*
                         全局队列异步执行：
                         1.调用代理方法完成形变
                         2.进行缓存
                         3.主线程执行完成回调
                         */
                         // ...
                    }
                    else {
                        /*
                         1.进行缓存
                         2.主线程执行完成回调
                         */
                        // ...
                    }
                }

                if (finished) {
                    // 从正在进行的操作列表中移除这个组合操作
                    // ...
                }
            }];
            // 设置组合操作的取消回调
            // ...
        }
        // 处理其他情况
        // 情况一：在缓存中找到图片(代理不允许下载 或者 没有设置SDWebImageRefreshCached选项 满足至少一项)
        else if (image) {
            // 使用image执行完成回调
            // 从正在进行的操作列表中移除组合操作
            // ...
        }
        // 情况二：在缓存中没找到图片 且 代理不允许下载
        else {
            // 执行完成回调
            // 从正在进行的操作列表中移除组合操作
            // ...
        }
    }];

    return operation;
}
```
这个方法主要完成了这些工作：<br/>
1.创建一个组合Operation，是一个SDWebImageCombinedOperation对象，这个对象负责对下载operation创建和管理，同时有缓存功能，是对下载和缓存两个过程的组合。<br/>
2.先去寻找这张图片 内存缓存和磁盘缓存，这两个功能在self.imageCache的queryDiskCacheForKey: done:方法中完成，这个方法的返回值既是一个缓存operation，最终被赋给上面的Operation的cacheOperation属性。
在查找缓存的完成回调中的代码是重点：它会根据是否设置了`SDWebImageRefreshCached`选项和代理是否支持下载决定是否要进行下载，并对下载过程中遇到NSURLCache的情况做处理，还有下载失败的处理以及下载之后进行缓存，然后查看是否设置了形变选项并调用代理的形变方法进行对图片形变处理。<br/>
3.将上面的下载方法返回的操作命名为subOperation，并在组合操作operation的cancelBlock代码块中添加对subOperation的cancel方法的调用。这样就完成了下面的工作1和2：

```objectivec
// 1.使能通过组合操作的属性cacheOperation控制缓存操作的取消
operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:^(UIImage *image, SDImageCacheType cacheType) {
	// ...
	// 需要下载的话，进行下面的过程
	id <SDWebImageOperation> subOperation = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *data, NSError *error, BOOL finished) {

	}
	operation.cancelBlock = ^{ // 2.使能通过组合操作的cancelBlock控制下载的取消
        [subOperation cancel];
         // ...
	};

	// 不需要下载的其他情况
	// ...
}
```
4.处理请他的情况：代理不允许下载但是找到缓存的情况，没有找到缓存且代理不允许下载的情况<br/>
5.这个方法最终返回的是operation也就是一个SDWebImageCombinedOperation对象，而不是下载操作。
注意以下区分：

```objectivec
本方法,也就是SDWebImageManager对象的- (id <SDWebImageOperation>)downloadImageWithURL:options:progress:completed:返回的是SDWebImageCombinedOperation对象

SDImageCache对象的- (NSOperation *)queryDiskCacheForKey: done:返回的是一个空的NSOperation对象(用于取消磁盘缓存查询和内存缓存备份)

SDWebImageDownloader对象的- (id <SDWebImageOperation>)downloadImageWithURL:options: progress:completed:返回的是一个已经放到队列中执行的下载操作，默认是SDWebImageDownloaderOperation对象
```
介于几个方法的语义不明  我强烈建议SD做一下修改:

>将SDWebImageOperation协议改名为SDCancellableOperation<br/><br/>
>将SDWebImageManager对象的- (id &lt;SDWebImageOperation&gt;)downloadImageWithURL:options:progress:completed:方法改名为- (id &lt;SDWebImageCombinedOperation&gt;)downloadAndCacheImageWithURL:options:progress:completed:<br/>介于没有对外暴漏SDWebImageCombinedOperation类，改名为- (id &lt;SDCancellableOperation&gt;)downloadAndCacheImageWithURL:options:progress:completed:即可<br/><br/>
>将SDImageCache对象的- (NSOperation *)queryDiskCacheForKey: done:改名为- (id &lt;SDCancellableOperation&gt;)queryDiskCacheForKey: done:，当然这个不是必须的<br/><br/>
>将SDWebImageDownloader对象的- (id &lt;SDWebImageOperation&gt;)downloadImageWithURL:options: progress:completed:改名为<br/>- (id &lt;SDCancellableOperation&gt;)downloadImageWithURL:options: progress:completed:

说了那么半天还没有介绍一项重要内容：上面这个下载方法中的操作选项参数是由枚举SDWebImageOptions来定义的，这个操作中的一些选项是与SDWebImageDownloaderOptions中的选项对应的。我们来看看这个SDWebImageOptions选项都有哪些：

```objectivec
typedef NS_OPTIONS(NSUInteger, SDWebImageOptions) {

    // 默认情况下，当URL下载失败时，URL会被列入黑名单，导致库不会再去重试，该标记用于禁用黑名单
    SDWebImageRetryFailed = 1 << 0,

    // 默认情况下，图片下载开始于UI交互，该标记禁用这一特性，这样下载延迟到UIScrollView减速时
    SDWebImageLowPriority = 1 << 1,

    // 该标记禁用磁盘缓存
    SDWebImageCacheMemoryOnly = 1 << 2,

    // 该标记启用渐进式下载，图片在下载过程中是渐渐显示的，如同浏览器一下。
    // 默认情况下，图像在下载完成后一次性显示
    SDWebImageProgressiveDownload = 1 << 3,

    // 即使图片缓存了，也期望HTTP响应cache control，并在需要的情况下从远程刷新图片。
    // 磁盘缓存将被NSURLCache处理而不是SDWebImage，因为SDWebImage会导致轻微的性能下载。
    // 该标记帮助处理在相同请求URL后面改变的图片。如果缓存图片被刷新，则完成block会使用缓存图片调用一次
    // 然后再用最终图片调用一次
    SDWebImageRefreshCached = 1 << 4,

    // 在iOS 4+系统中，当程序进入后台后继续下载图片。这将要求系统给予额外的时间让请求完成
    // 如果后台任务超时，则操作被取消
    SDWebImageContinueInBackground = 1 << 5,

    // 通过设置NSMutableURLRequest.HTTPShouldHandleCookies = YES;来处理存储在NSHTTPCookieStore中的cookie
    SDWebImageHandleCookies = 1 << 6,

    // 允许不受信任的SSL认证
    SDWebImageAllowInvalidSSLCertificates = 1 << 7,

    // 默认情况下，图片下载按入队的顺序来执行。该标记将其移到队列的前面，
    // 以便图片能立即下载而不是等到当前队列被加载
    SDWebImageHighPriority = 1 << 8,

    // 默认情况下，占位图片在加载图片的同时被加载。该标记延迟占位图片的加载直到图片已以被加载完成
    SDWebImageDelayPlaceholder = 1 << 9,

    // 通常我们不调用动画图片的transformDownloadedImage代理方法，因为大多数转换代码可以管理它。
    // 使用这个票房则不任何情况下都进行转换。
    SDWebImageTransformAnimatedImage = 1 << 10,
};
```
可以看到两个SDWebImageOptions与SDWebImageDownloaderOptions中的选项有一定的对应关系，实际上我们在使用SD时，使用SDWebImageManager的`-downloadImageWithURL:options:progress:completed:`方法较多，而几乎很少单独使用下载和缓存的功能，这个方法的组合功能中会使用设置的SDWebImageOptions值改变相应的SDWebImageDownloaderOptions值，同时也会对缓存方案有一定的一项。

`SDWebImageManager`中还有一个重要的属性：<br/>
决定缓存的key的使用方案的属性`@property (nonatomic, copy) SDWebImageCacheKeyFilterBlock cacheKeyFilter;`<br/>
这是一个block类型的值，会按照它定义的内容对url进行过滤，得到url对应的缓存key。还有一个根据url得到缓存key的方法，其内部就是调用了这个block。

```objectivec
- (NSString *)cacheKeyForURL:(NSURL *)url; // 如果外部没有传入self.cacheFilter 那么返回的是[url absoluteString]
```

`SDWebImageManager`中还有一些控制和查看执行状态的方法：
```objectivec
// 取消runningOperations中所有的操作，并全部删除
- (void)cancelAll;

// 检查是否有操作在运行，这里的操作指的是下载和缓存组成的组合操作,其实就是检查self.runningOperations中的组合操作个数是否大于0
- (BOOL)isRunning;
```

另外要说明`SDWebImageManager`中还定义了与缓存操作相关的方法，其实都是调用了self.imageCache(SDImageCache类型)的相关缓存方法实现的,如：

```objectivec
// 使用self.imageCache的store..方法进行内存和磁盘缓存
- (void)saveImageToCache:(UIImage *)image forURL:(NSURL *)url;

// 指定url的图片是否进行了缓存，优先查看内存缓存，再查看磁盘缓存，只要有就返回YES，两者都没有则返回NO
- (BOOL)cachedImageExistsForURL:(NSURL *)url;

// 指定url的图片是否进行了磁盘缓存
- (BOOL)diskImageExistsForURL:(NSURL *)url;


- (void)cachedImageExistsForURL:(NSURL *)url
                     completion:(SDWebImageCheckCacheCompletionBlock)completionBlock; // 获取指定url的缓存传递给回调，如果是内存缓存，在主队列异步执行回调；如果是磁盘缓存，在当前线程执行回调

- (void)diskImageExistsForURL:(NSURL *)url
                   completion:(SDWebImageCheckCacheCompletionBlock)completionBlock; // 获取指定url的磁盘缓存，只是磁盘缓存，和上面的实现相同，在当前线程执行回调
```
这些方法的具体实现可以查看本文第三部分：<a href="#anchor3_0">缓存SDImageCache</a>
