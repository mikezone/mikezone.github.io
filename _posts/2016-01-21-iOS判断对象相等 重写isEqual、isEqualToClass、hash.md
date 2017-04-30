---
layout: cnblog_post
title:  "iOS判断对象相等 重写isEqual、isEqualToClass、hash"
date:   2016-01-21 13:50:39
categories: iOS
---
相等的概念是探究哲学和数学的核心，并且对道德、公正和公共政策的问题有着深远的影响。
从一个经验主义者的角度来看，两个物体不能依据一些观测标准中分辨出来，它们就是相等的。在人文方面，平等主义者认为相等意味着要保持每个人的社会、经济、政治和他们住地的司法系统都一致。
对程序员来说，协调好逻辑和感官能力来理解我们塑造的'相同'的语义是一项任务。'相同的问题'(的探讨)太微妙，同时有太容易被忽视。对语义没有充分的理解就直接去实现它，可能会导致没必要的工作和不正确的结果。因此对数学和逻辑系统的深刻理解与按既定计划实现同样必要。

虽然所有的技术博文都是有诱惑你来读它的标题和代码，但请花几分钟时间来阅读和理解这些文字。逐字地复制看似有用的代码而不知道为什么这样写很有可能导致一些错误。相等性是个重要话题之一，但它仍包含了许多混乱的概念，尤其是在Objective-C中。

### Equality & Identity
首先，弄清楚equality和identity的区别很重要。
如果两个物体具有相同的观测属性，它们是可以相互等同的。但是，这两个对象仍然可以分辨出差异，它们各自的identity。在程序中，一个对象的identity是和它的内存地址关联的。

NSObject对象测试和另一个对象是否相同使用`isEqual:`方法，在它的基本实现里性等性检查本质上是对identity的检查，如果两个对象指向了相同的内存地址，它们被认为是相等的。

```objectivec
@implementation NSObject (Approximate)
- (BOOL)isEqual:(id)object {
  return self == object;
}
@end
```

对于内置的类，像NSArray, NSDictionary, 和 NSString，进行了一个深层的相等性比较，来测试在集合中的每个元素是否相等，这是一个应该也确实非常有用的做法。
NSObject 的子类要实现它们各自的`isEqual:`方法时，应该做到以下几点：

>1.实现一个`isEqualTo__ClassName__:`方法来执行有意义的值比较.<br/>
>2.重写`isEqual: `方法 来作类型和对象identity检查, 回调上述的值比较方法.<br/>
>3.重写 hash, 这个会在下一部分解释.

这里有一个NSArray实现这个的大概的思路(这个例子忽略了类簇, 实际实现会更具体复杂):

```objectivec
@implementation NSArray (Approximate)
- (BOOL)isEqualToArray:(NSArray *)array {
  if (!array || [self count] != [array count]) {
    return NO;
  }

  for (NSUInteger idx = 0; idx < [array count]; idx++) {
      if (![self[idx] isEqual:array[idx]]) {
          return NO;
      }
  }

  return YES;
}

- (BOOL)isEqual:(id)object {
  if (self == object) {
    return YES;
  }

  if (![object isKindOfClass:[NSArray class]]) {
    return NO;
  }

  return [self isEqualToArray:(NSArray *)object];
}
@end
```
下面的在Foudation中NSObject的子类已经自定义了判等实现，用了相关的方法：

```objectivec
NSAttributedString -isEqualToAttributedString:
NSData -isEqualToData:
NSDate -isEqualToDate:
NSDictionary -isEqualToDictionary:
NSHashTable -isEqualToHashTable:
NSIndexSet -isEqualToIndexSet:
NSNumber -isEqualToNumber:
NSOrderedSet -isEqualToOrderedSet:
NSSet -isEqualToSet:
NSString -isEqualToString:
NSTimeZone -isEqualToTimeZone:
NSValue -isEqualToValue:
```

当比较任何这些类的两个实例时，推荐使用它们各自的高级别的method而不是`isEqual:`
然而，我们的理论实现还没有完成，现在，让我们把注意力转向hash（一段插曲：先清理一下NSString的问题）。

##### NSString判等的奇怪案例
一个有趣的插曲，看一下这个代码：

```objectivec
NSString *a = @"Hello";
NSString *b = @"Hello";
BOOL wtf = (a == b); // YES
```
郑重地声明一下正确的比较两个NSString对象相等的方法是使用`-isEqualToString:`方法，无论如何也不能通过`==`操作符来比较两个NSString。
那么这里是怎么回事呢？为什么 NSArray或者NSDictionary字面量相同不会这样，而它(NSString)会这样呢。

