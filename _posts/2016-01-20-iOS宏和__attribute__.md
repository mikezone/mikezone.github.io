---
layout: cnblog_post
title:  "iOS宏和__attribute__"
date:   2016-01-20 13:50:39
categories: iOS
---
**本文目录**<br/>
1. iOS宏的经典用法<br/>
2. Apple的习惯<br/>
3. \_\_attribute\_\_<br/>

### iOS宏的经典用法

#### 1.常量宏、表达式宏
```objectivec
#define kTabBarH (49.0f)
#define kScreenH [UIScreen mainScreen].bounds.size.height
#define isScreenWidthEqual320 (fabs([UIScreen mainScreen].bounds.size.width - 320) < DBL_EPSILON)
```

#### 2.带参数的宏

```objectivec
// 例子1：同样是一个表达式
#define isNilOrNull(obj) (obj == nil || [obj isEqual:[NSNull null]])
// 例子2：也可以没有参数值 使用的时候要加上"()",是在使用的时候单独成为一行语句的宏
#define MKAssertMainThread() NSAssert([NSThread isMainThread], @"This method must be called on the main thread")
```
#### 3.函数宏(是一个没有返回值的代码块,通常当做一行语句使用) 

```objectivec
// do {} while(0) 一般是没有返回值的，仅仅是代码块而已,使用do {} while(0)意思是让它执行一次，相当于直接调用代码块
#define NSAssert(condition, desc, ...)	\
    do {				\
	__PRAGMA_PUSH_NO_EXTRA_ARG_WARNINGS \
	if (!(condition)) {		\
            NSString *__assert_file__ = [NSString stringWithUTF8String:__FILE__]; \
            __assert_file__ = __assert_file__ ? __assert_file__ : @"<Unknown File>"; \
	    [[NSAssertionHandler currentHandler] handleFailureInMethod:_cmd \
		object:self file:__assert_file__ \
	    	lineNumber:__LINE__ description:(desc), ##__VA_ARGS__]; \
	}				\
        __PRAGMA_POP_NO_EXTRA_ARG_WARNINGS \
    } while(0)
```

#### 4.内联函数 (一般有返回值)
```objectivec
CG_INLINE CGRect
CGRectMake(CGFloat x, CGFloat y, CGFloat width, CGFloat height) {
  CGRect rect;
  rect.origin.x = x; rect.origin.y = y;
  rect.size.width = width; rect.size.height = height;
  return rect;
}

NS_INLINE BOOL 
MKIsLeapYear() {
    NSCalendar *calendar = [NSCalendar currentCalendar];
    NSInteger year = [calendar component:NSCalendarUnitYear fromDate:[NSDate new]];
    return year % 100 == 0 ? year % 400 == 0 : year % 4 == 0;
}
```

#### 5.变参宏 函数可变参数
```objectivec
// 例如可以像如下几种方式使用NSAssert这个宏
NSAssert(10 > 11, @"错误1:%@", @"err desc 1");
NSAssert(10 > 11, @"错误1:%@\n错误2:%@", @"err desc 1", @"err desc 2");
```
关于宏定义中的#和##的说明
`#`有两个作用：
1.将变量直接转化为相应字面量的C语言字符串 如a=10 #a会转化为"10"
2.连接两个C字符串，使用如下

```objectivec
#define toString(a) #a  // 转为字符串
#define printSquare(x) printf("the square of "  #x " is %d.\n",(x)*(x)) // 连接字符串
```
使用如下

```objectivec
printf("%s\n", toString(abc)); // abc
printSquare(3); // the square of 3 is 9.
```
`##`的常见用处是连接，它会将在它之前的语句、表达式等和它之后的语句表达式等直接连接。这里有个特殊的规则，在逗号和__VA_ARGS__之间的双井号，除了拼接前后文本之外，还有一个功能，那就是如果后方文本为空，那么它会将前面一个逗号吃掉。这个特性当且仅当上面说的条件成立时才会生效，因此可以说是特例。

