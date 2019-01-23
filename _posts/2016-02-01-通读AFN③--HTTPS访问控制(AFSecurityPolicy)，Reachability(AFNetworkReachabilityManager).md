---
layout: cnblog_post
title:  "通读AFN③--HTTPS访问控制(AFSecurityPolicy)，Reachability(AFNetworkReachabilityManager)"
date:   2016-02-01 13:50:39
categories: iOS
---
这一篇主要介绍使用AFN如何访问HTTPS网站以及这些做法的实现原理，还有介绍AFN的网络状态监测部分AFNetworkReachabilityManager，这个模块会和苹果官方推荐的Reachability框架做一个对比。

本文所有的代码都运行在iOS9.2的模拟器上，并且在info.plist对ATS做了适配:设置允许非法的加载Allow Arbitrary Loads为YES。

不要认为在info.plist添加`NSAppTransportSecurity` > `NSAllowsArbitraryLoads`为YES
就以为弄懂iOS9网络适配了，有关具体细节问题请看南峰子的这篇文章<a href="http://southpeak.github.io/blog/2015/09/14/app-transport-security-ats/" target='blank'>App Transport Security(ATS)</a>

介于iOS有关HTTPS访问的认证过程代码并不是特别经常使用，本文会用大量的篇幅介绍HTTPS认证的过程，并会通过系统的NSURLSession完成一些认证相关的代码，毕竟AFN就是使用了这些代码来实现对HTTPS网站的访问支持的。

### HTTPS网站访问过程中，浏览器帮你做了什么<br/>
不同于普通的HTTP请求，当访问一个HTTPS的网站时，浏览器会帮我们很多隐藏的工作，这其实是SSL通道建立的三次握手过程：<br/>
1.发起请求。<br/>
首先当输入完https网址敲击回车之后，浏览器首先向服务器发送一个需要访问的请求，这个请求中包含着浏览器SSL 协议的版本号，加密算法的种类，产生的随机数，以及其他服务器和客户端之间通讯所需要的各种信息。<br/>
2.服务端返回证书。<br/>
服务器向客户端传送SSL 协议的版本号，加密算法的种类，随机数以及其他相关信息，同时服务器还将向客户端传送自己的证书，这些信息被保存在客户端被称作'被保护空间'的地方。这里最关键的就是证书信息。<br/>
3.浏览器验证证书信息。<br/>
浏览器利用服务器传过来的信息验证服务器的合法性，服务器的合法性包括：证书是否过期，发行服务器证书的CA 是否可靠，发行者证书的公钥能否正确解开服务器证书的“发行者的数字签名”，服务器证书上的域名是否和服务器的实际域名相匹配。<br/>
如果合法性验证没有通过，通讯将断开；如果合法性验证通过，将继续进行第四步。<br/>
4.客户端向服务器发送“预主密码”。<br/>
浏览器随机产生一个用于后面通讯的“对称密码”，然后用服务器的公钥（服务器的公钥从步骤②中的服务器的证书中获得）对其加密，然后将加密后的“预主密码”传给服务器。<br/>

4.1.如果服务器要求客户的身份认证（在握手过程中为可选），用户不光要传给服务器“预主密码”，还需建立一个随机数然后对其进行数据签名，将这个含有签名的随机数和客户自己的证书也传给服务器。

4.2.如果不需要，则只将“预主密码”传给服务器，并直接进行第6步。<br/>
5.服务端身份验证(需要才进行)。<br/>
如果服务器要求客户的身份认证，服务器必须检验客户证书和签名随机数的合法性，具体的合法性验证过程包括：客户的证书使用日期是否有效，为客户提供证书的CA 是否可靠，发行CA 的公钥能否正确解开客户证书的发行CA 的数字签名，检查客户的证书是否在证书废止列表（CRL）中。<br/>
检验如果没有通过，通讯立刻中断；<br/>
如果验证通过，进行下一步。<br/>
6.浏览器、服务端各自生成通话密码。<br/>
服务器将用自己的私钥解开加密的“预主密码”，然后执行一系列步骤来产生主通讯密码（客户端也将通过同样的方法产生相同的主通讯密码）。<br/>
7.约定通话密码。<br/>
服务器和客户端用相同的主通讯密码即“通话密码”，一个对称密钥用于SSL 协议的安全数据通讯的加解密通讯。同时在SSL 通讯过程中还要完成数据通讯的完整性，防止数据通讯中的任何变化。<br/>
8.浏览器通知服务器已准备就绪。<br/>
客户端向服务器端发出信息，指明后面的数据通讯将使用的步骤⑦中的主密码为对称密钥，同时通知服务器客户端的握手过程结束。<br/>
9.服务端通知浏览器已准备就绪。<br/>
服务器向客户端发出信息，指明后面的数据通讯将使用的步骤⑦中的主密码为对称密钥，同时通知客户端服务器端的握手过程结束。<br/>
10.开始数据通讯。<br/>
SSL 的握手部分结束，SSL安全通道建立完成，开始进行数据通讯开始，通讯过程中客户和服务器开始使用相同的对称密钥。<br/>
如果以https://www.baidu.com为例，这时候已经表现为baidu的主页打开了，但是SSL加密通道在下次请求的时候不用再次建立。

