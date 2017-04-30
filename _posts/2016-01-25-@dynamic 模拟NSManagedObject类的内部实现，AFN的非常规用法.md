---
layout: cnblog_post
title:  "@dynamic 模拟NSManagedObject类的内部实现，AFN的非常规用法"
date:   2016-01-25 13:50:39
categories: iOS
---
#### &#64;property和&#64;synthesize复习
@property生成`setter和getter的声明`，同时生成属性对应的成员变量，并且前面加一个下划线`_`。如果将getter和setter的实现同时重写之后，它不会帮助生成属性对应的变量名。 (TestOne.m)这种写法，无法编译通过，因为_name已经不存在。

```objectivec
@interface TestOne : NSObject

@property (nonatomic, copy) NSString *name;

@end


@implementation TestOne

- (void)setName:(NSString *)name {
    _name = name; // Use of undeclared identifier '_name'
}

- (NSString *)name {
    return _name; // Use of undeclared identifier '_name'
}

@end
```

比较特殊的一种情况是readonly的属性只会帮助生成getter的声明和变量。，如果当getter被重写之后，成员变量也不存在。

```objectivec
@interface TestTwo : NSObject

@property (readonly, nonatomic, copy) NSString *name;

@end


@implementation TestTwo

- (NSString *)name {
    return _name; // Use of undeclared identifier '_name'
}

@end
```
如果在.h使用&#64;property声明了方法属性，又想在.m重写方法怎么办呢。通常我们会使用&#64;synthesize.

&#64;synthesize是按照系统默认规则帮助生成`getter和setter的实现`，同时生成一个紧跟在关键字`@synthesize`后面的属性对应的成员变量，还可以更改属性对应的成员变量的名字。
同时如果使用了&#64;synthesize之后，还可以继续重写getter和setter的实现，同时它帮助生成的成员变量依然存在。下面的代码是没有任何问题的。

```objectivec
@interface TestThree : NSObject

@property (nonatomic, copy) NSString *name;

@end

@implementation TestThree
@synthesize name = _name; // 使用 @synthesize name 生成的成员变量是'name'

- (void)setName:(NSString *)name {
    _name = name;
}

- (NSString *)name {
    return _name;
}

@end
```
这时候&#64;property的作用仅仅是相当于对外声明了方法原型：

```objectivec
- (void)setName:(NSString *)name;

- (NSString *)name;
```
但是不可以将.h中的`@property (nonatomic, copy) NSString *name;`替换为上面两句方法声明，因为&#64;synthesize使用的前提是使用&#64;property声明过这个属性。

对于readonly的属性，同样可以使用&#64;synthesize关键字找到属性对应的成员变量名，然后重写getter就没有问题了。

### &#64;dynamic

我们都知道对应&#64;dynamic修饰的属性，需要手动实现这个属性的getter和setter，否则虽然编译阶段能够通过，但是运行时会造成崩溃，错误信息为没有指定的方法。编译时没问题，运行时才执行相应的方法，这就是所谓的动态绑定。
例如下面的代码

```objectivec
@interface TestFour : NSObject

@property (nonatomic, copy) NSString *name;

@end

@implementation TestFour

@dynamic name;

@end

// 在main.m中
TestFour *testFour = [[TestFour alloc] init];
testFour.name = @"Mike"; //已经奔溃 [TestFour setName:]: unrecognized selector sent to instance
NSLog(@"%@", testFour.name);
```
这里就体现出了&#64;synthesize和&#64;dynamic的区别:
&#64;synthesize的语义是如果你没有手动实现setter方法和getter方法，那么编译器会自动为你加上这两个方法的实现。
&#64;dynamic告诉编译器,属性的setter与getter方法需要实现，但是不会自动帮助生成。(当然对于readonly的属性只需提供getter实现即可)

&#64;dynamic最常用的使用是在NSManagedObject中，此时不需要显示编程setter和getter方法。原因是：其getter和setter方法会在运行时动态创建，由Core Data框架为此类属性生成存取方法。那么CoreData究竟如何帮助NSManagedObject的子类生成子类属性的getter和setter实现的呢。我们大体可以模拟一下这个过程：