```objectivec
// 用法1：代码和表达式直接连接
#define combine(a, b) a##b 
#define exp(a, b) (long)(a##e##b)  
// 用法2：给表达式加前缀
#define addPrefixForVar(a) mk_##a 
// 用法3，用于连接printf-like方法的format语句和可变参数
#define format(format, ...) [NSString stringWithFormat:format, ##__VA_ARGS__]
```
使用示例如下：

```objectivec
NSLog(@"%zd", combine(10, 556)); // 10556
NSLog(@"%f", combine(10, .556)); // 10.556000
NSLog(@"%tu", exp(2, 3)); // 2000
    
// int x = 10;
int mk_x = 30;
NSLog(@"%d", addPrefixForVar(x));  // 30
    
NSLog(@"%@", format(@"%@_%@", @"good", @"morning"));  // good_morning
NSLog(@"%@", format(@"hello"));  // hello
```
对于使用`##`连接可变参数的用法，如果不加`##`会导致编译无法通过，这是因为： 这个##在逗号和__VA_ARGS__之间，后面的值不存在的时候 会忽略## 前面的`,`
也就是说：
当有##   调用format(format) 替换为 [NSString stringWithFormat:format]
当没有## 调用format(format) 替换为 [NSString stringWithFormat:format ,]

还有一点要注意的是：`#`和`##`只能用在宏定义中，而不能使用在函数或者方法中。

### Apple使用宏的一些习惯

##### 1.类的声明除了引用的其他头文件 用下面一对宏标记
```objectivec
#define NS_ASSUME_NONNULL_BEGIN _Pragma("clang assume_nonnull begin")
#define NS_ASSUME_NONNULL_END   _Pragma("clang assume_nonnull end")
```
也可以使用下面的两句

```objectivec
#pragma clang assume_nonull begin
#pragma clang assume_nonull end
```
告诉clang编译器这两者之间内容非空

##### 2. 在类声明前，方法声明后都有可用性的标记宏
例如

```objectivec
NS_CLASS_AVAILABLE   // 类之前
NS_AVAILABLE(_mac, _ios) // 方法之后
NS_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, ...) // 方法之后
OBJC_SWIFT_UNAVAILABLE("use 'xxx' instead") // 针对swift特殊说明取代它的类和方法
```
不同功能的类之前的可用性标记不同，例如NSAutoreleasePool之前

```objectivec
NS_AUTOMATED_REFCOUNT_UNAVAILABLE
@interface NSAutoreleasePool : NSObject{ ...
```
对于这些标记究竟干了些什么，请看本文第三部分__attribute__。

##### 3.嵌套的宏(经常成对使用)

```objectivec
#define NS_DURING		@try {
#define NS_HANDLER		} @catch (NSException *localException) {
#define NS_ENDHANDLER		}
#define NS_VALUERETURN(v,t)	return (v)
#define NS_VOIDRETURN		return


#if __clang__
#define __PRAGMA_PUSH_NO_EXTRA_ARG_WARNINGS \
    _Pragma("clang diagnostic push") \
    _Pragma("clang diagnostic ignored \"-Wformat-extra-args\"")

#define __PRAGMA_POP_NO_EXTRA_ARG_WARNINGS _Pragma("clang diagnostic pop")
#else
#define __PRAGMA_PUSH_NO_EXTRA_ARG_WARNINGS
#define __PRAGMA_POP_NO_EXTRA_ARG_WARNINGS
#endif
```

##### 4.经常使用宏来配合条件编译 (为了适配当前的硬件环境，系统环境、语言环境、编译环境)
```objectivec
#if !defined(NS_BLOCKS_AVAILABLE)
    #if __BLOCKS__ && (MAC_OS_X_VERSION_10_6 <= MAC_OS_X_VERSION_MAX_ALLOWED || __IPHONE_4_0 <= __IPHONE_OS_VERSION_MAX_ALLOWED)
        #define NS_BLOCKS_AVAILABLE 1
    #else
        #define NS_BLOCKS_AVAILABLE 0
    #endif
#endif


#if !defined(NS_NONATOMIC_IOSONLY)
    #if TARGET_OS_IPHONE
	#define NS_NONATOMIC_IOSONLY nonatomic
    #else
        #if __has_feature(objc_property_explicit_atomic)
            #define NS_NONATOMIC_IOSONLY atomic
        #else
            #define NS_NONATOMIC_IOSONLY
        #endif
    #endif
#endif


#if defined(__cplusplus)
#define FOUNDATION_EXTERN extern "C"
#else
#define FOUNDATION_EXTERN extern
#endif
```

