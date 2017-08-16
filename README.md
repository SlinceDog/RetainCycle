## 理论简介

内存管理可以追溯到手动内存管理（Manual Retain Release，简称 MRR）。在 MRR，开发者创建的每一个对象，需要声明其拥有权，从而保持对象存在于内存中，当对象不再需要的时候撤销拥有权释放它。MRR 通过引用计数系统实现这套拥有权体系，也就是说每个对象有个计数器，通过计数加1表明被一个对象拥有，减1表明不再持有。当计数为零，对象将被释放。由于手动管理内存实在太烦人，因此苹果推出了自动引用计数（ARC）来解放开发者，不再需要开发者手动添加 retain 和 release 操作，从而可以专注于 App 开发。在 ARC，开发者将会定义一个变量为“strong”或“weak”。一个 weak 弱引用无法 retain 对象，而 strong 引用会 retain 这个对象，并将其引用计数加一。

- 说 iOS 的内存管理，就不得不从 ARC（Automatic Reference Counting / 自动引用计数） 说起， ARC 是 WWDC2011 和 iOS5 引入的变化。ARC 是 LLVM 3.0 编译器的特性，用来自动管理内存。

- 与 Java 中 GC 不同，ARC 是编译器特性，而不是基于运行时的，所以 ARC 其实是在编译阶段自动帮开发者插入了管理内存的代码，而不是实时监控与回收内存。

![forumImage20160511110244548](http://jbcdn2.b0.upaiyun.com/2017/02/734c3dfc55eef2963504874e7023c3a9.png)

ARC 的内存管理规则可以简述为：
--  每个对象都有一个『被引用计数』
-- 对象被持有，『被引用计数』+1
-- 对象被放弃持有，『被引用计数』-1
--『引用计数』=0，释放对象

##需要知道
-- 包含 NSObject 类的 Foundation 框架并没有公开
-- Core Foundation 框架源代码，以及通过 NSObject 进行内存管理的部分源代码     是公开的。
-- GNUstep 是 Foundation 框架的互换框架
-- GNUstep 也是 GNU 计划之一。将 Cocoa Objective-C 软件库以自由软件方式重新实现
-- 某种意义上，GNUstep 和 Foundation 框架的实现是相似的
-- 通过 GNUstep 的源码来分析 Foundation 的内存管理

##alloc retain release dealloc 的实现
###GNU – alloc
查看 GNUStep 中的 alloc 函数。
GNUstep/modules/core/base/Source/NSObject.m alloc:
```
+ (id) alloc
{
return [self allocWithZone: NSDefaultMallocZone()];
}

+ (id) allocWithZone: (NSZone*)z
{
return NSAllocateObject (self, 0, z);
}
```
GNUstep/modules/core/base/Source/NSObject.m NSAllocateObject:
```
struct obj_layout {
NSUInteger retained;
};

NSAllocateObject(Class aClass, NSUInteger extraBytes, NSZone *zone)
{
int size = 计算容纳对象所需内存大小;
id new = NSZoneCalloc(zone, 1, size);
memset (new, 0, size);
new = (id)&((obj)new)[1];
}
```
NSAllocateObject 函数通过调用 NSZoneCalloc 函数来分配存放对象所需的空间，之后将该内存空间置为 nil，最后返回作为对象而使用的指针。

我们将上面的代码做简化整理：
GNUstep/modules/core/base/Source/NSObject.m alloc 简化版本:
```
struct obj_layout {
NSUInteger retained;
};

+ (id) alloc
{
int size = sizeof(struct obj_layout) + 对象大小;
struct obj_layout *p = (struct obj_layout *)calloc(1, size);
return (id)(p+1)
return [self allocWithZone: NSDefaultMallocZone()];
}
```
alloc 类方法用 struct obj_layout 中的 retained 整数来保存引用计数，并将其写入对象的内存头部，该对象内存块全部置为 0 后返回。

一个对象的表示便如下图：

![forumImage20160511110244548](http://jbcdn2.b0.upaiyun.com/2017/02/04a4a08fba27276a2cd486e5e045df9b.png)

##GNU – retain
GNUstep/modules/core/base/Source/NSObject.m retainCount:
```
- (NSUInteger) retainCount
{
return NSExtraRefCount(self) + 1;
}

inline NSUInteger
NSExtraRefCount(id anObject)
{
return ((obj_layout)anObject)[-1].retained;
}
```
GNUstep/modules/core/base/Source/NSObject.m retain:
```
- (id) retain
{
NSIncrementExtraRefCount(self);
return self;
}

inline void
NSIncrementExtraRefCount(id anObject)
{
if (((obj)anObject)[-1].retained == UINT_MAX - 1)
[NSException raise: NSInternalInconsistencyException
format: @"NSIncrementExtraRefCount() asked to increment too far”];
((obj_layout)anObject)[-1].retained++;
}
```
以上代码中， NSIncrementExtraRefCount 方法首先写入了当 retained 变量超出最大值时发生异常的代码（因为 retained 是 NSUInteger 变量），然后进行 retain ++ 代码。

##GNU – dealloc
dealloc 将会对对象进行释放。
GNUstep/modules/core/base/Source/NSObject.m dealloc:

```
- (void) dealloc
{
NSDeallocateObject (self);
}

inline void
NSDeallocateObject(id anObject)
{
obj_layout o = &((obj_layout)anObject)[-1];
free(o);
}
```
###__unsafe_unretained

有时候我们除了 __weak 和 __strong 之外也会用到 __unsafe_unretained 这个修饰符，那么我们对 __unsafe_unretained 了解多少？

__unsafe_unretained 是不安全的所有权修饰符，尽管 ARC 的内存管理是编译器的工作，但附有 __unsafe_unretained 修饰符的变量不属于编译器的内存管理对象。赋值时即不获得强引用也不获得弱引用。

来运行一段代码：

```
id __unsafe_unretained obj1 = nil;
{
id __strong obj0 = [[NSObject alloc] init];

obj1 = obj0;

NSLog(@"A: %@", obj1);
}

NSLog(@"B: %@", obj1);
```
运行结果：
```
2017-01-12 19:24:47.245220 __unsafe_unretained[55726:4408416] A:
2017-01-12 19:24:47.246670 __unsafe_unretained[55726:4408416] B:
Program ended with exit code: 0
```
对代码进行详细分析：
```
id __unsafe_unretained obj1 = nil;
{
// 自己生成并持有对象
id __strong obj0 = [[NSObject alloc] init];

// 因为 obj0 变量为强引用，
// 所以自己持有对象
obj1 = obj0;

// 虽然 obj0 变量赋值给 obj1
// 但是 obj1 变量既不持有对象的强引用，也不持有对象的弱引用
NSLog(@"A: %@", obj1);
// 输出 obj1 变量所表示的对象
}

NSLog(@"B: %@", obj1);
// 输出 obj1 变量所表示的对象
// obj1 变量表示的对象已经被废弃
// 所以此时获得的是悬垂指针
// 错误访问
```
所以，最后的 NSLog 只是碰巧正常运行，如果错误访问，会造成 crash
在使用 __unsafe_unretained 修饰符时，赋值给附有 __strong 修饰符变量时，要确保对象确实存在

##Block 循环引用
当A对象里面强引用了B对象，B对象又强引用了A对象，这样两者的retainCount值一直都无法为0，于是内存始终无法释放，导致内存泄露。所谓的内存泄露就是本应该释放的对象，在其生命周期结束之后依旧存在。
