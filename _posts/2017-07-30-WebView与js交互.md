---
layout: cnblog_post
title: WebView与js交互
date: 2017-07-30T12:40:39.000Z
categories: iOS
---
<!--Category-->
<div id="navCategory">
	<b>本文目录</b>
	<ul>
		<li><a href="#anchor1_0">1.UIWebView与js交互</a></li>
        <li>
                <ul>
                <li><a href="#anchor1_1">向js执行环境注入方法</a></li>
                <li><a href="#anchor1_2">向js执行环境注入对象</a></li>
                </ul>
            </li>
        <li><a href="#anchor2_0">2.WKWebView与js交互</a></li>
        <li>
            <ul>
            <li><a href="#anchor2_1">向WKWebView的js环境注入代码</a></li>
            <li><a href="#anchor2_2">为js函数调用添加OC处理</a></li>
            </ul>
        </li>
	</ul>
</div>
截止到目前有两个可怕的事实我确实还不太能接受：①我从来没有在iOS项目中进行过WebView与js的交互；②大约1年多以前我做过Android的WebView与js交互。当时我也是顺手写了些iOS的UIWebView和WkWebView与js的交互过程，刚好最近经常有人问到，就将1年多之前的总结分享一下吧，因为实在是没有以实际开发为基础，个中纰漏，还望各位看官不要见笑。

<h2 id="anchor1_0">UIWebView与js交互</h2>

有一种拦截超链接跳转的方式完成UIWebView与js的交互，但是这种方法实在不能成为方式，在`webView:shouldStartLoadWithRequest:navigationType:`这个本该处理跳转逻辑的代码中掺和进业务代码，不光结构性不好，而且对于前端来说让&lt;a&gt;标签失去它本身的意义，这样的做法也会让前端看起来无比难受吧。

iOS7开始，苹果推出了JavaScriptCore框架。JavaScriptCore框架为基于Swift、OC、C的程序提供了执行js程序能力。可以使用JavaScriptCore向JavaScript环境插入自定义对象。JavaScriptCore框架作为开源的WebKit的一部分，提供给开发者的是JavaScript语言的浏览器执行环境，对前端人员来说不仅可以配合iOS人员编写的代码进行BOM和DOM编程，更可以使用环境中的自定义对象和方法。

获取当前WebView的js执行环境。

```objectivec
JSContext *context = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
```

我们可以直接使用这个环境来执行一段js代码：

```
[context evaluateScript:@"alert('Hi,Mike~')"];
```
当然你也可以直接使用BOM对象

```
[context evaluateScript:@"alert(document.location.href)"];
```

<h4 id="anchor1_1">向js执行环境注入方法</h4>

```objectivec
context[@"sayHello"] = ^{
    NSArray *args = [JSContext currentArguments];
    if (args.count && args.firstObject) {
        JSValue *name = args.firstObject;
        NSString *nameString = name.toString;
        NSLog(@"Hello, %@", nameString);
    }
};
```
<h4 id="anchor1_2">向js执行环境注入对象</h4>


如果想向js执行环境中注入一个js对象，那可就没那么简单了，你需要<br/>
①自定义一个遵守了JSExport协议的协议，这个协议中声明的属性和方法都是对js环境导出的。<br/>
②让要导出的对象类型遵守①中创建的协议，并按需实现属性的getter和setter，以及实现声明的方法。

例如我们要将当前的控制器导出为名称为vc的对象，并且将它的

```objectivec
- (void)sayHello:(NSString *)name {
    NSLog(@"%@", name);
}
```
这个方法导出为可以使用js这样调用的方法：

```js
vc.sayHello('Mike');
```

我们可以按照如下步骤实现：<br/>
①创建自定义的遵守JSExport协议的协议，并声明要导出的内容。

```objectivec
@protocol JSVCExport <JSExport>

- (void)sayHello:(NSString *)name;

@end
```

②让ViewController遵守协议，在ViewController中实现导出内容

```objectivec
// ViewController.m

@interface ViewController () <JSVCExport>
// ....
@property (nonatomic, weak) UIWebView *webView;
// ....
@end

@implementation WebViewController

- (void)sayHello:(NSString *)name {
    NSLog(@"%@", name);
}

@end
```

③执行注入

```objectivec
context["vc"] = self; // self为当前控制器
```
这样就可以让前端同学愉快地执行下面代码了

```js
vc.sayHello('Mike');
```

但是这样引发了一个严重的问题：内存泄露，产生的原因是下面的循环引用
<img src="/static/img/blogRes/webview_js_retain_cycle1.png" width="400" alt="webview、vc、context循环引用"/>

看到了循环引用的问题，有些人立马就想到了

```objectivec
__weak typeof(self) weakSelf = self;
context["vc"] = weakSelf;
```
以为这样就可以解决，实际上并不能。因为context对传递过来的对象都是强引用的。

我们可以使用两种方法来解决这个循环引用的问题：
<h5 id="anchor0_0">方法一：</h5>
引入一个新的实现了JSVCExport接口的类，而不再产生循环

```objectivec
// JSVCExportImp.h
@interface JSVCExportImp : NSObject<JSVCExport>

@end

// JSVCExportImp.m
@implementation JSVCExportImp

- (void)sayHello:(NSString *)name {
    NSLog(@"%@", name);
}

@end

// ViewController.m
context["vc"] = [JSVCExportImp new];
```
这样引用方式就变成了以下这样：
<img src="/static/img/blogRes/webview_js_retain_cycle2.png" width="400" alt="webview、vc、context循环引用2"/>

