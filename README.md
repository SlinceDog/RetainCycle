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
![forumImage20160511110244548](http://jbcdn2.b0.upaiyun.com/2016/09/2d489b41fcc29f40aed1e4a518faa995.jpeg)
这是2个对象之间的，相应的，这种循环还能存在于3，4……个对象之间，只要相互形成环，就会导致Retain Cicle的问题。

当然也存在自身引用自身的，当一个对象内部的一个obj，强引用的自身，也会导致循环引用的问题出现。常见的就是block里面引用的问题。

### weak、strong的实现原理
在ARC环境下，id类型和对象类型和C语言其他类型不同，类型前必须加上所有权的修饰符。

所有权修饰符总共有4种：

1.strong修饰符 2.weak修饰符 3.unsafe_unretained修饰符 4.autoreleasing修饰符

一般我们如果不写，默认的修饰符是__strong。

要想弄清楚strong，weak的实现原理，我们就需要研究研究clang(LLVM编译器)和objc4 Objective-C runtime库了。

- __strong的实现原理

(1)对象持有自己

首先我们先来看看生成的对象持有自己的情况，利用alloc/new/copy/mutableCopy生成对象。

当我们声明了一个__strong对象
```
{
id __strong obj = [[NSObject alloc] init];
}
```
LLVM编译器会把上述代码转换成下面的样子
```
id __attribute__((objc_ownership(strong))) obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));
```
相应的会调用

```
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj,selector(init));
objc_release(obj);
```
上述这些方法都好理解。在ARC有效的时候就会自动插入release代码，在作用域结束的时候自动释放。

(2)对象不持有自己

生成对象的时候不用alloc/new/copy/mutableCopy等方法。

LLVM编译器会把上述代码转换成下面的样子

```
id __attribute__((objc_ownership(strong))) array = ((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("array"));
```
查看LLVM文档，其实是下述的过程

相应的会调用

```
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleasedReturnValue(obj);
objc_release(obj);
```
与之前对象会持有自己的情况不同，这里多了一个objc_retainAutoreleasedReturnValue函数。

这属于LLVM编译器的一个优化。objc_retainAutoreleasedReturnValue函数是用于自己持有(retain)对象的函数，它持有的对象应为返回注册在autoreleasepool中对象的方法或者是函数的返回值。

在ARC中原本对象生成之后是要注册到autoreleasepool中，但是调用了objc_autoreleasedReturnValue 之后，紧接着调用了 objc_retainAutoreleasedReturnValue，objc_autoreleasedReturnValue函数会去检查该函数方法或者函数调用方的执行命令列表，如果里面有objc_retainAutoreleasedReturnValue()方法，那么该对象就直接返回给方法或者函数的调用方。达到了即使对象不注册到autoreleasepool中，也可以返回拿到相应的对象。

### __weak的实现原理
声明一个__weak对象

```
{
id __weak obj = strongObj;
}
```
假设这里的strongObj是一个已经声明好了的对象。

LLVM转换成对应的代码

```
id __attribute__((objc_ownership(none))) obj1 = strongObj;
```
相应的会调用

```
id obj ;
objc_initWeak(&obj,strongObj);
objc_destoryWeak(&obj);
```
objc_initWeak的实现其实是这样的
```
id objc_initWeak(id *object, id value) {   
*object = nil; 
return objc_storeWeak(object, value);
}
```
也是会去调用objc_storeWeak函数。objc_initWeak和objc_destroyWeak函数都会去调用objc_storeWeak函数，唯一不同的是调用的入参不同，一个是value，一个是nil。

objc_storeWeak函数的用途就很明显了。由于weak表也是用Hash table实现的，所以objc_storeWeak函数就把第一个入参的变量地址注册到weak表中，然后根据第二个入参来决定是否移除。如果第二个参数为0，那么就把__weak变量从weak表中删除记录，并从引用计数表中删除对应的键值记录。

所以如果weak引用的原对象如果被释放了，那么对应的weak对象就会被指为nil。原来就是通过objc_storeWeak函数这些函数来实现的。

##weakSelf、strongSelf的用途

在提weakSelf、strongSelf之前，我们先引入一个Retain Cicle的例子。

假设自定义的一个student类

```
#import 
typedef void(^Study)();
@interface Student : NSObject
@property (copy , nonatomic) NSString *name;
@property (copy , nonatomic) Study study;
@end
```

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
[super viewDidLoad];

Student *student = [[Student alloc]init];
student.name = @"Hello World";

student.study = ^{
NSLog(@"my name is = %@",student.name);
};
}
```
到这里，大家应该看出来了，这里肯定出现了循环引用了。student的study的Block里面强引用了student自身。会在内部持有它。retainCount值会加一。

我们用Instruments来观察一下。添加Leak观察器。

当程序运行起来之后，在Leak Checks观察器里面应该可以看到红色的❌，点击它就会看到内存leak了。有2个泄露的对象。Block和Student相互循环引用了。

![forumImage20160511110244548](http://jbcdn2.b0.upaiyun.com/2016/09/17be0b3a71ea00caee64b07cf4e9028f.jpeg)

打开Cycles & Roots 观察一下循环的环。

![forumImage20160511110244548](http://jbcdn2.b0.upaiyun.com/2016/09/855bd8fd7078953a2f0e37dc43a618c8.jpeg)

这里形成环的原因block里面持有student本身，student本身又持有block。

那再看一个例子2：

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
[super viewDidLoad];

Student *student = [[Student alloc]init];
student.name = @"Hello World";

student.study = ^(NSString * name){
NSLog(@"my name is = %@",name);
};
student.study(student.name);
}
```
我把block新传入一个参数，传入的是student.name。这个时候会引起循环引用么？