##### 5. 由\_\_attribute\_\_定义的宏处处可见。
(本文在第三部分详细介绍\_\_attribute\_\_.).
例如下面这些常见的宏都属于这种：

```objectivec
// NSLog 方法后面使用的宏
#define NS_FORMAT_FUNCTION(F,A) __attribute__((format(__NSString__, F, A)))


//  这些都是在<Foundation/NSObjCRuntime.h>中定义的宏
#define NS_RETURNS_RETAINED __attribute__((ns_returns_retained))
#define NS_RETURNS_NOT_RETAINED __attribute__((ns_returns_not_retained))
#define NS_RETURNS_INNER_POINTER __attribute__((objc_returns_inner_pointer))
#define NS_AUTOMATED_REFCOUNT_UNAVAILABLE __attribute__((unavailable("not available in automatic reference counting mode")))
#define NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE __attribute__((objc_arc_weak_reference_unavailable))
#define NS_REQUIRES_PROPERTY_DEFINITIONS __attribute__((objc_requires_property_definitions)) 
#define NS_REPLACES_RECEIVER __attribute__((ns_consumes_self)) NS_RETURNS_RETAINED
#define NS_RELEASES_ARGUMENT __attribute__((ns_consumed))
#define NS_VALID_UNTIL_END_OF_SCOPE __attribute__((objc_precise_lifetime))
#define NS_ROOT_CLASS __attribute__((objc_root_class))
#define NS_REQUIRES_SUPER __attribute__((objc_requires_super))
#define NS_DESIGNATED_INITIALIZER __attribute__((objc_designated_initializer))
#define NS_PROTOCOL_REQUIRES_EXPLICIT_IMPLEMENTATION __attribute__((objc_protocol_requires_explicit_implementation))
#define NS_CLASS_AVAILABLE(_mac, _ios) __attribute__((visibility("default"))) NS_AVAILABLE(_mac, _ios)
#define NS_CLASS_DEPRECATED(_mac, _macDep, _ios, _iosDep, ...) __attribute__((visibility("default"))) NS_DEPRECATED(_mac, _macDep, _ios, _iosDep, __VA_ARGS__)
#define NS_SWIFT_NOTHROW __attribute__((swift_error(none)))
#define NS_INLINE static __inline__ __attribute__((always_inline))
#define NS_REQUIRES_NIL_TERMINATION __attribute__((sentinel(0,1)))
```
而我们经常看到的这样的几个宏

```objectivec
#define NS_AVAILABLE(_mac, _ios) CF_AVAILABLE(_mac, _ios)
#define NS_AVAILABLE_MAC(_mac) CF_AVAILABLE_MAC(_mac)
#define NS_AVAILABLE_IOS(_ios) CF_AVAILABLE_IOS(_ios)
```
这三个其实被定义在了<CoreFoundation/CFAvailability.h>中

```objective
#define CF_AVAILABLE(_mac, _ios) __attribute__((availability(macosx,introduced=_mac)))
#define CF_AVAILABLE_MAC(_mac) __attribute__((availability(macosx,introduced=_mac)))
#define CF_AVAILABLE_IOS(_ios) __attribute__((availability(macosx,unavailable)))
```

总之你会发现只要是不具备表达式或者代码块功能的宏,绝大多数都是由\_\_attribute\_\_定义的，那么\_\_attribute\_\_到底是什么呢，请继续：

### \_\_attribute\_\_

GNU C 的一大特色就是\_\_attribute\_\_机制。\_\_attribute\_\_可以设置函数属性（Function Attribute ）、变量属性（Variable Attribute ）和类型属性（Type Attribute ）。
\_\_attribute\_\_的书写方式是：\_\_attribute\_\_后面会紧跟一对原括弧，括弧里面是相应的__attribute__参数,格式如：
\_\_attribute\_\_((attribute-list))
其位置约束为：`放于声明的尾部“;” 之前`。