对于访问的过程中，通常会在第3步出现问题，以12306的购票页面为例：
当进行到第3步的时候，浏览器验证为：发行服务器证书的CA是不可靠的，可以在Chrome的地址栏中点击被打了红叉的锁来查看这个页面的证书颁发机构，
<img src="http://qiniu.storage.mikezh.com/img%2FSnip20160131_9.png" width="600" alt="12306HTTPS证书"/>
我们可以搜索到这个命名为'SRCA'的机构实际上是‘中铁认证中心’也就是12306自己的认证系统，它是用了自己的认证系统给自己颁发了一个SSL加密证书，而Chrome怎么会认可它呢。顺便看了一下百度的证书:
<img src="http://qiniu.storage.mikezh.com/img%2FSnip20160131_10.png" width="600" alt="baiduHTTPS证书"/>
这是一个由美国Symantec Trust Network组织颁发的证书，是一个比较权威的证书颁发机构，几乎在所有的浏览器中都是认可的。而baidu使用的证书是这个机构的根证书的子证书，而之所以浏览器能认可它，是因为根证书通过webtrust国际认证，并已经内置到各大浏览器如谷歌，火狐，微软等系统中。
那么这毕竟只是浏览器默认的一种认证方式，毕竟我们还是需要访问12306的，这里就要改变一下第3步验证的结果，在浏览器中，我们可以手动选择信任，然后继续向下进行。
<img src="http://qiniu.storage.mikezh.com/img%2FSnip20160131_12.png" width="400" alt="手动信任证书"/>
这样就能访问这些网站了。

### 使用系统的NSURLSession模拟浏览器完成HTTPS的证书认证
与浏览器的验证过程相似，iOS的HTTPS验证过程也要走类似的步骤，不过不用担心的是，很多过程我们也不需要处理，只需要处理好第3步就行了，当我们进行访问一个HTTPS网站时，当走到第二步的时候，也就是服务器返回证书时，需要我们在本地自己完成证书信任的过程，如果使用session创建的task进行网络访问，这时候就会进入到`- URLSession:didReceiveChallenge:completionHandler:`这个代理方法中,这时候已经完成了HTTPS访问的第二步，session会让我们在这个方法中完成第3步的过程。这个方法的参数有如下的解释：

| 参数 | 解释 |
| -----  | -----  |
| challenge |   一个包含了授权请求的对象 |  
| completionHandler |  你的代理方法一定会调用的一个handler. 它的参数是<br/>disposition—描述challenge如何被处理的几个常量中的一个 <br/>credential—如果disposition是NSURLSessionAuthChallengeUseCredential,credential是授权验证时会被使用到的凭据,其他情况为NULL. | 

challenge参数需要另外说明的是`challenge`是一个`NSURLAuthenticationChallenge`对象，代表着进行https请求进行时，服务端发送过来的质询，当接收到质询之后就要开始进行客户端的验证了。

这个对象中最重要的属性就是`protectionSpace`它代表着对需要验证的受保护空间的验证，是一个`NSURLProtectionSpace`类型的对象。NSURLProtectionSpace对象包含请求的主机host、端口号port、代理类型proxyType、使用的协议protocol、服务端要求客户端对其验证的方法authenticationMethod等重要的信息，还有代表着服务器SSL传输状态的`SecTrustRef`类型的属性serverTrust，不过当且仅当authenticationMethod为NSURLAuthenticationMethodServerTrust这个属性值才不为Nil.

这里还要说明一下服务端指定的验证方法的类型，验证方法的类型有很多种，这里不再一一列举，我们通常会见到这样几种类型：

