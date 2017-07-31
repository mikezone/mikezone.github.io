---
layout: cnblog_post
title: NSOperation
date: 2017-07-20T12:40:39.000Z
categories: iOS
---
<!--Category-->
<div id="navCategory">
	<b>本文目录</b>
	<ul>
		<li><a href="#anchor1_0">NSOperation</a></li>
        <li>
            <ul>
            <li><a href="#anchor1_1">NSInvocationOperation、NSBlockOperation</a></li>
            <li><a href="#anchor1_2">NSOperation基本属性</a></li>
            <li><a href="#anchor1_3">start和main方法</a></li>
            <li><a href="#anchor1_4">自定义NSOperation（OC、Swift）</a></li>
            <li><a href="#anchor1_5">其他属性和方法</a></li>
            </ul>
        </li>
        <li><a href="#anchor2_0">NSOperationQueue</a></li>
        <li><a href="#anchor4_0">NSOperation与GCD对比</a></li>
	</ul>
</div>
<!--Category结束-->
<h2 id="anchor1_0">NSOperation特性介绍</h2>
NSOperaion + NSOperationQueue实现并发编程的技术在iOS2.0已经存在，并在iOS4.0时期，也就是GCD诞生的时候进行了重写。

NSOperation类是一个抽象类，你使用它来封装代码和与任务关联的数据。因为他是抽象的，所以不能直接使用这个类，而是使用它的子类或系统定义好的子类（NSInvocationOperation 或者 NSBlockOperation）来执行实际的任务。尽管是抽象的，但是NSOperation的基本实现包含了明确的逻辑来协调你的任务执行的安全性。这个内置逻辑的实现可以使你只需关注你的任务的实现，而不用关注用来保证他与其他系统对象良好协作的整合代码。

一个operation对象是一次性对象，这意味着它一旦执行它的任务就不能再次执行。你通常通过把它们添加到一个operation queue(一个NSOperationQueue类的实例)来执行操作。一个operation queue要么会在子线程中直接运行它们，要么会使用libdispatch库(也称为GCD)间接地运行。想要获取更多关于queue如何执行operations的信息，参见NSOperationQueue。

如果你不想使用operation queue，你可以直接调用start方法执行一个operation。手动执行操作会加重你的代码的负担，因为开始一个没有处于准备状态的operation会触发异常。ready这个属性代表着operation的准备状态。

Foundation 提供了两个NSOperation具体子类，可以直接使用:

| 类 | 描述 |
| ----- | ----- |
| NSInvocationOperation | 可以直接使用的类，基于应用的一个对象和 selector 来创 建 operation object。如果你已经有现有的方法来执行需要的任务，就可以使用这个类。|
| NSBlockOperation | 可以直接使用的类，用来并发地执行一个或多个 block 对象。operation object 使用“组”的语义来执行多个 block 对 象，所有相关的 block 都执行完成之后，operation object 才算完成。|
| NSOperation | 基类，用来自定义子类 operation object。继承 NSOperation 可以完全控制 operation object 的实现，包括修改操作执 行和状态报告的方式。 |

所有的NSOperation对象都支持以下关键特性：

| 支持建立基于图的operation objects依赖。可以阻止某个operation运行，直到它依赖的所有 operation 都已经完成。 |
| 支持可选的 completion block，在 operation 的主任务完成后调用。 |
| 支持应用使用 KVO 通知来监控 operation 的执行状态。 |
| 支持 operation 优先级，从而影响相对的执行顺序。 |
| 支持取消语义，允许你中止正在执行的任务。|

Operation 被设计用来帮助你提升应用的并发等级。Operation也是一种用来将程序行为组织和封装为简单的独立的区块的好的方式。为代替在主线程上的一小部分的代码的执行， 你可以提交一个或多个operation对象到队列中来让相应的工作异步执行在一个或多个其他线程。

<h4 id="anchor1_1">NSInvocationOperation和NSBlockOperation</h4>

通常我们通过将operation添加到 operation queue 中来执行该操作。这样operation会在一个与当前线程不同的子线程中执行。

如果已经现有一个方法，需要并发地执行，就可以直接创
建 NSInvocationOperation 对象，而不需要自己继承并实现自定义的NSOperation。
NSInvocationOperation类是一个NSOperation的子类来管理被指定的invocation包装好的单个任务。你可以使用这个类来初始化一个由在指定对象上的对selector的调用构成的operation。这个类实现了一个非并发的操作。

```objectivec
- (NSOperation*)taskWithData:(id)data {
    NSInvocationOperation *operation = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(myTaskMethod:) object:data];
    return operation;
}

// This is the method that does the actual work of the task.
- (void)myTaskMethod:(id)data {
    // Perform the task.
}
```

NSBlockOperation 对象用于封装一个或多个 block 对象，一般创建时 会添加至少一个 block，然后再根据需要添加更多的 block。
当 NSBlockOperation 对象执行时，会把所有 block 提交到默认优先级的并发 dispatch queue。然后 NSBlockOperation 对象等待所有 block 完成 执行，最后标记自己已完成。因此可以使用 block operation 来跟踪一组 执行中的 block，有点类似于 thread join 等待多个线程的结果。区别在于 block operation 本身也运行在一个单独的线程，应用的其它线程在等
待 block operation 完成时可以继续工作。

```objectivec
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock: ^{
    NSLog(@"Beginning operation.\n");
    // Do some work.
}];
```
使用 addExecutionBlock: 可以添加更多 block 到这个 block operation 对象。如果需要顺序地执行 block，你必须直接提交到所需的 dispatch queue.