### 模拟CoreData NSManagedObject类的底层实现
根据OC运行时的特性，当子类的方法没有实现的实现，会去寻找父类的方法实现，为了语义上更好理解我使用Person和它的父类Super用来测试：(这其中Person模拟的是NSManagedObject的子类)

```objectivec
@interface Person : Super
@property (nonatomic, copy) NSString *name;
@end

@implementation Person
@dynamic name;
@end

@interface Super : NSObject
@end

@implementation Super
- (void)setName:(NSString *)name {
    NSLog(@"执行了Super的setName:");
    // setName ....
}
- (NSString *)name {
    NSLog(@"执行了Super的name");
    return @"Mike在Super中设置";
}
@end

Person *person = [[Person alloc] init];
person.name = @"Mike";
NSLog(@"%@", person.name);
```
因此上面的程序执行结果为：

```objectivec
执行了Super的setName:
执行了Super的name
Mike在Super中设置
```
那么问题就来了NSManagedObject是如何截获它子类的所有属性的getter和setter方法的调用，并完成代码实现的，毕竟它不会傻乎乎地把所有的属性都写一个getter和setter方法吧。虽然不知道它的具体实现方法，但是可以模拟一下这个过程。

这里提供一种利用消息转发机制来实现，主要用到两个方法`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`和`- (void)forwardInvocation:(NSInvocation *)anInvocation`:

`methodSignatureForSelector:`方法用来返回参数指定的SEL的指定的方法签名(方法签名是各种编程语言中通用的概念，主要是对方法的返回值类型、参数类型和参数的个数的描述，函数重载的概念就是同名方法具有不同的方法签名)。对于`- (void)setName:(NSString *)name`方法的方法签名就是：返回值为void类型参数个数为1参数类型为NSString。用自然语言描述是这样，如果用OC的方法编码描述就是：&#64;"v@:@"(为什么这样写：可参考我之前关于Runtime的日志).`methodSignatureForSelector:`调用的时机为：对象进行任何方法调用都会经过这个方法的过滤。

要注意的是，如果通过通过调用这个方法之后没有得到参数Selector对应的方法签名，那么就会直接导致奔溃，错误为：`unrecognized selector sent to xxx`。而如果找到了方法的签名，则会继续调用`forwardInvocation:`以求通过消息转发的方式找到方法实现。

`forwardInvocation:`方法用来实现消息转发，也就是在它的内部处理一些接收到的方法的实现细节，或者将实现的细节交给其他对象。有关这个方法apple有一段长长的文档，我简单地翻译了一下：

>当一个对象发送了一个消息，它却没有相应的实现方法，runtime系统会给这个接受者一个机会来委派这个消息给另一个接受者。它通过创建一个NSInvocation对象委派这个消息，这个对象代表着这个消息同时会发送给接受者一个包含着这个NSInvocation对象作为参数的forwardInvocation:消息。然后，接收者的forwardInvocation:方法就可以选择转发这个消息给另一个对象。（如果那个对象也不对这个消息响应，它也会给一个机会转发它）。
因此，forwardInvocation:方法允许一个对象为某些消息建立与对这个对象有影响的其它对象的关系。比如，转发对象能"继承"一些它要转发的对象的特性。<br/>
> **IMPORTANT**<br/>
>想要响应你的对象自己不能识别的方法，除了`forwardInvocation:`外,你还必须重写`methodSignatureForSelector:`方法.消息转发机制使用从`methodSignatureForSelector:`获得的信息来创建被转发的NSInvocation对象。 重写方法时，必须为指定的selector提供一个合适的方法签名，。要么是通过预先格式化的签名，要么就从另一个对象中获取。

>一个`forwardInvocation:`方法的实现包括两项任务：
>设置一个能够在anInvocation中响应消息编码的对象，对所有的消息而言，这个对象不必相同。
>用anInvocation对那个对象发送消息。anInvocation会持有结果值，runtime系统会把这个结果值取出并传递给最初的发送者。

这里有一个`-methodSignatureForSelector:`和`-forwardInvocation:`方法的使用示例
例如在Super类中的实现修改为：