```objectivec
NSURLAuthenticationMethodHTTPBasic 
NSURLAuthenticationMethodHTTPDigest
NSURLAuthenticationMethodNTLM
NSURLAuthenticationMethodClientCertificate
NSURLAuthenticationMethodServerTrust
```
其中HTTP Basic、HTTP Digest与NTLM认证都是基于用户名/密码的认证，ClientCertificate(客户端证书)认证要求从客户端上传证书。客户端需要按照服务端指定的认证方法进行认证，否则可能会按照错误处理。例如使用HTTP Basic方式，客户端需要将用户名和密码信息放到凭据中，然后传递给服务端;如果使用的是ServerTrust方式，那么客户端就要将信任的凭据发给服务端。

一般在HTTPS访问的第3步过程中，服务端要求的认证方法几乎总是ServerTrust方式。有遇到过一些网络代理工具使用HTTP Digest的验证方式，在浏览器端进行访问的时候就弹出一个要求输入账号和密码的弹窗。

对于completionHandler参数是一个最终处理凭据的回调，要求在创建好包含验证信息的凭据之后必须调用，这样才会将验证的信息发送给服务端，也就意味着第3步的完成，开始进行第4步。
它的第一个参数是处理的选项，是一个枚举类型：

```objectivec
typedef NS_ENUM(NSInteger, NSURLSessionAuthChallengeDisposition) {
    NSURLSessionAuthChallengeUseCredential = 0,   // 使用服务器发回的凭据,不过可能为空     
    NSURLSessionAuthChallengePerformDefaultHandling = 1,  // 默认的处理方法，凭据参数会被忽略
    NSURLSessionAuthChallengeCancelAuthenticationChallenge = 2,  //取消整个请求，忽略凭据参数 
    NSURLSessionAuthChallengeRejectProtectionSpace = 3, // 这次质询被拒绝，下次再试 ,凭据参数被忽略
} NS_ENUM_AVAILABLE(NSURLSESSION_AVAILABLE, 7_0);
```
理清上面的思路之后，我们可以试一试使用系统的session访问HTTPS网站了：

```objectivec
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSURL *url = [NSURL URLWithString:@"https://www.baidu.com"];
    [[self.session dataTaskWithURL:url completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        if (error) {
            NSLog(@"%@", error);
            return ;
        }
        NSLog(@"%@", response);
    }] resume];
}

#pragma mark - NSURLSessionDelegate

- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * __nullable credential))completionHandler {
    // 判断服务器的身份验证的方法是否是：ServerTrust方式
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        
        // 创建一个新凭据，这个凭据指定了'握手'是被信任的
        NSURLCredential *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
        
        if (credential != nil) {
            // 完成'处置',将信任凭据发给服务端
            completionHandler(NSURLSessionAuthChallengeUseCredential, credential);
        }
        // 如果credential == nil 以下回调会自动完成
        // completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, credential);
    }
}
```
因为我们使用的是使用第2步中服务端传回来的证书，所以即使是对付https://kyfw.12306.cn/otn/leftTicket/init这样的流氓页面也同样是可以的。但是对于iOS9来说并不是这样，必须设置了Allow Arbitrary Loads为YES才会达到预期效果。

对于AFN，无论实在iOS9之前还是iOS9之后，当访问https://kyfw.12306.cn/otn/leftTicket/这个页面的时候都会走不通，这是因为AFN对于自签名的HTTPS网站有着特殊的验证(有关验证细节，请看本文下一部分)，必须证书提前导入到项目中，将Chrome中的证书导入到项目中，请参见下图：
<img src="http://qiniu.storage.mikezh.com/img%2Fchrome_cer.gif" width="720" alt="Chrome生成证书"/>
将生成的证书文件`kyfw.12306.cn.cer`加入到xcode项目中，使用AFN按照如下方式调用即可：

```objectivec
NSString *cerPath = [[NSBundle mainBundle] pathForResource:@"kyfw.12306.cn.cer" ofType:nil];
NSData *cerData = [NSData dataWithContentsOfFile:cerPath];
NSSet *set = [[NSSet alloc] initWithObjects:cerData, nil];

AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];

manager.responseSerializer = [AFHTTPResponseSerializer serializer];
manager.securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeCertificate withPinnedCertificates:set];
manager.securityPolicy.allowInvalidCertificates = YES;

[manager GET:@"https://kyfw.12306.cn/otn/leftTicket/init" parameters:nil success:^(NSURLSessionDataTask *task, id responseObject) {
    NSLog(@"%@", [[NSString alloc] initWithData:responseObject encoding:NSUTF8StringEncoding]);
} failure:^(NSURLSessionDataTask *task, NSError *error) {
    NSLog(@"%@",error);
}];
```
这样便能正确的访问自签名的网站了。

