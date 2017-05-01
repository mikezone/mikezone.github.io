---
layout: cnblog_post
title: NSCache和NSURLCache、网络缓存优化
date: 2016-02-24T13:50:39.000Z
categories: iOS
---
<div><a name="labelTop"></a></div>
<!--Category-->
<div id="navCategory">
	<b>本文目录</b>
	<ul>
		<li><a href="#anchor1_0">一种缓存优化方案</a></li>
        <li><a href="#anchor2_0">响应头'Last-Modified'和请求头'If-Modified-Since'</a></li>
        <li><a href="#anchor3_0">'Keep-Alive'响应头和不离线的URLSession</a></li>
        <li><a href="#anchor4_0">'Expires'响应头</a></li>
        <li><a href="#anchor5_0">这篇文章的意义</a></li>
	</ul>
</div>
<!--Category结束-->

##### 正文开始
首先要说一件重要的事：<br/>
NSCache和NSURLCache一点关系也没有<br/>
NSCache和NSURLCache一点关系也没有<br/>
NSCache和NSURLCache一点关系也没有

然后我推荐大家阅读一下这两篇文章：<br/>
南峰子<a href="http://southpeak.github.io/blog/2015/02/11/cocoa-foundation-nscache/" target='blank'>Foundation:NSCache</a><br/>
mattt<a href="http://nshipster.cn/nsurlcache/" target='blank'>NSURLCache</a>

需要注意的一点是：<br/>
设置NSURLCache的大小时，大多使用下面的代码

```objectivec
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  NSURLCache *URLCache = [[NSURLCache alloc] initWithMemoryCapacity:4 * 1024 * 1024
                                                       diskCapacity:20 * 1024 * 1024
                                                           diskPath:nil];
  [NSURLCache setSharedURLCache:URLCache];
}
```
但是即使没有这两句代码，iOS也会自动参与缓存的，只不过使用的是系统创建的NSURLCache类，同样是可以通过NSURLCache的sharedURLCache方法获取。

在某些情况下，应用中的系统组件会将缓存的内存容量设为0MB，这就禁用了缓存。解决这个行为的一种方式就是通过自己的实现子类化NSURLCache，拒绝将内存缓存大小设为0。如可以使用如下代码进行设置：

```objectivec
@interface MKNonZeroingURLCache : NSURLCache

@end

@implementation MKNonZeroingURLCache

- (void)setMemoryCapacity:(NSUInteger)memoryCapacity {
    if (memoryCapacity == 0) {
        return;
    }
    [super setMemoryCapacity:memoryCapacity];
}

@end

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    MKNonZeroingURLCache *urlCache = [[MKNonZeroingURLCache alloc] initWithMemoryCapacity:4 * 1024 * 1024 diskCapacity:20 * 1024 * 1024 diskPath:nil];
    [NSURLCache setSharedURLCache:urlCache];
    
    return YES;
}
// ...
@end
```
另外，在应用没有运行的状态下，如系统遇到磁盘空间太小的情况，系统也会主动清除一些磁盘缓存的。

说点题外话：`setSharedURLCache:`这个方法的命名和编程行为也是可以学习的。它告诉我们，单例的创建并不都是一成不变的使用sharedXXX方法，也可以使用一个setSharedXXX:传递一个自定义的本类对象，虽然单例对象是外部创建而不是预设的，但是这样创建之后sharedXXX方法依然是获取单例的方法。


本篇文章主要介绍一种网络缓存优化的策略，实际上这个优化方案是提升网络性能的一个小方案。提升网络性能是一个大的课题，它主要包括以下几个方面的改善：

>网络请求性能优化的策略<br/>
>一.减少请求带宽<br/>
>1.请求压缩<br/>
>2.响应压缩<br/>
>二.降低请求延迟<br/>
>如：为NSURLReqeust开启管道支持<br/>
>三.避免网络请求<br/>
>主要是使用缓存优化<br/>

<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor1_0">回到顶部</a></div>