<h4 id="anchor1_2">基本属性</h4>

NSOperation有几个关于操作执行状态的属性，贯穿了操作从创建到销毁的整个声明周期。

```
@property (readonly, getter=isCancelled) BOOL cancelled;
@property (readonly, getter=isExecuting) BOOL executing;
@property (readonly, getter=isFinished) BOOL finished;
@property (readonly, getter=isReady) BOOL ready;
```

`@property (readonly, getter=isCancelled) BOOL cancelled;`<br/>
一个标识操作是否被取消的布尔值。<br/>
默认值是NO。调用cancel方法会设置这个值为YES。一旦取消了，这个操作的状态必须变为finished。<br/>
取消一个操作不会停止接收者正在执行的代码。一个操作对象有责任间歇性地调用这个方法并在当这个方法返回YES时停止操作本身的进行。<br/>
你应该总是在做任何为了完成任务的工作之前检查这个属性的值，着通常意味着在开始你的自定义main方法执行检查它。一个操作在开始执行之前或者在正在执行时的任何时刻被取消都是有可能的。因此，在main方法开始时(还有贯穿整个方法的任何阶段)检查这个值会让你在操作被取消的时候尽快地退出执行。

operation 开始执行之后，会一直执行任务直到完成，或者显式地取消操作。取消可能在任何时候发生，甚至在 operation 执行之前。尽管 NSOperation 提供了一个方法可以让应用取消一个操作，但是识别出取消事件则是你的事情。如果 operation 直接终止，可能无法回收所有已分配的内存或资源。因此 operation 对象需要检测取消事件，并优雅地退出执行。

operation对象阶段性地调用isCancelled方法，如果返回YES(表示已 取消)，则立即退出执行。不管是自定义NSOperation 子类，还是使用系统提供的两个具体子类，都需要支持取消。isCancelled 方法本身非常轻量，可以频繁地调用而不产生大的性能损失。以下地方可能需要调用 isCancelled:

>在执行任何实际的工作之前<br/>
>在循环的每次迭代过程中，如果每个迭代相对较长可能需要调用
多次<br/>
>代码中相对比较容易中止操作的任何地方


`@property (readonly, getter=isExecuting) BOOL executing;`<br/>
一个标识操作当前是否正在执行的布尔值。<br/>
如果操作当前正在执行它的主任务这个value的值为YES，否则为NO。<br/>
当实现一个并发操作对象时，你必须重写这个属性的实现来获取操作的执行状态。在你的自定义实现中，你必须在任何操作状态改变的时候生成对isExecuting键的KVO通知。想要获取更多关心手动生成KVO的通知，参见Key-Value Observing Programming Guide。<br/>
你不需要针对非并发的操作重新实现这个属性。