```objectivec
@implementation Super
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSString *selName = NSStringFromSelector(aSelector);
    if ([selName isEqualToString:@"tenAdd:"]) { // 过滤`tenAdd:`方法，指定它的方法签名
        return [NSMethodSignature signatureWithObjCTypes:"i@:@"]; // 'i'：int， '@:'：OC方法，'@'：对象类型
    } else {
        return [super methodSignatureForSelector:aSelector];
    }
    
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSString *selName = NSStringFromSelector([anInvocation selector]);
    if ([selName isEqualToString:@"tenAdd:"]) { // 在这里实现`tenAdd:`方法
        NSNumber *arg;
        [anInvocation getArgument:&arg atIndex:2]; // 0:self  1:_cmd  2:@(3)
        NSLog(@"%@", NSStringFromClass([[anInvocation target] class])); // Person
        int result = 10 + [arg intValue];
        [anInvocation setReturnValue:&result];
    } else {
        [super forwardInvocation:anInvocation];
    }
}
@end
```
然后进行如下的测试

```objectivec
Person *person = [[Person alloc] init];
// 在子类(Person类)中调用`tenAdd:`方法    
NSLog(@"%zd", [person performSelector:@selector(tenAdd:) withObject:@(3)]); // 13
```
在上面的Super实现中会接收到来自子类的对于各种方法的调用消息，使用`methodSignatureForSelector:(SEL)aSelector`获取一个方法的签名,如果发现方法名是&#64;"tenAdd:"那么它的签名就是"i@:@"，在方法`forwardInvocation:`中实现了&#64;"tenAdd:"方法的内容用10和传递过来的参数相加，并将计算的结果作为这个方法调用(invocation类型)的返回值。

理清这两个方法的用法，可以来改造Super类来模拟NSManagedObject类的实现了，使用静态的可变数组tableData代表了数据库表中已经存在的数据，columnNames代表所有的列名的集合，对应着每个Person类的属性名。通过person的id来获取一条记录所在的行，为了方便，在本例中所有的行号都传递0。代码如下：

```objectivec
@implementation Super

static NSMutableArray *tableData = nil;
static NSArray *columnNames = nil;

+ (void)initialize {
    [super initialize];
    
    tableData = [NSMutableArray array];
    [tableData addObject:[NSMutableDictionary dictionaryWithDictionary:@{@"name":@"Mike"}]];
    [tableData addObject:[NSMutableDictionary dictionaryWithDictionary:@{@"name":@"John"}]];
    
    columnNames = @[@"name"];
}

- (NSDictionary *)rowDataInTableWithRowId:(NSInteger)rowId {
    return tableData[rowId];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSString *selName = NSStringFromSelector(aSelector);
    if ([selName rangeOfString:@"set"].location == 0) { // 处理setter
        return [NSMethodSignature signatureWithObjCTypes:"v@:@"];
    } else if ([columnNames containsObject:selName.description]){ // 处理getter
        return [NSMethodSignature signatureWithObjCTypes:"@@:"];
    } else {
        return [super methodSignatureForSelector:aSelector];
    }
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSString *selName = NSStringFromSelector([anInvocation selector]);
    if ([selName rangeOfString:@"set"].location == 0) { // setter
        NSString *propertyName = [selName substringWithRange:NSMakeRange(3, [selName length]- 3)];
        propertyName = [propertyName stringByReplacingCharactersInRange:NSMakeRange(0,1) withString:[[propertyName substringToIndex:1] lowercaseString]];
        propertyName = [propertyName characterAtIndex:propertyName.length - 1] == ':' ? [propertyName substringToIndex:propertyName.length - 1] : propertyName;
        NSString *arg;
        [anInvocation getArgument:&arg atIndex:2];
        id obj = [self rowDataInTableWithRowId:0]; // 这里始终传入0，代表着表中的第一条数据。实际上rowId应该从[anInvocation target]中获取， 也就是从Person对象中获取。
        [obj setObject:arg forKey:propertyName];
    } else if ([columnNames containsObject:selName.description]){ // getter
        id obj = [self rowDataInTableWithRowId:0];
        id value = [obj objectForKey:selName.description];
        [anInvocation setReturnValue:&value]; // 设置返回值
    } else {
        [super forwardInvocation:anInvocation];
    }
}

@end
```
在main.m中进行测试，