### 一种缓存优化方案
HTTP协议规格说明定义ETag为“被请求变量的实体值”。另一种说法是，ETag是一个可以与Web资源关联的记号（token）。Web资源可以是一个web页面、json或xml数据、文件等。Etag有点类似于文件hash或者说是信息摘要。

在浏览器默认的行为中，当进行一次URL请求，服务端会返回'Etag'响应头，下次浏览器请求相同的URL时，浏览器会自动将它设置为请求头'If-None-Match'的值。服务器收到这个请求之后，就开始做信息校验工作将自己本次产生的Etag与请求传递过来的'If-None-Match'对比，如果相同，则返回HTTP状态码304，并且response数据体中没有数据。

进一步剖析这个过程：第二次请求的时候从哪里获取到'Etag'的值并赋给请求头'If-None-Match'的？自然是浏览器的缓存中取出的。那么浏览器收到304状态码之后又干了什么？刚才说到response数据体中没有数据，但是浏览器仍需加载页面，它会从缓存中读取上次缓存的页面。

上面的浏览器和服务器的配合完成了这样一系列的工作：

```objectivec
if (本地没有缓存) {
	进行第一次请求
} else {本地有缓存
	取出上次response的Etag，作为这次请求的'If-None-Match'值
	进行网络请求
	if (服务器给的HTTP状态码 == 304) {
		// response的数据体为空，减少了一次数据传输
		// 缓存存在的先决条件满足，从缓存中取数据
	} else {
		// 不是304,说明请求的内容改变了，服务器给了新的数据,数据体不空
		// 使用最新的数据
	}
}
```
然而上面说的一大通都只是浏览器的行为，并不是iOS请求的默认行为，对于iOS开发而言，虽然不需要手动地管理缓存，但缓存策略会对上面的行为有影响。
iOS中定以的URLRequest缓存策略有以下几种：

```objectivec
typedef NS_ENUM(NSUInteger, NSURLRequestCachePolicy)
{
    NSURLRequestUseProtocolCachePolicy = 0,

    NSURLRequestReloadIgnoringLocalCacheData = 1, // 从不读取缓存，但请求后将response缓存起来
    NSURLRequestReloadIgnoringLocalAndRemoteCacheData = 4, // Unimplemented
    NSURLRequestReloadIgnoringCacheData = NSURLRequestReloadIgnoringLocalCacheData,

    // 以下两种在取缓存时，可能取到的是过期数据
    NSURLRequestReturnCacheDataElseLoad = 2, // 缓存中没有才去发起请求加载，有就不进行网络请求了
    NSURLRequestReturnCacheDataDontLoad = 3, // 缓存中没有不加载，绝不发起网络请求,缓存中没有则返回错误

    NSURLRequestReloadRevalidatingCacheData = 5, // Unimplemented
};
```
我们着重看一下默认缓存策略`UseProtocolCachePolicy`和忽略缓存的策略`ReloadIgnoringLocalCacheData`,

##### 当使用默认的缓存策略时：
第一次请求一个URL时，会将response和数据缓存起来，
再次请求相同的URL时，会使用缓存中的Etag作为这次请求的request的'If-None-Match'值，这样服务端会返回304并且response的数据体为空，此时iOS会帮助读取缓存中的数据体，修改次请求的response，将HTTP状态码改为200，使用修改后的response和缓存中取到的data作为参数执行完成回调。


以上过程看起来似乎很完美，除了状态码不是304，其他的过程和浏览器几乎一致。但是他有一个缺陷，在研究这个缺陷之前我们先弄清一个这么一个事实：请求内容可以分为三种 1.脚本2.用数据渲染的页面3.静态文件。

对于脚本请求的处理，服务端是会忽略Etag，而每次都会处理，这样返回的数据都是新的，返回HTTP状态码为200.
对于用数据渲染的页面，服务器会按照一定的计算规则，计算渲染之后的Etag，然后对比，再决定返回的是304或者200.
对于静态文件，有些服务器具有检测静态文件改变的能力，一旦文件发生改变，服务器会立刻检测到，从而返回200给客户端，而有些服务器检测文件改变的功能是有延迟的，或者根本没有这种功能，这样即使文件的内容改变了，服务器仍然认为没有改变，于是对比Etag依然相等，结果返回304.（这次测试使用了apache和Express，默认配置下的apache对文件改变的检测是有延迟的，Express则是实时检测的）