那么现在的问题就是什么情况下使用\_\_attribute\_\_，以及如何设置参数attribute-list，主要分为三种情况：
函数属性（Function Attribute ）、变量属性（Variable Attribute ）和类型属性（Type Attribute ）。
#### 1.函数属性（Function Attribute ）
 函数属性可以帮助开发者把一些特性添加到函数声明中，从而可以使编译器在错误检查方面的功能更强大。\_\_attribute\_\_机制也很容易同非GNU应用程序做到兼容之功效。
 
#####  \_\_attribute\_\_((format))
例如NSLog声明中使用到的宏：

```objectivec
#define NS_FORMAT_FUNCTION(F,A) __attribute__((format(__NSString__, F, A)))
```
\_\_attribute\_\_((format(\_\_NSString\_\_, F, A)))

该\_\_attribute\_\_属性可以给被声明的函数加上类似printf或者scanf的特征，它可以使编译器检查函数声明和函数实际调用参数之间的格式化字符串是否匹配。该功能十分有用，尤其是处理一些很难发现的bug。对于format参数的使用如下
format (archetype, string-index, first-to-check)
第一参数需要传递“archetype”指定是哪种风格；“string-index”指定传入函数的第几个参数是格式化字符串；“first-to-check”指定第一个可变参数所在的索引。
如NSLog对这个宏的使用如下：

```objectivec
FOUNDATION_EXPORT void NSLog(NSString *format, ...) NS_FORMAT_FUNCTION(1,2);
```
那么它如何进行语法检查，看下面的例子：

```objectivec
extern void myPrint(const char *format,...)__attribute__((format(printf,1,2)));

void test() {
    myPrint("i=%d\n",6);
    myPrint("i=%s\n",6);
    myPrint("i=%s\n","abc");
    myPrint("%s,%d,%d\n",1,2);
}
```
使用clang命令编译一下：

```sh
clang -c main.m
```
结果会出现下面的警告

```objectivec
main.m:70:22: warning: format specifies type 'char *' but the argument has type 'int' [-Wformat]
    myPrint("i=%s\n",6);
               ~~    ^
               %d
main.m:72:26: warning: format specifies type 'char *' but the argument has type 'int' [-Wformat]
    myPrint("%s,%d,%d\n",1,2);
             ~~          ^
             %d
main.m:72:21: warning: more '%' conversions than data arguments [-Wformat]
    myPrint("%s,%d,%d\n",1,2);
                   ~^
3 warnings generated.
```
如果将
\_\_attribute\_\_((format(printf,1,2)))直接去掉，再次编译，则不会有任何警告。

#####  \_\_attribute\_\_((noreturn))

该属性通知编译器函数从不返回值。当遇到类似函数还未运行到return语句就需要退出来的情况，该属性可以避免出现错误信息。C库函数中的abort（）和exit（）的声明格式就采用了这种格式。使用如下:

```objectivec
void fatal () __attribute__ ((noreturn));
          
void fatal (/* ... */) {
   /* ... */ 
   /* Print error message. */ 
   /* ... */
   exit (1);
}
```

##### \_\_attribute\_\_((constructor/destructor))

若函数被设定为constructor属性，则该函数会在main（）函数执行之前被自动的执行。类似的，若函数被设定为destructor属性，则该 函数会在main（）函数执行之后或者exit（）被调用后被自动的执行。拥有此类属性的函数经常隐式的用在程序的初始化数据方面。
例如：

```objectivec
static __attribute__((constructor)) void before() {
    
    printf("Hello");
}

static __attribute__((destructor)) void after() {
    printf(" World!\n");
}
int main(int argc, const char * argv[]) {
    printf("0000");
    return 0;
}
```
程序输出结果是：

```objectivec
Hello0000 World!
```
如果多个函数使用了这个属性，可以为它们设置优先级来决定执行的顺序：

```objectivec
__attribute__((constructor(PRIORITY)))
__attribute__((destructor(PRIORITY)))
```
如：