虽然解决了问题，但是为了解决问题无端增加了一个类，并且这个类本该处理逻辑却与控制器分离了，而且，如果需要使用控制器中的值的时候会变得特别难处理。
<h5 id="anchor0_0">方法二：</h5>
另外一种方式是关注于打破强引用。这里用的是YYKit中的一个弱代理类<a href="https://github.com/ibireme/YYKit/blob/master/YYKit/Utility/YYWeakProxy.m" target='blank'>YYWeakProxy</a>（这个类与YYKit框架零耦合，可单独使用），它是NSProxy的子类，用来代理其他对象，它内部保存一个属性用来弱引用被代理的对象，同时将所有的消息调用转发给被代理对象，因此被代理类的任何方法都能按照它本来的执行进行响应。

只需将传递给context的对象使用这个弱代理类包装一下即可：

```objectivec
context["vc"] = [YYWeakProxy proxyWithTarget:self];
```

这样引用方式就变成了如下所示：

<img src="/static/img/blogRes/webview_js_retain_cycle3.png" width="400" alt="webview、vc、context循环引用3"/>

<h2 id="anchor2_0">WKWebView与js交互</h2>
iOS8.0推出的WKWebView，性能相对于UIWebView有了较大的提升，使用也非常方便。WKWebView与js交互相比较UIWebView略有不同。
相对于UIWebView调用js代码的方式，WKWebView新增了一个完成回调的处理。

```objectivec
// UIWebView直接执行js
- (nullable NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;
// WKWebView直接执行js
- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^ _Nullable)(_Nullable id, NSError * _Nullable error))completionHandler;
```

再者，我们无法通过下面的方式获取到js执行环境了，

```objectivec
JSContext *context = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
```
因此上面说的关于UIWebView与js交互的方式对于WKWebView并不适用。WKWebView有它特有的与js交互的方式：

在介绍如何为WKWebView的js环境提供方法和对象时，先介绍WKWebView的一个重要属性。

```objectivec
@property (nonatomic, readonly, copy) WKWebViewConfiguration *configuration;
```
它是用来初始化一个web view的属性集合。
使用WKWebViewConfiguration类，你可以决定webpage渲染的速度，如何处理媒体播放，和一些其他的用户可选项目。
WKWebViewConfiguration仅用在web view第一次初始化的时候，你不能使用这个在web view创建之后改变它的设置。

WKWebViewConfiguration类又有一个属性：

```objectivec
@property (nonatomic, strong) WKUserContentController *userContentController;
```
它是一个与web view绑定的用户内容控制器。
WKUserContentController对象向web view提供了对JavaScript发送消息和注入用户脚本的方式。
这个用户内容控制器被web view的配置属性指定，并与web view绑定。

<h4 id="anchor2_1">向WKWebView的js环境注入代码</h4>
这里没有明确地说明注入对象或者方法，因为你可以添加任何不仅限于二者的任何预先准备好的js代码，如：

```objectivec
// 添加并执行一段js代码
WKUserScript *userScript = [[WKUserScript alloc] initWithSource:@"document.body.style.background = \"#077\";" injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
[webView.configuration.userContentController addUserScript:userScript];

// 添加js源文件中的所有内容
NSString *jsString = [NSString stringWithContentsOfFile:[[NSBundle mainBundle] pathForResource:@"tools.js" ofType:nil] encoding:NSUTF8StringEncoding error:NULL];
userScript = [[WKUserScript alloc] initWithSource:jsString injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
[webView.configuration.userContentController addUserScript:userScript];
```

这里的`WKUserScript`是对要注入的脚本的一个包装类型。

<h4 id="anchor2_2">为js函数调用添加OC处理</h4>
这里的`为js函数调用添加OC处理`看着似乎有点别扭，可相对于UIWebView的准备好OC方法来等待js调用，这里用`为js函数调用添加OC处理`来形容WKWebView更加贴切，因为它的实现过程类似于消息调用过程中的动态实现方法解析。同样是js调用OC方法，在这里却有了新的实现形式：
首先要在iOS端准备好动态消息处理的代码，并且向webView注册这个方法的Handler：

```objectivec
// 向webView注册这个方法的Handler
[webView.configuration.userContentController addScriptMessageHandler:(ViewController *)[YYWeakProxy proxyWithTarget:self] name:@"sayHello"]; // 这里我们同样使用YYWeakProxy来解除循环引用。

// 准备好处理方法处理的代码：遵守WKScriptMessageHandler协议，实现userContentController:didReceiveScriptMessage:;
@interface ViewController () <WKScriptMessageHandler>
// ...
@property (nonatomic, weak) WKWebView *webView;
// ...
@end

@implementation ViewController

#pragma mark - WKScriptMessageHandler

- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message {
    if ([message.name isEqualToString:@"sayHello"]) {
        if ([message.body isKindOfClass:[NSString class]] && [message.body length] > 0) {
            // 业务实现
            NSLog(@"Hello, %@", message.body);
        }
    }
}

@end
```

对于js调用方，应该这样调用：

```js
window.webkit.messageHandlers.sayHello.postMessage('Mike');
```
其调用原型为

```js
window.webkit.messageHandlers.{MESSAGE_NAME}.postMessage()
```

这样调用的方式或许非常让人接受不了，对于前端同学来说或许可以适当地对这个代码进行封装:

```js
// tools.js
function sayHello(name) {
    if (isIOS) {
        window.webkit.messageHandlers.sayHello.postMessage(name);
    } else if (isANDROID) {
        // ...
    }
}
```
这样这个tools.js可以直接当做SDK使用。

另外要注意一点就是，WKWebView使用`webView.configuration.userContentController`来设置交互内容相比UIWebView的优势是：<br/>
WKWebView跳转页无须再次注入，只要是一个webView就会公用userContentController；<br/>
而对于UIWebView每当超链接跳转一次之后注入的代码便会失效，因为它是向mainFrame注入的。