根据以上的描述就会暴露出使用默认缓存策略的一点劣势，如果服务器不能实时检测文件改变状态，那么文件是否改变的比对结果是不准确的。最糟糕的情况就是：当文件改变了，服务器认为仍然没有改变，从而返回了304，而没有携带最新的数据。


##### `ReloadIgnoringLocalCacheData`策略时：
每次请求前都会忽略缓存，request的header从来不会附带'If-None-Match'值, 服务器每次处理成功后都是返回200,这样每次都会拿到服务器的数据(每次response的Date头都是新的值)，服务器返回的response带有完整的数据体。iOS接收到数据之后，将response和数据缓存，并作为参数执行完成回调。

这里我们也能够看到使用`ReloadIgnoringLocalCacheData`策略暴漏出来的缺点：尽管服务器端的文件确实没有改变，但iOS依然不使用本地已有的缓存，而每次服务端还要将数据发给客户端，这样是多么浪费带宽！

##### 用这个不好，用这个也不好，到底该如何
我们期望的状态是这样的：
对于服务端，无论怎么做的配置，都希望文件是否改变的检查结果是最准确的。对于iOS客户端，得到状态码200自然不要多做什么处理，如果得到状态码304，则从缓存中取到数据。

于是进行了如下的缓存优化方案：

```objectivec
- (void)refreshedRequest:(NSString *)urlString success:(void (^)(NSHTTPURLResponse *httpResponse, id responseData))successs failure:(void (^)(NSError *error))failure {
    NSMutableURLRequest *urlRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:urlString]];
    
    NSURLCache *urlCache = [NSURLCache sharedURLCache];
    NSCachedURLResponse *cacheURLResponse = [urlCache cachedResponseForRequest:urlRequest];
    if (cacheURLResponse) {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)cacheURLResponse.response;
        NSString *cachedResponseEtag = [httpResponse.allHeaderFields objectForKey:@"Etag"];
        if (cachedResponseEtag) {
            [urlRequest setValue:cachedResponseEtag forHTTPHeaderField:@"If-None-Match"];
        }
    }

    [urlRequest setCachePolicy:NSURLRequestReloadIgnoringCacheData];
    [[[NSURLSession sharedSession] dataTaskWithRequest:urlRequest completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        if (!error && successs) {
            NSHTTPURLResponse *newHttpResponse = (NSHTTPURLResponse *)response;
            if (newHttpResponse.statusCode == 304) {
                // cached in local
                successs(newHttpResponse, cacheURLResponse.data);
            } else {
                // refreshed from server
                successs(newHttpResponse, data);
            }
        } else {
            if (failure) {
                failure(error);
            }
        }
    }] resume];
}
```
这样每次请求都使用忽略缓存的策略，但是要附带着"If-None-Match"头，它的值是上次请求的响应头"Etag"的值，于是服务器会每次都实时检查文件的修改状态，得到一个准确的状态值，最后决定返回304还是200。若是200，iOS则直接使用新的response和新的数据;如果是304，则使用新的response和缓存中的data。
这样既能够获取到最新的数据有能够节约带宽。两全其美。
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor2_0">回到顶部</a></div>

### 不可忽视的响应头'Last-Modified'和请求头'If-Modified-Since'
在上面说的服务端对文件的验证只涉及到ETag，而实际上服务端的验证过程比这个复杂，还需要使用'Last-Modified'值。'Last-Modified'值在服务器处理阶段代表着文件的上次修改时间，在处理结束后作为一个响应头放到response中。如果在请求中添加了'If-Modified-Since'头，并将这个值设置为上次请求时得到的响应头'Last-Modified'的值，那么这次请求，服务器的处理过程如下：