答案肯定是不会。原因是因为，student是作为形参传递进block的，block并不会捕获形参到block内部进行持有。所以肯定不会造成循环引用。

再改一下。看例子3：

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@property (copy,nonatomic) NSString *name;
@property (strong, nonatomic) Student *stu;
@end

@implementation ViewController

- (void)viewDidLoad {
[super viewDidLoad];

Student *student = [[Student alloc]init];

self.name = @"halfrost";
self.stu = student;

student.study = ^{
NSLog(@"my name is = %@",self.name);
};

student.study();
}
```
这样会形成循环引用么？答案也是否定的。

ViewController虽然强引用着student，但是student里面的blcok强引用的是viewController的name属性，并没有形成环。如果把上述的self.name改成self，也依旧不会产生循环引用。因为他们都没有强引用这个block。

那遇到循环引用我们改如何处理呢？？类比平时我们经常写的delegate，可以知道，只要有一边是__weak就可以打破循环。

常见的一种做法，利用__block解决循环的做法。例子4：
```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
[super viewDidLoad];

Student *student = [[Student alloc]init];

__block Student *stu = student;
student.name = @"Hello World";
student.study = ^{
NSLog(@"my name is = %@",stu.name);
stu = nil;
};
}
```
这样写会循环么？看上去应该不会。但是实际上却是会的。
![forumImage20160511110244548](http://jbcdn2.b0.upaiyun.com/2016/09/86e2ba0e148abd44999b3eaf925650f2.jpeg)
![forumImage20160511110244548](http://jbcdn2.b0.upaiyun.com/2016/09/5f1d74ffb984aa0d79e403c1b6856eac.jpeg)
由于没有执行study这个block，现在student持有该block，block持有block变量，block变量又持有student对象。3者形成了环，导致了循环引用了。 想打破环就需要破坏掉其中一个引用。__block不持有student即可。

只需要执行一下block即可。例子5：
```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
[super viewDidLoad];

Student *student = [[Student alloc]init];
student.name = @"Hello World";
__block Student *stu = student;