```objectivec
static __attribute__((constructor(101))) void before1() {
	printf("before1\n");
}

static __attribute__((constructor(102))) void before2() {
	printf("before2\n");
}

static __attribute__((destructor(201))) void after1() {
    printf("after1\n");
}

static __attribute__((destructor(202))) void after2() {
    printf("after2\n");
}
int main(int argc, const char * argv[]) {
    printf("0000\n");
    return 0;
}
```
输出的结果是：

```objectivec
before1
before2
0000
after2
after1
```
从输出的信息看，前处理都是按照优先级先后执行的，而后处理则是相反的.
需要注意的是：优先级是有范围的。0-100(包括100),是内部保留的，所以在编码的时候需要注意.
另外一个注意的点是，上面的函数没有声明而是直接实现这样\_\_attribute\_\_就需要放到前面,应该多使用函数声明和实现分离的方法：

```objectivec
static void before1() __attribute__((constructor(1)));

static void before1() {
	printf("before1\n");
}
```
其他的函数属性（Function Attribute ）还有：noinline, always_inline, pure, const, nothrow, sentinel, format, format_arg, no_instrument_function, section, constructor, destructor, used, unused, deprecated, weak, malloc, alias, warn_unused_result, nonnull等等，可以参考 <a href="https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Function-Attributes.html" target='blank'>GNU C Function Attribute</a>查看它们的具体使用方法。

如何同时使用多个属性：
可以在同一个函数声明里使用多个__attribute__，并且实际应用中这种情况是十分常见的。只需要把它们用','隔开就可以，如：

```objectivec
 __attribute__((noreturn, format(printf, 1, 2)))
```

#### 2.类型属性 (Type Attributes)

##### \_\_attribute\_\_ aligned
该属性设定一个指定大小的对齐格式（以字节 为单位），例如：

```objectivec
struct S {
    short f[3];
} __attribute__ ((aligned (8)));

typedef int more_aligned_int __attribute__ ((aligned (8)));
```
该声明将强制编译器确保（尽它所能）变量类型为struct S 或者more_aligned_int的变量在分配空间时采用至少8字节对齐方式。
如上所述，你可以手动指定对齐的格式，同样，你也可以使用默认的对齐方式。如果aligned 后面不紧跟一个指定的数字值，那么编译器将依据你的目标机器情况使用最大最有益的对齐方式。例如：

```objectivec
struct S {
    short f[3];
} __attribute__ ((aligned));
```
这里，如果sizeof(short)的大小为2(byte)，那么，S的大小就为6。取一个2的次方值，使得该值大于等于6，则该值为8，所以编译器将设置S类型的对齐方式为8字节。
aligned 属性使被设置的对象占用更多的空间，相反的，使用packed 可以减小对象占用的空间。
需要注意的是，attribute属性的效力与你的连接器也有关，如果你的连接器最大只支持16字节对齐，那么你此时定义32 字节对齐也是无济于事的。

##### \_\_attribute\_\_ packed
使用该属性对struct或者union类型进行定义，设定其类型的每一个变量的内存约束。当用在enum类型定义时，暗示了应该使用最小完整的类型（it indicates that the smallest integral type should be used）。

  下面的例子中，packed_struct 类型的变量数组中的值将会紧紧的靠在一起，但内部的成员变量s不会被“pack” ，如果希望内部的成员变量也被packed 的话，unpacked-struct也需要使用packed进行相应的约束。

```objectivec
struct my_unpacked_struct {
    char c;
    int i;
};

struct my_packed_struct {
    char c;
    int  i;
    struct my_unpacked_struct s;
}__attribute__ ((__packed__));
```

下面的例子中使用这个属性定义了一些结构体及其变量，并给出了输出结果和对结果的分析。
程序代码为：

```objectivec
struct p {
    int a;
    char b;
    short c;
}__attribute__((aligned(4))) pp;

struct m {
    char a;
    int b;
    short c;
}__attribute__((aligned(4))) mm;

struct o {
    int a;
    char b;
    short c;
}oo;

struct x {
    int a;
    char b;
    struct p px;
    short c;
}__attribute__((aligned(8))) xx;
int main() {
    printf("sizeof(int)=%d,sizeof(short)=%d.sizeof(char)=%d\n",sizeof(int),sizeof(short),sizeof(char));
    printf("pp=%d,mm=%d \n", sizeof(pp),sizeof(mm));
    printf("oo=%d,xx=%d \n", sizeof(oo),sizeof(xx));
    return 0;
}
```
输出结果为：