```objectivec
if 计算出的'ETag' != 请求头中的'If-Non-Match' || 查询到的'Last-Modified'(上次修改的时间) != 请求头中的'If-Modified-Since'
	返回的response状态码200 和 数据
else
   返回的reponse状态码304
```
'Etag'与'Last-Modified'不同的是：
'Etag'更强调的是实体内容，它代表着文件的信息摘要，它是由服务器计算出来的类似于md5的值，使用'Etag'的验证是基于内容的。
'Last-Modified'实际上就是文件上次修改的时间，仅仅是一个时间戳，是从文件属性读取出来的，使用'Last-Modified'的验证是基于时间的。

了解了这些我们就可以改造上面的代码，使用双重验证：

```objectivec
- (void)refreshedRequest:(NSString *)urlString success:(void (^)(NSHTTPURLResponse *httpResponse, id responseData))successs failure:(void (^)(NSError *error))failure {
    NSMutableURLRequest *urlRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:urlString]];
    
    NSURLCache *urlCache = [NSURLCache sharedURLCache];
    NSCachedURLResponse *cacheURLResponse = [urlCache cachedResponseForRequest:urlRequest];
    if (cacheURLResponse) {
        NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)cacheURLResponse.response;
        NSString *cachedResponseEtag = [httpResponse.allHeaderFields objectForKey:@"Etag"];
        if (cachedResponseEtag) {
            [urlRequest setValue:cachedResponseEtag forHTTPHeaderField:@"If-None-Match"];
        }
        // 增加对上次修改时间的验证
        NSString *cachedResponseModified = [httpResponse.allHeaderFields objectForKey:@"Last-Modified"];
        if (cachedResponseModified) {
            [urlRequest setValue:cachedResponseModified forHTTPHeaderField:@"If-Modified-Since"];
        }
    }

    // ....
}
```

不过现在比较悲剧的是，各个web容器已经将Etag值的计算方法玩坏了，在计算Etag是依赖的参数不仅仅有文件的内容信息，还有文件的修改时间，这样一来，Etag的功能就相当于最初设计的Etag的功能+Last-Modified的功能。所以说上面的改造代码没有什么实在意义，只使用Etag就可以。

为了Etag确实不仅仅是基于内容的验证值，我做了一下测试：
先进行访问一次对http://127.0.0.1/blog文件(里面只有几个字符的文本文件)的访问，得到如下的response：

```objectivec
<NSHTTPURLResponse: 0x7fe143d170d0> { URL: http://127.0.0.1/blog } { status code: 304, headers {
    Connection = "Keep-Alive";
    Date = "Tue, 23 Feb 2016 04:16:36 GMT";
    Etag = "\"14-52c66cf22bd40\"";
    "Keep-Alive" = "timeout=5, max=100";
    Server = "Apache/2.4.16 (Unix) PHP/5.5.29";
} }
<7b0a0922 74657374 223a2268 656c6c6f 222c0a7d>
```
此时文件的MD5为：

```sh
MD5 (blog) = 35466082cffbc8fe4529a18a55f0260e
```
然后修改服务端文件的修改时间，但并没有修改文件的内容

```sh
# 将修改时间更改为2016年1月1日0点0分
touch -mt 201601010000 blog
```
这时文件的MD5值为：

```sh
MD5 (blog) = 35466082cffbc8fe4529a18a55f0260e # 没有改变
```
再次进行访问，这次访问使用忽略缓存的协议，并且带上Etag值，而不带修改时间值，得到的response是：

```objectivec
<NSHTTPURLResponse: 0x7fe143c0c870> { URL: http://127.0.0.1/blog } { status code: 200, headers {
    "Accept-Ranges" = bytes;
    Connection = "Keep-Alive";
    "Content-Length" = 20;
    Date = "Tue, 23 Feb 2016 04:20:42 GMT";
    Etag = "\"14-52833bf364000\"";
    "Keep-Alive" = "timeout=5, max=100";
    "Last-Modified" = "Thu, 31 Dec 2015 16:00:00 GMT";
    Server = "Apache/2.4.16 (Unix) PHP/5.5.29";
} }<7b0a0922 74657374 223a2268 656c6c6f 222c0a7d>
```
数据没有变，但是Etag仍然改变了。(以上是在apache+PHP的测试结果,使用Express也是这样)
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor3_0">回到顶部</a></div>