### AFN实现HTTPS访问的细节
说了那么多如何使用代码访问HTTPS网站，那么AFN是如何实现的呢,AFURLSessionManager中实现了`- URLSession:didReceiveChallenge:completionHandler:`代理方法：

```objectivec
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
{
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    __block NSURLCredential *credential = nil;

    if (self.taskDidReceiveAuthenticationChallenge) {
        disposition = self.taskDidReceiveAuthenticationChallenge(session, task, challenge, &credential);
    } else {
        if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
            if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
                disposition = NSURLSessionAuthChallengeUseCredential;
                credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
            } else {
                disposition = NSURLSessionAuthChallengeRejectProtectionSpace;
            }
        } else {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    }

    if (completionHandler) {
        completionHandler(disposition, credential);
    }
}
```
它的思路上这样的
如果主动通过manger的`setTaskDidReceiveAuthenticationChallengeBlock:`方法传递了taskDidReceiveAuthenticationChallenge的值那么，会按照传入的block处理这次质询，
如果没有传入就走AFN处理方式(else分支)：

如果验证方法为ServerTrust就会使用securityPolicy属性的方法针对host评判serverTrust的合法性，如果成功了就会使用服务端传来的证书进行处理，失败了则会拒绝本次质询。

如果验证方法不是ServerTrust，则使用默认的处理方式(NSURLSessionAuthChallengePerformDefaultHandling)处理。

那么，可以看出，这里最关键的就是评判合法性的过程了,我们重点来看一下。评判合法性的方法被定义在AFSecurity类中，是这个类唯一的对象方法：

```objectivec
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
{
    if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
        NSLog(@"In order to validate a domain name for self signed certificates, you MUST use pinning.");
        return NO;
    }

    NSMutableArray *policies = [NSMutableArray array];
    if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }

    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);

    if (self.SSLPinningMode == AFSSLPinningModeNone) {
        return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
    } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
        return NO;
    }

    switch (self.SSLPinningMode) {
        case AFSSLPinningModeNone:
        default:
            return NO;
        case AFSSLPinningModeCertificate: {
            NSMutableArray *pinnedCertificates = [NSMutableArray array];
            for (NSData *certificateData in self.pinnedCertificates) {
                [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
            }
            SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

            if (!AFServerTrustIsValid(serverTrust)) {
                return NO;
            }
            for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
                if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
                    return YES;
                }
            }
            
            return NO;
        }
        case AFSSLPinningModePublicKey: {
            NSUInteger trustedPublicKeyCount = 0;
            NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

            for (id trustChainPublicKey in publicKeys) {
                for (id pinnedPublicKey in self.pinnedPublicKeys) {
                    if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                        trustedPublicKeyCount += 1;
                    }
                }
            }
            return trustedPublicKeyCount > 0;
        }
    }
    
    return NO;
}
```
这段长度为60行的代码实现了这样的过程：<br/>
第一个if分支是对自签名访问设立条件：<br/>
domain不存在，或者<br/>
不允许无效证书，或者<br/>
不需要验证域名，或者<br/>
SSLPinningMode不是AFSSLPinningModeNone，而且必须上传了证书文件。如果是走了这个分支，就要求如果想要实现自签名的HTTPS访问成功，必须设置pinnedCertificates，且不能使用defaultPolicy，因为不能SSLPinningMode属性是readonly的，而defaultPolicy在创建的时候已经设置SSLPinningMode属性为AFSSLPinningModeNone。(我们刚才的实现方案就是在这条分支下完成的)

接下来是这样一块代码：

```objectivec
NSMutableArray *policies = [NSMutableArray array];
if (self.validatesDomainName) {
    [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
} else {
    [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
}

SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);

if (self.SSLPinningMode == AFSSLPinningModeNone) {
    return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
} else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
    return NO;
}
```
它完成的工作是：

先用policies数组组装验证策略，在通过SecTrustSetPolicies函数给serverTrust设置验证策略，不过AFN并没有接收函数的返回值，查看是否设置成功，不知道是为什么。<br/>
当SSLPinningMode为AFSSLPinningModeNone时，如果允许无效的证书(allowInvalidCertificates = YES)直接返回评测成功，如果不允许，按照刚才的验证策略验证，返回的是验证的结果。<br/>
当SSLPinningMode不是AFSSLPinningModeNone时，如果既没有验证成功又不允许无效证书，则直接返回评测失败。