@property (readonly, getter=isFinished) BOOL finished;`<br/>
一个标识操作是否已经执行完毕的布尔值。<br/>
如果操作的主任务执行完毕返回YES，否则，如果正在执行或者还没有开始执行返回NO。<br/>
当实现一个并发操作对象的时候，你必须重写这个属性的实现来返回操作finished状态。在你的自定义实现里，你必须在任何操作对象的finished状态改变的时候生成对isFinished这个key的通知。你不需要为非并发的操作重新实现这个属性。<br/>

`@property (readonly, getter=isReady) BOOL ready;`<br/>
一个标识操作现在是否可以执行的布尔值。<br/>
这个准备状态的值取决于他们依赖的操作和一些潜在的自定义条件。NSOperation类管理有关其他操作的依赖并且会基于他们之间的依赖影响这个准备值。<br/>
如果你想使用自定义条件定义你的操作对象的准备值，那么重新实现这个属性并返回一个准确反映接收者准备状态的值。如果你确实这么做了，你的自定义实现必须从super获取默认值，将这个准备状态值包含到这个属性的新值中。在你的自定义实现中，你必须在你的操作对象的准备状态发生变化的任何时候生成isReady键的KVO通知。想要获取更多关于生成KVO通知的信息，参见Key-Value Observing Programming Guide。<br/>


上面反复提到非并发操作和并发操作：这两者如何区别呢？
非并发操作：操作的代码执行全部在一个线程中
并发操作：操作的代码执行在多个线程中，例如在任务执行过程中向主线程传递数据。
举个最简单的例子，如果操作的start中定义了下面的任务，则是非并发操作：

```
- (void)start {
    // 开始执行在A线程
    // 之后的代码全部运行在A线程中
    NSUInteger a = 0;
    NSUInteger b = 1;
    NSUInteger c = a + b;
}
```

如果任务下面这样，则操作是并发操作：

```
- (void)start {
    // 在子线程A只执行
    NSTimeInterval timeInterval = 10.f;
    dispatch_async(dispatch_get_main_queue(), ^{
        // 这里的任务在主线程中执行
        [UIView animateWithDuration:timeInterval animations:^{
            
        }];
    });
}
```
或者像下面这样引发了新线程的创建，也是并发操作：

```
- (void)start {
    NSURL *url = [NSURL URLWithString:@"https://www.baidu.com"];
    [[[NSURLSession sharedSession] dataTaskWithURL:url] resume];
}
```

NSOperation有一个属性`isConcurrent`,它是一个用来标识操作是否异步执行它的任务的布尔值。这个属性现在已经弃用并使用`isAsynchronous`代替。对于与当前线程异步运行的操作来说，这个属性值为YES；对于与当前线程同步执行的操作来说，这个值为NO。默认的值是NO。如果要自定义实现一个NSOperation，那么这个值的的get是要重写的，以确保这个属性能正确地标识代码是同步还是异步执行，即是并发队列还是非并发队列。

可以看到并发队列与非并发队列的区别在于：当最后一行代码返回的时候能否标识着任务已经执行完成。例如上面的例子1：`NSUInteger c = a + b;`执行完返回时，代码已经全部执行完毕；而对于`dispatch_async(...)`返回时，因为是dispatch异步方法，所以只是将任务block提交到队列就返回了，所以并没有执行完毕；对于`[[[NSURLSession sharedSession] dataTaskWithURL:url] resume];`只是dataTask执行了开始的方法resume，并没有完成对url的访问。

另外强调的是：在并发操作中必须重写`executing`和`finished`两个属性，那么不重写操作的`executing`和`finished`两个属性会产生什么后果呢：
实际上，如果不重写start方法在返回的时候就将finished设置为YES，这样对于对于请求网络如果没有特别的要求还可以接受，对于动画的执行则会有意想不到的后果。

这个时候好像自定义NSOperation的注意事项都说的差不多了，但是还得提一下start方法和main方法的区别：
<h4 id="anchor1_3">start和main方法</h4>
<br/>

```
- (void)start;
```

开始operation的执行。<br/>
这个方法的默认实现会更新operation的执行状态，然后调用接收者的main方法。这个方法也会进行几个检查来确保操作可以正常运行。例如，如果接收者被取消了或者已经结束了，这个方法会简单地返回而不再调用main方法。(在In OS X v10.5中，如果操作已经结束了这个方法抛出一个异常)如果这个操作当前正在执行或者没有准备执行，这个方法会抛出一个NSInvalidArgumentException的异常。在OS X v10.5，这个方法会自动地捕获并且忽略任何你的main方法抛出的异常。在macOS 10.6以及之后的系统中, 异常被允许抛到这个方法之后。你应当永远不将异常抛到main方法外面。<br/>

>Note<br/>
>如果一个操作仍旧依赖着其他未执行完成的操作，不会被认为处于准备状态。<br/><br/>
>如果你正在实现一个并发的操作，你必须重写这个方法同时使用它来初始化你的操作。你的自定义实现在任何时候都绝对不能调用super。除了为你的任务配置执行环境之外，对于这个方法的实现必须追踪操作的状态，同时实现适当的状态改变。当操作开始执行直到之后的完成阶段，你应该生成对isExecuting和isFinished键的KVO通知。获取更多关于手动生成KVO通知的信息，参见Key-Value Observing Programming Guide。<br/><br/>
>你可以显式地调用这个方法如果你想手动执行你的操作。但是,对一个已经放到operation queue队列中的操作再执行这个方法或者在调用这个方法之后再将操作放入队列都是编程错误。一旦你添加一个操作到队列中，这个队列要对此操作承担所有责任。

```
- (void)main;
```

执行接收者的非并发任务。<br/>
这个方法的默认实现不会做任何事情。你应该重写这个方法来执行期望的任务。在你的实现中，不要调用super。这个方法会自动地执行在一个NSOperation提供的autorelease pool中，所以你不需要在你的实现中创建你自己的autorelease pool block。<br/>
如果你正在实现一个并发操作，你没必要重写这个方法，但是如果你打算在自定义的start方法中调用它就可以重写。

`总的来说：如果不是并发操作，那么可以只重写main方法;如果是并发操作，必须要重写start方法进行状态转变的控制。main方法只负责业务逻辑，start方法包含了操作状态相关的控制。`

<h4 id="anchor1_4">自定义NSOperation（OC、Swift）</h4>

要自定义一个NSOperation，你需要重写以下部分。不过对于非并发操作和并发操作这些要求是不同的：

| 方法 | 描述 |
| ----- | ----- |
| start | (必须)所有并发操作都必须覆盖这个方法，以自定义的实现替换默认行为。手动执行一个操作时，你会调用 start 方法。因此你对这个方法的实现是操作的起点，设置一个线程或其它执行环境，来执行你的任务。你的实现在任何时候都绝对不能调用 super。 |
| main |(可选)这个方法通常用来实现 operation 对象相关联的任务。尽管你可以在 start 方法中执行任务，使用 main 来实现任务可以让你的代码更加清晰地分离设置和任务代码 |
| isExecuting <br/> isFinished| (必须)并发操作负责设置自己的执行环境，并向外部 client 报告 执行环境的状态。因此并发操作必须维护某些状态信息，以知道是 否正在执行任务，是否已经完成任务。使用这两个方法报告自己的 状态。 这两个方法的实现必须能够在其它多个线程中同时调用。另外这些 方法报告的状态变化时，还需要为相应的 key path 产生适当的 KVO 通知。|
| isConcurrent | (必须)标识一个操作是否并发 operation，覆盖这个方法并返回 YES|


SDWebImage的`SDWebImageDownloaderOperation`的实现是一个很好的例子：

```objectivec
@interface SDWebImageDownloaderOperation ()

// ...

@property (assign, nonatomic, getter = isExecuting) BOOL executing;
@property (assign, nonatomic, getter = isFinished) BOOL finished;

// ...

@end

@implementation SDWebImageDownloaderOperation

@synthesize executing = _executing;
@synthesize finished = _finished;

- (id)initWithRequest:(NSURLRequest *)request
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock {
    if ((self = [super init])) {
        // ...
        _executing = NO;
        _finished = NO;
        // ...
    }
    return self;
}