### 'Keep-Alive'响应头和不离线的URLSession
"Keep-Alive"响应头会控制客户端进行发起请求的间隔。例如：

```objectivec
"Keep-Alive" = "timeout=5, max=100"
```
其中timeout值代表着最小间隔，也就是说如果这次发送请求之后，要在5秒之后发起的请求才会进行网络访问。
max值代表着最大的尝试次数，在timeout时间内发起请求会使这个值-1直到变为0再变为设定值。
以上两个值就控制着这样一个过程：刚刚访问的一个请求，获取到了数据并进行了缓存，如果还没有过去5秒再次发起同样的请求，则不进行网络访问，直接读取缓存并且将响应头修改为`"Keep-Alive" = "timeout=5, max=99"`，如果这次请求还没过去5秒又进行请求，同样不进行网络访问，直接读取缓存，修改响应头为`"Keep-Alive" = "timeout=5, max=98"`.....直到max变为0，再来一次又变回100.

看到这里我们又能体会到缓存优化的必要性，服务端当设定了这个响应头时，也可以不受影响地拿到实时数据。

这里还要说的一个问题是Session的在线状态，例如上面的访问中，每次访问需要使用相同的session才能做到max值不断-1，如果session值改变了，相当于一次新的请求，获得的始终是`"Keep-Alive" = "timeout=5, max=100"`,也就是说，下面的两种状况是不同的。

```objectivec
- (void)onlyUseOneSession {
    NSMutableURLRequest *urlRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:kURLString]];
    
    [[[NSURLSession sharedSession] dataTaskWithRequest:urlRequest completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        // ...
    }] resume];
}

- (void)useDifferentSession {
    NSMutableURLRequest *urlRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:kURLString]];
    
    NSURLSession *session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration]];
    
    [[session dataTaskWithRequest:urlRequest completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        // ...
    }] resume];
    
    session = nil;
}
```
iOS的URLSession相当于一个浏览器窗口，它们虽然能够共用cookie和NSURLCache，但是每个session因配置不同和状态不同，会对相同url的访问有差别的。如果访问使用相同Session，那么就能公用一套配置和访问历史等信息，管理起来也是非常方便的，而如果使用的Session都是一些局部变量，那么使用之后就会离线，而且再也无法获取到这些session。因此在开发中建议使用[NSURLSession sharedSession];
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor4_0">回到顶部</a></div>

### 'Expires'响应头
这个响应头的值也是一个时间，代表着连接过期时间，它允许客户端在这个时间之前不去发网络请求，与'Keep-Alive'功能相似，但是'Keep-Alive'指定的是时间长度，而'Expires'指定的是时刻。

当服务端返返回响应头有这个值，依然是使用优化过的缓存比较稳妥。
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor5_0"></a></div>

### 这篇文章的意义
有关针对'Etag'和HTTP304状态码进行优化缓存的文章不胜枚举，那么为什么还要这样一篇文章。我想原因主要有以下几个：<br/>
1.大家都知道这个缓存的原理，可是没有讲得太明白，或者干脆直接就不讲直接上代码。以至于有人存在这种想法：我每次都用默认缓存策略也好好的，你为什么要让我优化。我认为本篇文章对于默认缓存策略和忽略缓存策略二者的优缺点描述还是很有必要的。<br/>
2.很多人的代码并没有建立在使用系统的NSURLCache的基础上，更有甚者，直接使用自定义的属性存取'Etag'的值，apple看到这样的代码会哭的，那么本地缓存中信息存在的意义是什么。<br/>
3.我个人比较想让大家读一下NSCache和NSURLCache的那几篇文章，对平常的工作确实相当有帮助的。<br/>
4.虽然我不是mattt，但我觉得mattt从未讽刺过SDWebImage。请不要将缓存和持久化存储混为一谈，也不要将文件缓存和URL缓存混为一谈。<br/>