(这里让我想到了另一种访问12306实现的方案：

```objectivec
manager.securityPolicy.validatesDomainName = NO;
manager.securityPolicy.allowInvalidCertificates = YES;
```
既不用使用证书，也不用自己创建securityPolicy。
)

接下来看一下那个长长的switch：

如果self.SSLPinningMode是AFSSLPinningModeCertificate：取出self.pinnedCertificates中的所有证书，通过SecTrustSetAnchorCertificates函数设置证书验证策略，失败则直接返回评测失败，否则检查本地的证书是否包含服务端的证书
，如果是返回评测成功，否则返回评测失败。

如果self.SSLPinningMode是AFSSLPinningModePublicKey：取出服务端证书的所有公钥，和self.pinnedPublicKeys中所有公钥，遍历检查有没有相等的两项，有则返回评测成功。我尝试给securityPolicy的pinnedPublicKeys赋值一个公钥集合，但是它并没有对外提供接口，self.pinnedPublicKeys是一个私有属性，并且是计算型的，是从本地的证书self.pinnedCertificates中提取出来的。

有关AFSecurityPolicy最核心的部分基本上将完了，最后我们还是要总结一下，访问可恶的12306的两种方法：

```objectivec
// 方式一 两句就可以
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
manager.responseSerializer = [AFHTTPResponseSerializer serializer];
manager.securityPolicy.validatesDomainName = NO; // 关键语句1
manager.securityPolicy.allowInvalidCertificates = YES; // 关键语句2
[manager GET:@"https://kyfw.12306.cn/otn/leftTicket/init" parameters:nil success:^(NSURLSessionDataTask *task, id responseObject) {
    NSLog(@"%@", responseObject);
} failure:^(NSURLSessionDataTask *task, NSError *error) {
}];


// 方式二 需要将证书导入到项目中
// 准备：将证书的二进制读取，放入set中
NSString *cerPath = [[NSBundle mainBundle] pathForResource:@"kyfw.12306.cn.cer" ofType:nil];
NSData *cerData = [NSData dataWithContentsOfFile:cerPath];
NSSet *set = [[NSSet alloc] initWithObjects:cerData, nil];

AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
manager.responseSerializer = [AFHTTPResponseSerializer serializer];
manager.securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeCertificate withPinnedCertificates:set]; // 关键语句1
manager.securityPolicy.allowInvalidCertificates = YES; // 关键语句2
[manager GET:@"https://kyfw.12306.cn/otn/leftTicket/init" parameters:nil success:^(NSURLSessionDataTask *task, id responseObject) {
    NSLog(@"%@", responseObject);
} failure:^(NSURLSessionDataTask *task, NSError *error) {
}];
```

### AFN的AFNetworkReachabilityManager和Reachability
有关AFNetworkReachabilityManager使用比较简单，不做太多的解释，只是罗列一些注意点。
AFN开启必须开启监控之后才能获取到新的网络状态，如果不开启各种网络状态都为不可到达，例如

```objectivec
AFNetworkReachabilityManager *reachabilityManager = [AFNetworkReachabilityManager sharedManager]; 

NSLog(@"%zd", reachabilityManager.isReachableViaWiFi); // 始终是0
NSLog(@"%zd", reachabilityManager.isReachable);
NSLog(@"%zd", reachabilityManager.isReachableViaWWAN);
```
即使开启了网络监控，也无法再第一时间获取到网络状态，例如下面的代码执行之后，第一时间查看各种状态依然不可达，这是因为它会在网络状况改变时，异步改变单例中存储的状态。

```objectivec
AFNetworkReachabilityManager *reachabilityManager = [AFNetworkReachabilityManager sharedManager];
[reachabilityManager startMonitoring]; // 从开启监控  到得到下列值需要一定的时间
NSLog(@"%zd", reachabilityManager.isReachableViaWiFi); // 立刻调用为0 ，过一段时间后准确
NSLog(@"%zd", reachabilityManager.isReachable); // 立刻调用为0 ，过一段时间后准确
NSLog(@"%zd", reachabilityManager.isReachableViaWWAN); // 立刻调用为0 ，过一段时间后准确
```

其实我使用较多的还是Reachability框架，
Reachability具有获取实时网络状态的`-currentReachabilityStatus`方法，不需要开启监控，只要用实例调用即可。
Reachability同样可以进行网络状态改变的监控，可以用`-startNotifier`方法开启，但是没法传入回调。但是每当网络状态改变的时候会发送一个`kReachabilityChangedNotification`通知，可以接收这个通知完成回调。