- (void)start {
    @synchronized (self) {
        if (self.isCancelled) {
            self.finished = YES;
            [self reset];
            return;
        }
        // ...
        self.executing = YES;
        // ...
    }

    [self.connection start];

    if (self.connection) {
        // ...

        if (!self.isFinished) {
            // ...
        }
    }
    // ...
}

// ...

- (void)setFinished:(BOOL)finished {
    [self willChangeValueForKey:@"isFinished"];
    _finished = finished;
    [self didChangeValueForKey:@"isFinished"];
}

- (void)setExecuting:(BOOL)executing {
    [self willChangeValueForKey:@"isExecuting"];
    _executing = executing;
    [self didChangeValueForKey:@"isExecuting"];
}

- (BOOL)isConcurrent {
    return YES;
}

end
```

AFN2.x的`AFURLConnectionOperation`类则使用了自定义的值来表示isFinished和isExcuting的状态：

```objectivec
@interface AFURLConnectionOperation ()

// ...
@property (readwrite, nonatomic, assign) AFOperationState state;
// ...

@end

@implementation AFURLConnectionOperation

- (BOOL)isExecuting {
    return self.state == AFOperationExecutingState;
}

- (BOOL)isFinished {
    return self.state == AFOperationFinishedState;
}

- (BOOL)isConcurrent {
    return YES;
}

@end
```
并通过以下两个方法、函数的组合实现多所有状态值改变的通知发送：

```objectivec
static inline NSString * AFKeyPathFromOperationState(AFOperationState state) {
    switch (state) {
        case AFOperationReadyState:
            return @"isReady";
        case AFOperationExecutingState:
            return @"isExecuting";
        case AFOperationFinishedState:
            return @"isFinished";
        case AFOperationPausedState:
            return @"isPaused";
        default: {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunreachable-code"
            return @"state";
#pragma clang diagnostic pop
        }
    }
}