student.study = ^{
NSLog(@"my name is = %@",stu.name);
stu = nil;
};

student.study();
}
```
这样就不会循环引用了。

使用block解决循环引用虽然可以控制对象持有时间，在block中还能动态的控制是block变量的值，可以赋值nil，也可以赋值其他的值，但是有一个唯一的缺点就是需要执行一次block才行。否则还是会造成循环引用。

值得注意的是，在ARC下__block会导致对象被retain，有可能导致循环引用。而在MRC下，则不会retain这个对象，也不会导致循环引用。

接下来说说weakSelf 和 strongSelf的用法了。

1.weakSelf

说道weakSelf，需要先来区分几种写法。 weak typeof(self)weakSelf = self; 这是AFN里面的写法。。

define WEAKSELF typeof(self) __weak weakSelf = self; 这是我们平时的写法。。

先区分typeof() 和 typeof() ，其实两者都是一样的东西，只不过是C里面不同的标准，兼容性不同罢了。

抽象出来就是这2种写法。

define WEAKSELF __weak typeof(self)weakSelf = self;

define WEAKSELF typeof(self) __weak weakSelf = self;

这样子看就清楚了，两种写法就是完全一样的。

我们可以用WEAKSELF来解决循环引用的问题

例子6：

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
[super viewDidLoad];

Student *student = [[Student alloc]init];
student.name = @"Hello World";
__weak typeof(student) weakSelf = student;

student.study = ^{
NSLog(@"my name is = %@",weakSelf.name);
};

student.study();
}
```
这样就解决了循环引用的问题了。

解决循环应用的问题一定要分析清楚哪里出现了循环引用，只需要把其中一环加上weakSelf这类似的宏，就可以解决循环引用。如果分析不清楚，就只能无脑添加weakSelf、strongSelf，这样的做法不可取。

在上面的例子3中，就完全不存在循环引用，要是无脑加weakSelf、strongSelf是不对的。在例子6中，也只需要加一个weakSelf就可以了，也不需要加strongSelf。

2.strongSelf

上面介绍完了weakSelf，既然weakSelf能完美解决Retain Circle的问题了，那为何还需要strongSelf呢？

如果block里面不加strong typeof(weakSelf)strongSelf = weakSelf会如何呢？
```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
[super viewDidLoad];

Student *student = [[Student alloc]init];
student.name = @"Hello World";
__weak typeof(student) weakSelf = student;

student.study = ^{
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
NSLog(@"my name is = %@",weakSelf.name);
});
};

student.study();
}
```
```
my name is = (null)
```
为什么输出是这样的呢？

重点就在dispatch_after这个函数里面。在study()的block结束之后，student被自动释放了。又由于dispatch_after里面捕获的weak的student，在原对象释放之后，__weak对象就会变成null，防止野指针。所以就输出了null了。

那么我们怎么才能在weakSelf之后，block里面还能继续使用weakSelf之后的对象呢？

究其根本原因就是weakSelf之后，无法控制什么时候会被释放，为了保证在block内不会被释放，需要添加__strong。

在block里面使用的__strong修饰的weakSelf是为了在函数生命周期中防止self提前释放。strongSelf是一个自动变量当block执行完毕就会释放自动变量strongSelf不会对self进行一直进行强引用。

```
#import "ViewController.h"
#import "Student.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
[super viewDidLoad];

Student *student = [[Student alloc]init];

student.name = @"Hello World";
__weak typeof(student) weakSelf = student;

student.study = ^{
__strong typeof(student) strongSelf = weakSelf;
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
NSLog(@"my name is = %@",strongSelf.name);
});

};

student.study();
}
```
输出

```
my name is = Hello World
```
至此，我们就明白了weakSelf、strongSelf的用途了。

weakSelf 是为了block不持有self，避免Retain Circle循环引用。在 Block 内如果需要访问 self 的方法、变量，建议使用 weakSelf。