```objectivec
Person *person = [[Person alloc] init];
person.name = @"修改后的Mike";

NSLog(@"%@", person.name);  // 修改后的Mike
```
正式如我们期望的结果。这样我们没有改变Person类的任何代码，只是修改了它的父类，便将Person类所有属性的getter和setter全都实现了。而且这是一个通用的代码，即使再给Person类增加别的属性，也没有任何问题。

### AFN一些&#64;dynamic非常规代码
最近看到了AFN中使用&#64;dynamic的代码，但是这些并不是我们常用的代码。如在(2.x版本)AFHTTPRequestOperation.m中：

```objectivec
@interface AFURLConnectionOperation ()
@property (readwrite, nonatomic, strong) NSURLRequest *request;
@property (readwrite, nonatomic, strong) NSURLResponse *response;
@end

@interface AFHTTPRequestOperation ()
@property (readwrite, nonatomic, strong) NSHTTPURLResponse *response;
@property (readwrite, nonatomic, strong) id responseObject;
@property (readwrite, nonatomic, strong) NSError *responseSerializationError;
@property (readwrite, nonatomic, strong) NSRecursiveLock *lock;
@end

@implementation AFHTTPRequestOperation
@dynamic response;
@dynamic lock;
// ...
// ...
@end
```
AFHTTPRequestOperation本来就是AFURLConnectionOperation的子类，可是在这里竟然又为它加了一个拓展，对于response和lock属性本来也是从AFURLConnectionOperation继承而来，但在子类中又使用了扩展重新定义了一下，而在AFURLConnectionOperation.m中他们也有同样的定义：

在.h中

```objectivec
@interface AFURLConnectionOperation
@property (readonly, nonatomic, strong) NSURLResponse *response;
// ...
@end
```

在AFURLConnectionOperation.m中AFURLConnectionOperation的扩展中

```objectivec
@interface AFURLConnectionOperation ()
@property (readwrite, nonatomic, strong) NSRecursiveLock *lock;
@property (readwrite, nonatomic, strong) NSURLResponse *response;
// ...
@end
```

在AFHTTPRequestOperation.h中

```objectivec
@interface AFURLConnectionOperation : AFURLConnectionOperation
@property (readonly, nonatomic, strong) NSHTTPURLResponse *response;
// ...
@end
```

对此我做了如下分析，(AFURLConnectionOperation简称为URLCO, AFHTTPRequestOperation简称为HTTPRO)

首先为什么URLCO中已经定义了response属性，为什么子类HTTPRO中还要定义，这个是很好理解的：

URLCO中的reponse是NSURLResponse类型，而HTTPRO中的response是NSHTTPURLResponse类型，对于使用HTTPRO的response属性的情况，省去了诸如类型强转等重复性的操作，面向接口编程的原则中有`抽象类的属性使用抽象类`的思想，而这里是`具体的类的属性使用具体的类`。

为什么要在扩展中重新定义属性response?

实际上这里的response定义是和&#64;dynamic配合使用的：

首先，response已经在URLCO.h中定义为readonly如果要在URLCO.m中为成员变量赋值，或者修改其值，要使用&#64;synthesize生成成员变量的getter和setter同时保证成员变量依然存在。AFN并没有使用这种方法，而是通过在扩展中增加readwrite的关键字再次定义这样同样使getter和setter可用，属性也依然存在，而在URLCO.m中不需要&#64;dynamic修饰属性，因为已经可以确保getter和setter的实现。

再者，在子类HTTPRO中，同样使用上面的方法依然使得在HTTPRO.m中response的getter和setter可用了，但是在getter和setter的实现内部与父类URLCO中的reponse属性失去了联系，这是后配合使用&#64;dynamic response使得本来的setter和gette实现变得不可用，但是对于调用依然没有编译期间的错误，而在运行阶段，实际是调用了父类URLCO中的gette和setter，继续将方法调用向上传递。

而对于为什么要在HTTPRO中再次定义父类的扩展？

对于request属性来说是有用的，因为子类中没有定义过request，但是对response来说是没有什么意义的，个人觉得这句可以直接删掉。