这都是一种被称为`字符串驻留`的优化技术做的，因为这种优化不同的值可以对一份不可变的字符串值的备份进行拷贝。NSString类型的a指针和b指针对驻留字符串 @"Hello"进行了相同的拷贝。注意这个优化仅仅对静态声明的不可变字符串有效。

更有趣的是，OC的selector的名字也会被当做驻留字符串存储在一个共用的字符串pool中。

### Hashing

最日常的面向对象编程来说，对象判等最主要的用法在于决定集合成员。为了让这一步更快一些，自定义判等实现的类应该也实现hash:
>1.对象相等是相互的([a isEqual:b] ⇒ [b isEqual:a])
>2.如果对象相等，它们的hash值必须相等([a isEqual:b] ⇒ [a hash] == [b hash])
>但是，反过来不一定成立：如果它们的hash值相等，两个对象不一定相等。([a hash] == [b hash] ¬⇒ [a isEqual:b])

现在快速翻看一下《计算机科学》101:

hash表式编程中的基本的数据结构，它可以使NSSet & NSDictionary 快速地(O(1))查找它的元素。
我们也可以通过对比着数组很好地理解hash表：

Arrays按照有序的索引存储元素，因此一个大小为n的数组会把元素放在索引1，2直到n-1.为了确定数组中的一个元素存在了哪里，不得不一个个检查每个位置(除非数组碰巧已经排序好，但这是另一回事)。

Hash表使用了略微不同的方法。而不是按顺序存储元素，hash表在内存中分配了n个位置，同时用一个函数来计算在这个范围内计算一个位置。一个hash函数是确定性的，同时一个好的hash函数使用一个相对均匀的散列来生成值，而且不会有太多的计算过程。当两个不同的对象计算出相同的hash值时，会产生hash冲突。当冲突发生时，hash表会寻找冲突点同时把新加的对象放到第一个可用的位置。当hash表变得越来越拥挤，冲突的可能性会增加，这会导致花费更多的时间来寻找空间(这就是为什么均匀散列的hash函数不菲的原因。)

一个关于实现hash函数的错误共识来自于随之发生的断言，这个错误的共识认为hash值必须是不同的。这个错误共识会导致不必要地复杂实现，包括从Java textbooks复制过来的质数的神奇咒语。实际上，一个简单的对关键属性hash值的XOR(异或运算)对于99%的情况来说已经够用了。

技巧就是思考对象中的哪个值是关键的。
对NSDate来说，对参照日期的时间间隔已经够用了：

```objectivec
@implementation NSDate (Approximate)
- (NSUInteger)hash {
  return (NSUInteger)abs([self timeIntervalSinceReferenceDate]);
}
```
对UIColor来说，移位之后的RGB值是非常方便计算的

```objectivec
@implementation UIColor (Approximate)
- (NSUInteger)hash {
  CGFloat red, green, blue;
  [self getRed:&red green:&green blue:&blue alpha:nil];
  return ((NSUInteger)(red * 255) << 16) + ((NSUInteger)(green * 255) << 8) + (NSUInteger)(blue * 255);
}
@end
```

### 在子类中实现 -isEqual: 和 hash 
综合在一起，这里有一个如何在子类重写默认的判等实现的例子：

```objectivec
@interface Person
@property NSString *name;
@property NSDate *birthday;

- (BOOL)isEqualToPerson:(Person *)person;
@end

@implementation Person

- (BOOL)isEqualToPerson:(Person *)person {
  if (!person) {
    return NO;
  }

  BOOL haveEqualNames = (!self.name && !person.name) || [self.name isEqualToString:person.name];
  BOOL haveEqualBirthdays = (!self.birthday && !person.birthday) || [self.birthday isEqualToDate:person.birthday];

  return haveEqualNames && haveEqualBirthdays;
}

#pragma mark - NSObject

- (BOOL)isEqual:(id)object {
  if (self == object) {
    return YES;
  }

  if (![object isKindOfClass:[Person class]]) {
    return NO;
  }

  return [self isEqualToPerson:(Person *)object];
}

- (NSUInteger)hash {
  return [self.name hash] ^ [self.birthday hash];
}

@end
```

如果想满足好奇心或者出于学究式的研究，看一下这个 <a href="https://mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html" target='blank'>Mike Ash的文章</a> ，解释了通过移位和翻转组合值如何改善了可能产生重叠(冲突)的hash.

本文完全翻译自： <a href="http://nshipster.com/equality/" target='blank'>http://nshipster.com/equality/</a>