strongSelf的目的是因为一旦进入block执行，假设不允许self在这个执行过程中释放，就需要加入strongSelf。block执行完后这个strongSelf 会自动释放，没有不会存在循环引用问题。如果在 Block 内需要多次 访问 self，则需要使用 strongSelf。

关于Retain Circle最后总结一下，有3种方式可以解决循环引用。

总结一下

EOCNetworkFetcher.h

```
typedef void (^EOCNetworkFetcherCompletionHandler)(NSData *data);

@interface EOCNetworkFetcher : NSObject

@property (nonatomic, strong, readonly) NSURL *url;

- (id)initWithURL:(NSURL *)url;

- (void)startWithCompletionHandler:(EOCNetworkFetcherCompletionHandler)completion;

@end
```
EOCNetworkFetcher.m

```
@interface EOCNetworkFetcher ()

@property (nonatomic, strong, readwrite) NSURL *url;
@property (nonatomic, copy) EOCNetworkFetcherCompletionHandler completionHandler;
@property (nonatomic, strong) NSData *downloadData;

@end

@implementation EOCNetworkFetcher

- (id)initWithURL:(NSURL *)url {
if(self = [super init]) {
_url = url;
}
return self;
}

- (void)startWithCompletionHandler:(EOCNetworkFetcherCompletionHandler)completion {
self.completionHandler = completion;
//开始网络请求
dispatch_async(dispatch_get_global_queue(0, 0), ^{
_downloadData = [[NSData alloc] initWithContentsOfURL:_url];
dispatch_async(dispatch_get_main_queue(), ^{
//网络请求完成
[self p_requestCompleted];
});
});
}

- (void)p_requestCompleted {
if(_completionHandler) {
_completionHandler(_downloadData);
}
}

@end
```
EOCClass.m

```
@implementation EOCClass {
EOCNetworkFetcher *_networkFetcher;
NSData *_fetchedData;
}

- (void)downloadData {
NSURL *url = [NSURL URLWithString:@"http://www.baidu.com"];
_networkFetcher = [[EOCNetworkFetcher alloc] initWithURL:url];
[_networkFetcher startWithCompletionHandler:^(NSData *data) {
_fetchedData = data;
}];
}
@end
```
在这个例子中，存在3者之间形成环

1、completion handler的block因为要设置_fetchedData实例变量的值，所以它必须捕获self变量，也就是说handler块保留了EOCClass实例；

2、EOCClass实例通过strong实例变量保留了EOCNetworkFetcher，最后EOCNetworkFetcher实例对象也会保留了handler的block。

方法一：手动释放EOCNetworkFetcher使用之后持有的_networkFetcher，这样可以打破循环引用

```
- (void)downloadData {
NSURL *url = [NSURL URLWithString:@"http://www.baidu.com"];
_networkFetcher = [[EOCNetworkFetcher alloc] initWithURL:url];
[_networkFetcher startWithCompletionHandler:^(NSData *data) {
_fetchedData = data;
_networkFetcher = nil;//加上此行，打破循环引用
}];
}
```
方法二：直接释放block。因为在使用完对象之后需要人为手动释放，如果忘记释放就会造成循环引用了。如果使用完completion handler之后直接释放block即可。打破循环引用

```
- (void)p_requestCompleted {
if(_completionHandler) {
_completionHandler(_downloadData);
}
self.completionHandler = nil;//加上此行，打破循环引用
}
```
方法三：使用weakSelf、strongSelf
```
- (void)downloadData {
__weak __typeof(self) weakSelf = self;
NSURL *url = [NSURL URLWithString:@"http://www.baidu.com"];
_networkFetcher = [[EOCNetworkFetcher alloc] initWithURL:url];
[_networkFetcher startWithCompletionHandler:^(NSData *data) {
__typeof(&*weakSelf) strongSelf = weakSelf;
if (strongSelf) {
strongSelf.fetchedData = data;
}
}];
}
```