- (void)setState:(AFOperationState)state {
    if (!AFStateTransitionIsValid(self.state, state, [self isCancelled])) {
        return;
    }

    [self.lock lock];
    NSString *oldStateKey = AFKeyPathFromOperationState(self.state);
    NSString *newStateKey = AFKeyPathFromOperationState(state);

    [self willChangeValueForKey:newStateKey];
    [self willChangeValueForKey:oldStateKey];
    _state = state;
    [self didChangeValueForKey:oldStateKey];
    [self didChangeValueForKey:newStateKey];
    [self.lock unlock];
}
```

对于Swift中无法像OC一样再次使用@property声明父类中名字相同的属性(executing, finished)来实现对属性的覆盖, 可以像AFN那样实现新增一个值用于保存这两个状态的实际值，然后重写这两个状态的getter和setter，并在setter中生成通知，如下：

```swift
open class SteerableOperation: Operation {
    private var isFinishValue: Bool = false
    private var isExecutingValue: Bool = false
    override open var isFinished: Bool {
        get {
            return isFinishValue
        }
        set {
            self.willChangeValue(forKey: "isFinished")
            isFinishValue = newValue
            self.didChangeValue(forKey: "isFinished")
        }
    }
    override open var isExecuting: Bool {
        get {
            return isExecutingValue
        }
        set {
            self.willChangeValue(forKey: "isExecuting")
            isExecutingValue = newValue
            self.didChangeValue(forKey: "isExecuting")
        }
    }
}
```
<h4 id="anchor1_5">其他属性和方法</h4>

```- (void)addDependency:(NSOperation *)op;```<br/>
使接收者依赖指定操作的完成。<br/>
接收者认为直到所有它的依赖操作执行完毕才准备就绪。如果接收者已经正在执行任务，则添加依赖没有实际效果。这个方法会改变接收者的isReady属性和dependencies属性。<br/>
在一组操作之间创建一个循环的依赖是一个编程错误。这样做会导致操作之间的死锁，使程序freeze。<br/>
参数	<br/>
operation<br/>
接收者依赖的操作. 同样的依赖不应该再次添加给接受者，这样做的结果是未知的。<br/>

```- (void)removeDependency:(NSOperation *)op;```
移除接受者对指定操作的依赖。<br/>
这个方法会改变接收者的isReady和dependencies属性。<br/>
参数	<br/>
operation<br/>
要从接受者中移出的依赖操作。<br/>

```@property (readonly, copy) NSArray<NSOperation *> *dependencies;```<br/>
一个操作数组，这个操作数组中所有的操作必须在当前对象执行开始之前全部执行完毕。<br/>
这个属性包含了NSOperation对象数组。使用```addDependency:```向这个数组添加对象。<br/>
一个操作对象必须直到所有它的依赖操作执行完毕才能执行。操作不会在当执行完成时从这个列表移除。你可以使员工这个列表来追踪所有的依赖操作，包括那些已经执行完毕的操作。将操作从这个列表中移出的唯一方法是使用`removeDependency:`方法。<br/>

```@property NSOperationQueuePriority queuePriority;```<br/>
操作在操作队列中的执行优先级。<br/>
这个属性包含了操作的相对优先级。这个值用来影响操作出队和执行的顺序。返回值总是对应着一个预定义的常量。(若查看有效值的列表，参见NSOperationQueuePriority。)如果没有显式设置优先级，这个方法返回NSOperationQueuePriorityNormal。<br/>
你应该在仅当需要对没有依赖的操作之间的划分出相对优先级时使用priority值。Priority值不应该用来实现不同操作对象之间的依赖管理。如果你需要建立操作之间的依赖，使用`addDependency:`方法代替。<br/>
如果你试图指定一个与定义好的常量不同的优先级值，操作对象自动调整你指定的值，从第一个接近的定义的常量开始，趋向NSOperationQueuePriorityNormal优先级调整。例如，如果你指定的值是-10，这个操作会调整这个值为NSOperationQueuePriorityVeryLow常量。相似的，如果你指定+10,这个操作会调整值为NSOperationQueuePriorityVeryHigh常量。

```@property (nullable, copy) void (^completionBlock)(void)```<br/>
这个block在操作的主任务执行完成之后执行。<br/>
这个完成block没有任何参数和返回值。<br/>
这个完成block的执行环境是不确定的，但通常会是在子线程中。因此，你不应该使用这个block来做任何需要特定执行环境的工作。相反的，你应该避免在主线程或指定线程中才能进行的工作。例如，如果你自定义一个线程来协调操作的完成，你应该使用一个完成block来ping这个线程。<br/>
你提供的完成block会在当finish属性变成YES的时候执行。因为完成block会在操作已经完成任务这种语义环境下才执行，你必须使用完成block来执行一些认为是任务一部分的附加工作。一个finished属性值为YES的操作对象必须完成了它的所有的预定义的任务相关的工作。这个完成block应该被用来通知相关对象工作已经完成或者执行其他相关的任务，但又不是操作的实际任务。<br/>
一个finished操作要么是因为被取消了，要么是因为成功地执行的它的任务。你应该在编写block代码的时候考虑这个因素。相似地，你不应该做任何关于依赖操作成功执行完成的假设，因为他们可能是被取消了。<br/>
在iOS8、macOS 10.10或者之后的系统中，这个属性block开始制定之后在被设置为nil。<br/><br/>

```
- (void)waitUntilFinished
```
阻塞当前线程的执行，直到操作对象完成它的工作。<br/>
操作对象必须从不自己调用这个方法，并且要要避免与它提交到相同队列的其他操作调用。这样做会导致操作死锁。相反的，你的app的其他部分可以根据需要调用这个方法，来阻止其他任务的完成，直到目标操作对象完成。对另一个操作队列中的目标操作对象调用这个方法通常是安全的，尽管在每个操作相互等待时仍有可能造成死锁。<br/>
一个典型的使用这个方法的场景应该是，在操作初次创建的地方调用它。在把这个操作提交到队列之后，调用这个方法来等待操作执行完毕。<br/><br/>


 iOS8 新增的几个属性：

```
@property double threadPriority NS_DEPRECATED(10_6, 10_10, 4_0, 8_0);
@property NSQualityOfService qualityOfService NS_AVAILABLE(10_10, 8_0);
@property (nullable, copy) NSString *name NS_AVAILABLE(10_10, 8_0);
```

`@property double threadPriority NS_DEPRECATED(10_6, 10_10, 4_0, 8_0);`<br/>
线程的优先级，当正在执行操作时使用。<br/>

`@property NSQualityOfService qualityOfService NS_AVAILABLE(10_10, 8_0);`<br/>
为操作授权系统配额的重要性的相对级别。<br/>
服务等级会影响到操作获取如CPU时间、网络资源、磁盘资源等系统资源的优先级。有较高服务质量的的操作会获取到更高对于系统资源的优先级，因此他们可以更快地执行任务。使用服务等级来保证操作根据用户明确的指定的比其他任务重要的优先级做响应的调整。<br/>
这个属性对需要执行操作的最小服务等级的影响非常显著。这个属性的默认值是NSQualityOfServiceBackground，你应该竟可能不改变它。当改变了服务等级，使用合适的最小值执行相应的任务。例如，如果用户初始化一个任务并等待它执行完毕，对这个属性赋值为`NSQualityOfServiceUserInteractive`。如果资源充足，系统会给操作一个更高的服务等级。更多的信息，参见Energy Efficiency Guide for iOS Apps中的 <a href='https://developer.apple.com/library/content/documentation/Performance/Conceptual/EnergyGuide-iOS/PrioritizeWorkWithQoS.html#//apple_ref/doc/uid/TP40015243-CH39'>Prioritize Work with Quality of Service Classes</a>和 Energy Efficiency Guide for Mac Apps中的 <a href="https://developer.apple.com/library/content/documentation/Performance/Conceptual/power_efficiency_guidelines_osx/PrioritizeWorkAtTheTaskLevel.html#//apple_ref/doc/uid/TP40013929-CH35">Prioritize Work at the Task Level</a>。

`@property (nullable, copy) NSString *name NS_AVAILABLE(10_10, 8_0);`<br/>
操作的名字。<br/>
为操作对象赋值一个名字，用来在debug的时候识别。

<h2 id="anchor2_0">NSOperationQueue</h2>
NSOperationQueue类调节控制一组NSOperation对象的执行。在被添加到队列之后，操作保留在队列中，直到他被显示地取消或者执行完成任务。队列中还未执行的操作是根据优先级别和互操作对象依赖组织的，他们会依据这些相应地执行。一个应用可以创建多个操作队列来提交操作。<br/>
依赖为操作提供了一个绝对的执行顺序，即使这些操作在不同的操作队列中。一个操作在它的依赖操作执行完毕之前被认为是没有准备就绪的。对于已经准备就绪的操作，操作队列总是执行相对其他操作有最高优先级的操作。了解如何设置有限级别和依赖，参见NSOperation。<br/>
你不能直接移除一个已经添加到队列中的操作。一个操作会一直保留在队列中，直到得知任务执行完毕。完成任务不一定是操作完整地执行完了任务。一个操作也可以被取消。取消一个操作对象会使一个操作留在队列中，但是通知对象它被尽快中止。对于当前正在执行的操作，这意味着操作对象的任务代码必须检查取消状态，停止它正在进行的工作，同时标记自己为finished状态。对于在队列中还没有执行的操作来说，队列必须仍然调用operation对象的start方法来继续使操作可以处理取消事件和标记自己为finished状态。<br/>

>**Note**<br/>
>取消一个操作导致操作忽略它的任何依赖。这个行为会使队列尽可能快地执行队列的start方法。相应地，start方法将操作转为finished状态来使它能够从队列移除。<br/>
>操作队列通常提供会提供用来运行他们的操作的线程。操作队列使用libdispatch库(又称为 GCD)来初始化他们的操作的执行。结果是，操作总是执行在另外的线程，不论它们被设计成异步的还是同步的操作。获取更多关于使用操作队列的信息，参见<a href="https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091">Concurrency Programming Guide</a>


<h4>对操作的添加、获取</h4>
`- (void)addOperation:(NSOperation *)op;`<br/>
向接收者添加指定的操作对象。<br/>
一旦添加，指定的操作就保留在队列中，直到他完成执行。<br/>
一个操作对象最多能被添加到一个队列，如果添加一个其他队列中的操作将会抛出NSInvalidArgumentException异常。相似地，如果操作当前正在执行或者已经执行完成，这个方法会抛出NSInvalidArgumentException异常。<br/>
参数<br/>
要添加到队列中的操作。在内存管理的程序类型的应用中，这个对象会被操作队列持有。在垃圾回收类型的应用中， 队列强引用操作对象。<br/><br/>

`- (void)addOperations:(NSArray<NSOperation *> *)ops waitUntilFinished:(BOOL)wait`<br/>
向队列添加指定的操作数组。<br/>
一个操作对象最多能被添加到一个队列，如果当前正在执行或执行完成也不能被添加。如果参数中的操作有符合这种错误的情况的，这个方法抛出NSInvalidArgumentException异常。<br/>
一旦添加，指定的操作就保留在队列中，直到他finished方法返回YES。<br/>
参数<br/>
ops<br/>
你要添加到队列中的操作数组。<br/>
wait<br/>
如果传入YES，当前线程会被阻塞，直到所有的指定的操作执行完毕。如果传入NO，操作被添加队列后立即返回。<br/><br/>

`- (void)addOperationWithBlock:(void (^)(void))block`<br/>
将一个指定的block包装在操作对象中，然后将它添加到接收者。<br/>
这个方法向接收者添加一个block通过第一次将它包装在操作对象的方式。你不应该试图获取这个新创建的操作对象的引用或者判断它的类型信息。<br/><br/>

`@property (readonly, copy) NSArray<__kindof NSOperation *> *operations;`<br/>
队列中当前的操作数组。<br/>
这个数组包含这个0或多个NSOperation对象，他们的顺序为添加到队列中的顺序。这个顺序不一定会影响他们的执行顺序。<br/>
你可以使用这个属性来获取子给定时刻队列中的操作。操作会留在队列中，直到他们执行完成他们的任务。因此，这个数组包含的操作可能是正在执行的或者是等待执行的。这个列表也会包括了获取时正在执行但随后便执行完毕的操作。<br/>
你可以使用KVO来监控这个值的改变。配置一个观察者来监控操作队列的keyPath。<br/><br/>

`@property (readonly) NSUInteger operationCount;`<br/>
队列中当前的操作数目。<br/>
因为这个数目是根据队列中操作的完成状态改变的，所以这个值反应的是获取这个属性的时刻的瞬时值。这个值会因为你使用的时刻而不同。因此，不要使用这个值来进行对象枚举或者其他精确的计算。<br/>
你可以使用KVO来监控这个值的改变。<br/>

<h4>对操作的执行过程、相互间的协调控制</h4>

`@property NSInteger maxConcurrentOperationCount;`<br/>
在同一时间可以执行的最大操作数。<br/>
这个值仅仅会影响到当前队列在同一时间正在执行的操作。其他操作队列也会平行地执行他们的最大操作数。<br/>
减少并发操作数目不会影响任何当前正在执行的操作。指定这个值为NSOperationQueueDefaultMaxConcurrentOperationCount(推荐这样做)会使系统基于当前条件设置最大操作数。<br/>
默认的属性值是NSOperationQueueDefaultMaxConcurrentOperationCount。你可以通过KVO来监控它的变化。<br/><br/>

`@property (getter=isSuspended) BOOL suspended;`<br/>
一个标识是否在在分派操作的执行。<br/>
当这个属性值为NO，队列不断地开始准备执行的操作。设置为YES，将阻止任何操作开始，但是已经正在执行的操作会继续执行。你可以继续添加操作到队列中，但是这些操作直到你将这个属性改为NO时才会分派执行。<br/>
操作当且仅当执行完成时才会移出队列。但是，为了完成执行，一个操作必须首先要开始。因为一个挂起的队列不能开始新的操作，它不会移除任何在队列中还没有执行操作(包括取消了的操作)。<br/>
你可以使用KVO监控这个属性值。<br/>
默认的值是NO<br/><br/>

`- (void)cancelAllOperations;`<br/>
取消所有队列中和正在执行的操作。<br/>
这个方法对当前队列中的所有操作调用cancel方法。<br/>
取消操作不会自动地从队列中移出或者停止他们当前的执行。对于在队列中等待执行的操作来说，这个队列必须仍然在得知操作已经被取消或变为finished状态之前尝试执行操作。对于正处于执行中的操作来说，操作必须检查取消状态然后停止正在进行的工作，这样他可以变为finished状态。对于两种情况，一个完成的(或取消了的)操作在被移出队列之前，仍然有机会执行他们的completion block。<br/><br/>

`- (void)waitUntilAllOperationsAreFinished;`<br/>
阻塞当前线程，直到接收者队列中的操作和正在执行的操作都执行完毕。<br/>
当调用后，这个方法会阻塞当前线程并等待接收者的所有操作执行完毕。尽管当前的线程被阻塞了，但接收者继续启动队列中的操作和管理他们的执行。在这期间，当前线程不能向队列中添加操作，但是其他线程可以。一旦所有的等待的操作都执行完毕，这个方法返回。如果队列中没有操作，这个方法会立即返回。<br/><br/>

<h4>对NSOperationQueue对象本身的方便操作</h4>

`@property (nullable, copy) NSString *name;`<br/>
操作队列的名字。<br/>
名字提供了可以在运行时识别你的操作队列的方法。<br/>
可以使用名字在调试时提供额外的环境或者分析你的代码。<br/>
默认的属性值是“NSOperationQueue <id>”，这里的<id>是操作队列的内存地址。你可以通过KVO监控这个属性值。<br/><br/>

`@property (class, readonly, strong, nullable) NSOperationQueue *currentQueue`<br/>
返回操作启动当前操作的操作队列。<br/>
你可以在运行的操作对象里使用这个方法来获取开始它的操作队列的引用。若在运行操作环境之外调用这个方法通常会返回nil
返回值<br/>
启动操作的操作队列，当queue没有被指定时返回nil。<br/><br/>

`@property (class, readonly, strong) NSOperationQueue *mainQueue`<br/>
返回与主线程关联的操作队列。<br/>
这个返回的队列会在app的主线程执行操作。这个在主线程上的操作的执行与其他必须在主线程上的任务（如，事件服务和更新app的UI）是交叉进行的。队列在run loop common modes中执行这些操作，这个模式被描述为NSRunLoopCommonModes常量。队列的underlyingQueue属性值是主线程的dispatch queue;这个属性不能被设置为其他值。<br/><br/>

<h5>iOS8.0新增的</h5>
`@property NSQualityOfService qualityOfService`<br/>
队列申请操作执行使用的默认的服务级别。<br/>
这个属性指定了申请操作对象添加到队列中的服务等级。如果操作对象已经显式设置服务级别，这个值会被替代。默认的值依赖于你如何创建队列。对于自己创建的队列来说，默认的值是NSOperationQualityOfServiceBackground。对于mainQueue方法返回的队列来说，默认值是NSOperationQualityOfServiceUserInteractive并且不能被修改。<br/>
服务级别会影响对操作对象指定的获取CPU时间、网络资源、磁盘资源等资源的优先级。有高级服务质量的操作会对系统资源的使用有更高的优先级以使他们能更快地执行任务。使用服务等级来确保相对不重要的任务，有更高的优先级操作有相应的级别的用户请求响应。<br/><br/>

`@property (nullable, assign /* actually retain */) dispatch_queue_t underlyingQueue`<br/>
用来执行操作的dispatch队列。<br/>
默认的属性值是nil。你可以设置这个值为一个已经存在的dispatch queue来将操作入队，操作会和提交到这个diaptch queue中的block交叉执行。<br/>
这个属性值当且仅当没有操作在队列中时设置;当operationCount不为0设置这个值会引起NSInvalidArgumentException异常。这个属性的值一定不能是dispatch_get_main_queue的返回值。对underlying dispatch queue的服务质量的级别的设置会覆盖任何对操作队列的qualityOfService属性值的设置。<br/>
Note<br/>
OS_OBJECT_IS_OBJC为true，这个属性值自动持有赋值的queue。

<h2 id="anchor4_0">NSOperation与GCD对比</h2>
NSOperation+ NSOperaionQueue技术虽与GCD不同，但是却与之相关，开发者可以把操作以NSOperation子类的形式放在队列中，而这些操作也能够并发执行。其于GCD派发队列有相似之处，这并非巧合。“操作队列”(operation queue）在GCD之前就有了，其中某些设计原理因操作队列而流行，GCD就是基于这些原理构建的。实际上，从iOS4与Mac OSX 10.6开始，操作队列在底层是用GCD来实现的。

在两者的诸多差别中，首先要注意：GCD是纯C的API，而操作队列则是Objective-C的对象。在GCD中，任务用块来表示，而块是个轻量级数据结构。与之相反，“操作”(operation)则是个更为重量级的Objective-C对象。虽说如此，但GCD并不总是最佳方案。有时候采用对象所带来的开销微乎其微，使用完整对象所带来的好处反而大大超过其缺点。

用NSOperationQueue类的`addExecutionBlock:`方法搭配NSBlockOperation类来使用操作队列，其语法与纯GCD方式非常类似。使用NSOperation及NSOPerationQueue的好处如下：

>**取消某个操作**。如果使用操作队列，那么想要取消操作是很容易的。运行任务之前，可以在NSOperation对象上调用cancel方法，fairplay刚发会设置对象内的标志位，用以表明此任务不需执行，不过，已经启动的任务无法取消。若是不使用操作队列，而是把块安排到GCD队列，那就无法取消了。那套框架是“安排好任务之后就不管了”(fire and forget)。开发者可以在应用程序层自己来实现取消功能，不过这样做需要编写很多代码，而那些代码其实已经由操作队列实现好了。<br/><br/>
>**指定操作间的依赖关系**。一个操作可以依赖其他多个操作。开发者能够指定操作之间的依赖关系，使特定的操作必须在另外一个操作顺利执行完毕后方可执行。比方说，从服务器下载并处理文件的工作，可以使用操作来表示，而在处理其他文件之前，必须先下载“清单文件”（manifest file）。后续的下载操作，都要依赖于先下载清单文件这一操作，如果操作队列允许并发的话，那么后续的多个下载操作就可以同时执行，但前提是他们所依赖的那个清单文件下载操作已经执行完毕。<br/><br/>
>**通过键值观测机制监控NSOPeration对象的属性**。NSOperation对象有许多属性都适合通过KVO来监听，比如可以通过isCancelled属性来判断任何是否已取消，又比如可以通过isFinished属性来判断任务是否已完成。如果想在某个任务变更其状态时得到通知，或是想用比GCD更为精细的方式来控制所要执行的任务，那么KVO会很有用。<br/><br/>
>**指定操作的优先级**。操作的优先级表示此操作与队列中其他操作之间的优先关系。优先级好的操作先执行，优先级低的后执行。操作队列的调度算法(scheduling algorithm)虽“不透明”（opaque），但必然是经过一番深思熟虑才写成的。反之，GCD则没有直接实现此功能的办法。GDD的队列确实有优先级，不过那是针对整个队列来说的，而不是针对每个block来说的。而令开发者在GCD上自己来编写调度算法，又不太合适。因此在优先级这一点上，操作队列所提供的功能要比GCD更为便利。NSOperation对象也有“线程优先级”（thread priority），这决定了运行此操作的线程处在何种优先级上。用GCD也可以实现此功能，然而采用操作队列更简单，只需设置一个属性。<br/><br/>
>**重用NSOperation对象**。系统内置了一些NSOperaion的子类(比如NSBlockOperaion)供开发者调用，要是不想用这些固有子类的话，那就得自己来创建了。这些类就是普通的Objective-C对象，能够存放任何信息。对象在执行时可以充分利用存于其中的信息，而且还可以随意调用定义在类中的方法。这就比派发队列中哪些简单的block要强大徐国。这些NSOperation类可以在代码中多次使用，他们符合软件开发中的“不重复”(Dont't Repeat Yoursel, DRY)原则。

`总结来说：`NSOperation和NSOperationQueue与GCD的区别主要在于：<br/>
1.对程序员来说：NSOperation是使用面向对象的思想进行编程，使用时着重于对对象属性的调整已达到完成实现，这样的代码也是更加模块化和便于管理的；GCD使用的是纯C的函数调用，没有那么直观，虽然能实现大部分相似的功能，但是代码的离散程度太高，不便于管理。<br/>
2.对于具体的任务实现来说：NSOperation方式进行的是更细粒度的控制，具体体现在以下几方面：<br/>
①通过属性值的改变就实现了操作本身状态转变流程的操控：状态的暂停、取消、完成等行为控制和增加完成回调，这些功能贯穿了操作对象的生命周期；<br/>
②还可以细化对操作之间的控制：设置依赖、操作服务质量(优先级）；<br/>
③细化控制队列对于操作的影响：使用方法操作继续向指定队列添加指定的操作，并发数控制到具体值，取消所有操作等操作。<br/>
以上这些细粒度的操作都是使用GCD较难实现的。<br/>
GCD更倾向于粗粒度，它关注于三个点①任务②将任务放入队列③等待完成，对于这中间的过程都不再参与或者说不再感兴趣了,尽管使用barrier、group、semaphore这些技术和dispatch_suspend与dispatch_resume函数可以进行对执行顺序或执行流程的控制，但都是基于队列的宏观控制，无法细微到任务内部。
<br/><br/>

正如大家所见，操作队列中有很多地方胜过派发队列。操作队列提供了多种执行任务的方式，而且都是写好了的，直接就能使用。开发者不用再编写负责的调度器，也不用自己来实现取消操作或指定操作优先级的功能呢，这些事情操作队列都已经实现好了。
有一个API选用了操作队列而非派发队列，这就是NSNotificationCenter，开发者可通过其中的方法来注册监听器，以便在发生相关事件时得到通知，而这个方法接受的参数是block，而不是选择子。方法原型如下：

```objectivec
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block NS_AVAILABLE(10_6, 4_0);
```
本来这个方法也可以不适用操作队列，而是把处理通知时间所用的block安排在派发队列里。但实际上并没有这样做，其设计者显然适用了高层的Objective-C API。在这种情况下，两套方案的运行效率没多大差距。设计这个方法的人可能不想使用派发队列，因为那样做将依赖于GCD，而这种依赖没有必要，block本身和GCD无关，所以如果仅使用block的话，就不会引入对GCD的依赖了。也有可能是编写这个方法的人想全部用Objective-C来描述，而不想使用纯C的东西。

经常会有人说：应该尽可能选用高层API，只在确有必要时才来求助于底层。笔者也统一这个说法，但我并不盲从。某些功能确实可以用高层的Objective-C方法来做，但这并不等于说它就一定比底层实现方案好。要想确定哪种方案更佳，最好还是测试一下性能。