```objectivec
sizeof(int)=4,sizeof(short)=2.sizeof(char)=1
pp=8,mm=12 
oo=8,xx=24 
```

分析：
1.sizeof(pp):
sizeof(a)+sizeof(b)+sizeof(c)=4+1+1=6<8 所以sizeof(pp)=8

2.sizeof(mm):
sizeof(a)+sizeof(b)+sizeof(c)=1+4+2=7
但是a后面需要用3个字节填充，但是b是4个字节，所以a占用4字节，b占用4 个字节，而c又要占用4个字节。所以sizeof(mm)=12

3.sizeof(oo):
sizeof(a)+sizeof(b)+sizeof(c)=4+1+2=7
因为默认是以4字节对齐，所以sizeof(oo)=8

4.sizeof(xx):
sizeof(a)+ sizeof(b)=4+1=5
sizeof(pp)=8; 即xx是采用8字节对齐的，所以要在a，b后面添3个空余字节，然后才能存储px，
4+1+(3)+8+1=17
因为xx采用的对齐是8字节对齐，所以xx的大小必定是8的整数倍，即xx的大小是一个比17大又是8的倍数的一个最小值，17<24，所以sizeof(xx)=24.
  
其他的函数属性（Function Attribute ）还有：aligned, packed, transparent_union, unused, deprecated and may_alias等等，可以参考<a href="https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Type-Attributes.html" target='blank'>GNU C Function Attribute</a>查看它们的具体使用方法。

#### 3.变量属性 (Variable Attribute)

变量属性最常用的是aligned和packed(和上面介绍的用法相同)，其他变量属性的用法请参考<a href="https://gcc.gnu.org/onlinedocs/gcc-4.0.0/gcc/Variable-Attributes.html" target='blank'>GNU C Variable Attribute</a>。

#### 4.Clang特有的\_\_attribute\_\_
如同GCC编译器, Clang也支持 \_\_attribute\_\_, 而且对它进行的一些扩展.
如果要检查指定属性的可用性，你可以使用__has_attribute指令。如：

```objectivec
#if defined(__has_feature)
  #if __has_attribute(availability)
    // 如果`availability`属性可以使用，则....
  #endif
#endif
```
下面介绍一些clang特有的attribute

##### availability
该属性描述了方法关于操作系统版本的适用性， 使用如下:

```objectivec
(void) __attribute__((availability(macosx,introduced=10.4,deprecated=10.6,obsoleted=10.7)));
```
他表示被修饰的方法在首次出现在 OS X Tiger(10.4),在OS X Snow Leopard(10.6)中废弃，在 OS X Lion(10.7)移除。
clang可以利用这些信息决定什么时候使用它使安全的。比如：clang在OS X Leopard（）编译代码，调用这个方法是成功的，如果在OS X Snow Leopard编译代码，调用成功但是会发出警告指明这个方法已经废弃，如果是在OS X Lion，调用会失败，因为它不再可用。
availability属性是一个以逗号为分隔的参数列表，以平台的名称开始，包含一些放在附加信息里的一些里程碑式的声明。
introduced：第一次出现的版本。
deprecated：声明要废弃的版本，意味着用户要迁移为其他API
obsoleted：声明移除的版本，意味着完全移除，再也不能使用它
unavailable：在这些平台不可用
message：一些关于废弃和移除的额外信息，clang发出警告的时候会提供这些信息，对用户使用替代的API非常有用。
这个属性支持的平台：
ios，macosx。

##### overloadable
clang 在C语言中实现了对C++函数重载的支持，使用overloadable属性可以实现。例如：

```objectivec
#include <math.h>
float __attribute__((overloadable)) tgsin(float x) { return sinf(x); }
double __attribute__((overloadable)) tgsin(double x) { return sin(x); }
long double __attribute__((overloadable)) tgsin(long double x) { return sinl(x); }
```
要注意的是overloadable仅仅对函数有效， 你可以重载方法，不过范围局限于返回值和参数是常规类型的如：id和void *。