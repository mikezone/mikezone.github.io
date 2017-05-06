---
layout: cnblog_post
title:  "RunLoop和autorelease的一道面试题"
date:   2015-04-21 13:50:39
categories: iOS
---
<!--prettify.js-->
<link rel="stylesheet" type="text/css" href="http://cdn.bootcss.com/prettify/r298/prettify.min.css">
<script src="http://cdn.bootcss.com/prettify/188.0.0/prettify.min.js"></script>
<script type="text/javascript">
  window.onload = function() {
    prettyPrint();
  }
</script>

<p><a style="height: 0px;" name="labelTop"></a></p>
<p><span> 有这么一道iOS面试题 以下代码有没有什么问题?如果有?如何解决? </span></p>
<pre class="prettyprint">for (int i = 0; i &lt; largeNumber; i++) {	
	NSString *str = [NSString stringWithFormat:@"hello -%04d", i];
	str = [str stringByAppendingString:@" - world"]; 
}
</pre>
<h2><span>局部释放池和RunLoop释放池的概念：</span></h2>
<p><span> 主线程的RunLoop是默认开启的(视图用[[NSRunLoop currentRunLoop] runUntilDate:[NSDate date]]来停止它，也是做不到的)， 每一次消息循环开始的时候会先创建自动释放池，这次循环结束前,会释放自动释放池，然后RunLoop等待下次事件源。 在这个过程中，由RunLoop创建的释放池类似于一个全局的释放池。但是开发者可以任何执行的地方创建释放池，也就是局部的释放池，这时的释放池类似于代码块 当释放池结束的时候会自动释放。因此一般情况下，局部的自动释放池很快就被释放了，而RunLoop释放池会等一次消息循环结束的时候释放。 </span></p>
<h2><span>什么样的对象会交给释放池管理：</span></h2>
<p><span> 返回当前类的实例的类方法创建出来的对象，都是autorelease的，会交给所在的释放池进行管理。 例如创建一个Person类,使用[[self alloc]init]方法创建的对象的管理不会交给它所在的释放池，而是根据引用计数来控制释放的时机, 如果使用[[[self alloc]init] autorelease]创建的对象，会交给所在的释放池管理，控制其释放的时机。 </span></p>
<pre class="prettyprint">
- (void)test {
    @autoreleasepool {
        Person *p = [[Person alloc] init];
        p = nil;
        NSLog(@"---");
    }
    NSLog(@"autorelease结束");
}
</pre>
<p><span>执行结果：</span></p>
<pre class="prettyprint">Person---dealloc
---
autorelease结束
</pre>
<p>&nbsp;</p>
<pre class="prettyprint">
- (void)test1 {    
    @autoreleasepool {
        Person *p = [Person person]; // 内部是[[[self alloc] init] autorelease]
        p = nil;
        NSLog(@"---");
    }
    NSLog(@"autorelease结束");
}
</pre>
<p><span>执行的结果为：</span></p>
<pre class="prettyprint">
---
Person---dealloc
autorelease结束
</pre>
<p><span> 因此自动释放池被销毁或耗尽时会向池中所有使用autorelease创建的对象发送release 消息,释放所有autorelease的对象，而不是所有的对象。 </span></p>
<h2><span>回到面试的问题:</span></h2>
<p><span> 当我们使用for循环创建很多个使用autorelease方式创建的NSString对象的时候，将所有的对象的释放权都交给了RunLoop 的释放池，而RunLoop的释放池会等待这个事件处理之后才会释放，因此就会使对象无法及时释放，堆积在内存造成内存泄露，可以在Debug Navigation 中观察到内存激增。为了验证确实是因为autorelease这种创建方式引起的内存泄露，我做了如下的测试： </span></p>
<pre class="prettyprint">
int largeNumber = 100000000;
- (void)correctSolution1{
    for (int i = 0; i &lt; largeNumber; i++) {
        NSString *str = [[NSString alloc] initWithFormat:@"hello -%04d", i];
        str = [str stringByAppendingString:@" - world"];
    }
}
// stringWithFormat:本质上是调用了alloc + initWithFormat: + autorelease
// 我在本例中将stringWithFormat:方法换成了alloc + initWithFormat:
// 这样做问题就解决了：内存几乎没有变化。反向验证了内存飙升确实是autorelease创建方式造成的。
</pre>
<p><span>但是在编写代码的时候我们仍然习惯用类的快速创建方法，而不是alloc+init。也就是说，为了方便写程序，又使用了底层实现是alloc+init+autorelease的快速创建对象的方法(如 stringWithFormat:)。因此解决的方案就是添加局部的释放池，以及时释放内存，如果将局部释放池添加到循环外：</span></p>
<pre class="prettyprint">
- (void)wrongSolution{
    @autoreleasepool {
        for (int i = 0; i &lt; largeNumber; i++) {
            NSString *str = [NSString stringWithFormat:@"hello -%04d", i];
            str = [str stringByAppendingString:@" - world"];
        }
    }
}
</pre>
<p><span>这样显然是没有效果的，释放池需要等循环执行之后再释放内存，这和使用RunLoop创建的释放池没有什么区别。 较好的方案就是每次循环的时候添加一个释放池：</span></p>
<pre class="prettyprint">
- (void)correctSolution2{
    for (int i = 0; i &lt; largeNumber; i++) {
        @autoreleasepool {
            NSString *str = [NSString stringWithFormat:@"hello -%04d", i];
            str = [str stringByAppendingString:@" - world"];
        }
    }
}
</pre>
<p><span> 这样每一次循环的结束时都会释放一次内存，因而这个循环全部执行完成时也几乎不消耗内存。 </span></p>
<h3><span>总结是：</span></h3>
<p><span> 做多线程开发时,需要在线程调度方法中手动添加自动释放池，尤其是当执行循环的时候，如果循环内部有使用类的快速创建方法创建的对象， 一定要将循环体放到自动释放池中。</span></p>