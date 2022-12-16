---
title: Runtime 探究
date: 2018-12-19 11:00:51
categories: iOS
tags:
- Runtime
keywords: Runtime,消息转发机制,Runtime的应用,Swizzling
description:
images: "/postCover/runtime探究.png"
---

对于 Runtime 的了解一直很少，面试的时候，总是会提及，因此开始去了解 Runtime。在网上找寻这类资料和文章并开始详读，在详读后对于 Runtime 有了更深了解，但是有些地方仍有些模糊，这样半知半解的状态可不是好事，因此我通过知识整理，模拟向人讲述的方式，以查解自己的模糊之处。若整理的有误之处，望各位大神指出，谢谢！
<!-- more -->
### Runtime 简介
> Runtime 又叫运行时，是一套用 C 和 汇编编语言 写的 API库。Objective-C 扩展了 C 语言，并加入了面向对象特性和 Smalltalk 式的消息传递机制，使得 Objective-C 具有面向对象和动态机制的特性，这两个特性便是用 Runtime 来实现的。

> Runtime其实有两个版本: “modern” 和 “legacy”。我们现在用的 Objective-C 2.0 采用的是现行 (Modern) 版的 Runtime 系统，只能运行在 iOS 和 macOS 10.5 之后的 64 位程序中。而 macOS 较老的32位程序仍采用 Objective-C 1 中的（早期）Legacy 版本的 Runtime 系统。这两个版本最大的区别在于当你更改一个类的实例变量的布局时，在早期版本中你需要重新编译它的子类，而现行版就不需要。

> Runtime 基本是用 C 和汇编写的，可见苹果为了动态系统的高效而作出的努力。你可以在[这里](https://opensource.apple.com/source/objc4/)查看苹果维护的开源代码，另外可以从[此处](https://opensource.apple.com/tarballs/objc4/)下载对应版本压缩包。苹果和GNU各自维护一个开源的 runtime 版本，这两个版本之间都在努力的保持一致。

### NSObject
根据源码可以看到 NSObject 的定义
```Objective-C
typedef struct objc_class *Class;

@interface NSObject <NSObject> {
  Class isa  OBJC_ISA_AVAILABILITY;
}

// objc.h 中 NSObject对应的结构体
struct objc_object {
  Class isa  OBJC_ISA_AVAILABILITY;
};

// objc_object 结构体的实现
struct objc_object {
  private:
  isa_t isa;
  ...
}
```
在 objc_object 中 有一个指向 Class 的 isa 指针，即 objc_object 的类。

#### isa
isa指针 是一个 union 联合体实例，如下： 
```Objective-C
union isa_t {
  isa_t() { }
  isa_t(uintptr_t value) : bits(value) { }

  Class cls;
  uintptr_t bits;
  #if defined(ISA_BITFIELD)
  struct {
    ISA_BITFIELD;  // defined in isa.h
  };
  #endif
};

// ISA_BITFIELD 宏
#   define ISA_BITFIELD                                                        \
uintptr_t nonpointer        : 1;                                         \
uintptr_t has_assoc         : 1;                                         \
uintptr_t has_cxx_dtor      : 1;                                         \
uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ \
uintptr_t magic             : 6;                                         \
uintptr_t weakly_referenced : 1;                                         \
uintptr_t deallocating      : 1;                                         \
uintptr_t has_sidetable_rc  : 1;                                         \
uintptr_t extra_rc          : 8
```
要了解 isa 的实现可看 [从 NSObject 的初始化了解 isa](https://github.com/draveness/analyze/blob/master/contents/objc/从%20NSObject%20的初始化了解%20isa.md#shiftcls)

#### objc_class
Class 是一个 struct objc_class 的结构体实例，objc_class 的定义如下：
```Objective-C
struct objc_class {
  Class isa  OBJC_ISA_AVAILABILITY;

  #if !__OBJC2__
  Class super_class                                        OBJC2_UNAVAILABLE;
  const char *name                                         OBJC2_UNAVAILABLE;
  long version                                             OBJC2_UNAVAILABLE;
  long info                                                OBJC2_UNAVAILABLE;
  long instance_size                                       OBJC2_UNAVAILABLE;
  struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
  struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
  struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
  struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
  #endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */

// objc_class 结构体实现
struct objc_class : objc_object {
  // Class ISA;
  Class superclass;
  cache_t cache;             // formerly cache pointer and vtable
  class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
  ...
}
```
objc_class 中有指向父类的指针、类名、版本、实例的大小、实例变量列表、方法列表、缓存、遵守的协议列表，可见类其实就是一个 objc_class 结构体实例也是一个对象。

#### Meta Class
在 objc_class 中也有一个 isa 指针，这个 isa 指针指向的是 Meta Class（元类），该类保存了创建类对象以及类方法所需的所有信息。元类也是一个对象，可以向元类发送消息，此类消息便是类方法（带 “+” 的方法）。每个类都会有一个单独对应的元类，因为每个类只有一个methodLists，方法不能重名，因此需要有一个类来保存类方法。其中对应关系如下图：
![](https://upload-images.jianshu.io/upload_images/1194012-d7b097e86f9e488d.png?imageMogr2/auto-orient/strip|imageView2/2/w/625/format/webp)
- 每个对象(instance)的 isa 指针指向对象类(class)，而对象类指向唯一的元类(meta class)。
- 每个类都有指向父类的指针，class -> superClass、class的meta class -> superClass 的 meta class。
- 每个meta class 都指向 Root meta class
- Root class 便是 NSObject，NSObject 没有超类，所以指向 nil， Root meta class 指向 NSObject，形成闭环。

在main方法执行之前，从 dyld到runtime这期间，类对象和元类对象被创建，具体详细可看 [iOS 程序 main 函数之前发生了什么](http://blog.sunnyxx.com/2014/08/30/objc-pre-main/)

#### method
method 在 Objective-C 中称为方法，在 C 语言称为函数，表示能够独立完成一个功能的一段代码，method 的源码定义如下：
```Objective-C
typedef struct objc_method *Method;

struct objc_method {
  SEL method_name                                          OBJC2_UNAVAILABLE;
  char *method_types                                       OBJC2_UNAVAILABLE;
  IMP method_imp                                           OBJC2_UNAVAILABLE;
}

// objc_method 实现
struct method_t {
  SEL name;
  const char *types;
  IMP imp;

  struct SortBySELAddress :
  public std::binary_function<const method_t&,
  const method_t&, bool>
  {
    bool operator() (const method_t& lhs,
    const method_t& rhs)
    { return lhs.name < rhs.name; }
  };
};
```
method 是一个 objc_method 结构体实例，结构体中有：方法名、方法类型、方法实现的内存地址 三个属性。

##### SEL
SEL 的源码定义：
```Objective-C
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```
SEL 是 selector 在 Objective-C 中的表示类型（Swift中是Selector类）。SEL 是一个 objc_selector 结构体实例，而 objc_selector 是一个映射到方法的C字符串。selector 是一个以方法名为区分的选择器，不同类中相同名字的方法对应的方法选择器是一样的，即使方法名字相同而变量类型不同也会导致它们具有相同的方法选择器，因此这也导致 Objective-C 不支持方法重载。

##### IMP
IMP 的源码定义：
```Objective-C
/// A pointer to the function of a method implementation. 
typedef id (*IMP)(id, SEL, ...); 
```
该指针指向方法实现的内存地址

##### types
Type Encoding类型编码，具体可看 [官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

有兴趣深入了解objc的方法结构可看 [深入解析 ObjC 中方法的结构](https://github.com/draveness/analyze/blob/master/contents/objc/深入解析%20ObjC%20中方法的结构.md#深入解析-objc-中方法的结构)

#### objc_cache
objc_cache 的源码定义：
```Objective-C
struct objc_cache {
  unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
  unsigned int occupied                                    OBJC2_UNAVAILABLE;
  Method buckets[1]                                        OBJC2_UNAVAILABLE;
};

// objc_cache 实现
#if __LP64__
typedef uint32_t mask_t;  // x86_64 & arm64 asm are less efficient with 16-bits
#else
typedef uint16_t mask_t;
#endif

typedef uintptr_t cache_key_t;

struct cache_t {
  // 记录方法名对应的方法实现 IMP
  struct bucket_t *_buckets;
  // 分配用来缓存bucket的总数
  mask_t _mask;
  // 表明目前实际占用的缓存bucket的个数。
  mask_t _occupied;
};

struct bucket_t {
  // 方法名
  cache_key_t _key;
  // 方法实现内存地址
  IMP _imp;
};
```
当方法调用时，通过调用者isa指针获取调用者的类，然后在类的 methodLists 中查找方法实现，如果没找到则往上父类中查找 ，这样的调用方式代价比较大。为了优化效率，会首先去 cache 中缓存的方法中查找，如果没有，就开始遍历类的方法列表，找到后以 方法名为 key，方法实现 IMP 为 value 缓存到 cache中，等下次调用就能加快调用效率。

### 消息传递
#### 消息发送
在Objective-C 中方法的调用如：[receiver message]，在编译后会转化成用于消息发送的C函数函数形式，如下：
```Objective-C
id objc_msgSend(void /* id self, SEL op, ... */ )
```
当objc_msgSend 函数调用后会做以下事情：
1. 检查 selectors 是否要忽略如：GC selectors（macOS中GC垃圾回收机制用到的方法）。
2. 检查调用者是否为nil，如果为nil则忽略此消息不做处理。
3. 去 class 的缓存中查找该方法的实现 IMP，如果找到便将方法名和IMP缓存到 cache 中，调用该方法。如果没找到便去父类的方法列表中查找，一直找到根类 NSObject 都没找到的话就开始进入消息转发阶段。

有的时候，我们会被表面调用者迷惑，如：
```Objective-C
@implementation Son : Father
NSLog(@"%@", NSStringFromClass([self class]));
NSLog(@"%@", NSStringFromClass([super class]));
```
看表面，我们会以为结果是：Son、Father，但正确结果是 Son、Son。self 是类的一个隐藏参数，每个方法的实现的第一个参数即为self，代表对象自身，
而 super 并不是隐藏参数，它实际上只是一个”编译器标示符”，它负责告诉编译器，当调用方法时，以 self 去调用父类的方法。下面是 super 调用的实现
```Objective-C
void objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )

// Specifies the superclass of an instance. 
struct objc_super {
  // Specifies an instance of a class.
  __unsafe_unretained id receiver;

  // Specifies the particular superclass of the instance to message. 
  #if !defined(__cplusplus)  &&  !__OBJC2__
  /* For compatibility with old objc-runtime.h header */
  __unsafe_unretained Class class;
  #else
  __unsafe_unretained Class super_class;
  #endif
  /* super_class is the first class to search */
};
#endif
```
错觉在于我们以为最终是 super_class->class 方法，然而在 objc_super 结构体中 super_class 并非是消息接收者，receiver 才是接收者，根据注释我们可以知道 receiver 其实就是 self，所以最终消息发送是这样的
```Objective-C
objc_msgSend(objc_super->receiver, @selector(class))
```

#### 消息转发
在继承树上找不到方法实现，这个时候就会将消息转发给予接收者最后三次机会去处理，如果还是未处理便会执行 doesNotRecognizeSelector: 方法方法终止程序，报 unrecognized selector 错误。

消息转发分为两大阶段，第一阶段先征询接收者，所属的类是否能动态添加方法以处理此未知消息。第二阶段涉及 “完整的消息转发机制”，在此阶段无法使用动态添加方法来处理未知消息。此时，runtime 会请求接收者以其他手段来处理与消息相关的方法调用。这里细分为两小步，首先，请接收者看看是否有其他对象能够处理这条消息。若有则将消息转发给该对象处理，消息转发结束。若没有“备援的接收者”，则启动完整的消息转发机制，runtime会将消息有关的全部细节都封装到 NSInvocation 中，再给接收者最后一次机会去设法解决。

##### 动态方法解析
在直至 NSObject 还没找到方法实现时，若未知消息是实例方法便会调用 -resolveInstanceMethod: 若未知消息是类方法便会调用 +resolveClassMethod: ，这两个方法类似，都是提供机会，动态添加一个处理该消息的方法。
```Objective-C
// 使用 runtime 需导入 runtime库，message.h 包含了 objc.h 和 runtime.h
#import <objc/message.h>

// 被添加的方法
void add_method(id self, SEL _cmd) {}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
  if (sel == @selector(method)) {

    // 添加 add_method 作为 method 的方法实现
    class_addMethod([self class], sel, (IMP)add_method, "v@:");
    return YES;
  }
  return [super resolveInstanceMethod:sel];
}
```

##### 备援接收者
如接收者未能动态添加方法去处理消息，那么下一步便会寻找能处理该消息的接收者。该步骤的处理方法为 -forwardingTargetForSelector: ，可以在此方法中根据方法来选用备援接收者。
```Objective-C
- (id)forwardingTargetForSelector:(SEL)selector {
  if (selector == @selector(method)) {

    //返回备援接收者去接收这个消息
    return [OtherObj new];
  }

  return [super forwardingTargetForSelector:selector];
}
```
此方法处理，在外部看，消息似乎是由对象自己处理的，这一现象与继承类似，子类调用自己未实现而父类实现了的方法，因此可用来模拟 “多继承” 特性。

##### 完整的消息转发
如果没有备援接收者，那么唯一能做的便是启用完整的消息转发机制。首先通过 -methodSignatureForSelector: 获取消息方法的细节(即方法参数、返回类型等)，封装成一个 NSMethodSignature(签名) 对象，如果获取消息方法失败，便返回nil，这时便会执行 doesNotRecognizeSelector: 方法终止程序，报 unrecognized selector 错误。
```Objective-C
- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
  if ([NSStringFromSelector(selector) isEqualToString:@"method"]) {

    return [NSMethodSignature signatureWithObjCTypes:"v@:"];
  }

  return [super methodSignatureForSelector:selector];
}
```
-methodSignatureForSelector: 中生成NSMethodSignature(签名)对象后，便用此签名对象生成 NSInvocation 对象并通过 -forwardInvocation: 方法发给目标对象，
在此方法中还有最后一次机会去处理未知消息。
```Objective-C
- (void)forwardInvocation:(NSInvocation *)invocation {

  SEL sel = invocation.selector;
  OtherObj *obj = [OtherObj new];
  if([obj respondsToSelector:sel]) {

    [invocation invokeWithTarget:obj];
  } else {

    [super forwardInvocation:invocation];;
  }
}
```
如果没有在此方法处理未知消息的话，runtime 会将消息转发给父类处理，直至 NSObject 根类都不能处理的话，便会执行 doesNotRecognizeSelector: 方法终止程序，报 unrecognized selector 错误。

![](http://r.photo.store.qq.com/psb?/V11AuUlP0teFMG/uK1ivvVobqWkC3xNa9It0ew*ZA2g6kxhK8dq6P1cZuE!/r/dLYAAAAAAAAA)


### Runtime 应用
#### 实现“多继承”特性
在消息的转发的第三步的寻找备援接收者中，对象调用未实现的方法，在 forwardingTargetForSelector: 中返回备用接受者去调用该方法，在表面就像是对象自己调用的，这就像子类调用自己未实现而父类实现了的方法。

即使我们利用转发消息来实现了“假”继承，但是NSObject类还是会将两者区分开。像respondsToSelector:和 isKindOfClass:这类方法只会考虑继承体系，不会考虑转发链。
```Objective-C
BOOL result1 = [objc respondsToSelector:@selector(method)];
BOOL result2 = [otherObj respondsToSelector:@selector(method)];
```
result1 为NO, result2 为 YES，因为objc中没有该方法的实现，也并不是 objc 调用的，而是备援接收者 otherObj。

因此如果非要制造假象，反应出这种“假”的继承关系，那么需要重新实现 respondsToSelector:和 isKindOfClass:。
```Objective-C
- (BOOL)respondsToSelector:(SEL)selector
{
  if ([super respondsToSelector:selector]) {

    return YES;
  } else {

    // 在此将判断对象变成假继承来源对象
    if ([otherObj respondsToSelector:selector]) {

      return YES;
    }
  }
  return NO;
}

```
除了respondsToSelector:和 isKindOfClass:之外，instancesRespondToSelector: 中也应该写一份转发算法。如果使用了协议，conformsToProtocol: 也一样需要重写。类似地，如果一个对象转发它接受的任何远程消息，它得给出一个 methodSignatureForSelector: 来返回准确的方法描述，这个方法会最终响应被转发的消息。比如一个对象能给它的替代者对象转发消息，它需要像下面这样实现 methodSignatureForSelector:
```Objective-C
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector
{
  NSMethodSignature* signature = [super methodSignatureForSelector:selector];
  if (!signature) {

    signature = [otherObj methodSignatureForSelector:selector];
  }

  return signature;
}
```
需要引起注意的一点，实现methodSignatureForSelector方法是一种先进的技术，只适用于没有其他解决方案的情况下。它不会作为继承的替代。如果必须使用这种技术，请确保完全理解类做的转发和转发的类的行为。请勿滥用！

#### Method Swizzling 方法交换
Method Swizzing是发生在运行时的，用于在运行时将两个Method的 IMP 和 SEL 进行交换，可以通过这一特性来实现 AOP 编程。方法交换实现如下：
```Objective-C
#import <objc/runtime.h>

+ (void)load {
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    Class class = [self class];
    SEL originalSelector = @selector(viewWillAppear:);
    SEL swizzledSelector = @selector(xxx_viewWillAppear:);

    Method originalMethod = class_getInstanceMethod(class,originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class,swizzledSelector);

    //judge the method named  swizzledMethod is already existed.
    BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));

    // if swizzledMethod is already existed.
    if (didAddMethod) {
      class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    } else {
      method_exchangeImplementations(originalMethod, swizzledMethod);
    }
  });
}

```
+load 会在类初始加载时调用，还有一个 +initialize 方法，但是 +initialize方法是懒加载调用的，所以你可能没有机会调用，因此我们应将 swizzling 的代码在 +load 调用。

在 +load 方法调用执行 swizzling 代码后，方法才会交换生效，如果再次调用便会再次交换变成原状 xxx_viewWillAppear 的 imp实现依然是 xxx_viewWillAppear IMP，因此交换方法的代码只应调用一次，所以我们应将 swizzling 的代码在 dispatch_once 中执行，请注意不要调用 [super load] ，因为如果父类也 Swizzling 同一个方法的 代码，父类的Swizzling就失效了。

#### AOP 面向切面
> 可以通过预编译方式和运行期动态代理实现在不修改源代码的情况下给程序动态统一添加功能的一种技术。
可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

比如像 身份控制、记录日志这些事务不属于任何一模块并且又很难抽象出一个模块，但是各模块都使用了，这类事务就像一个纵向切面，涉及许多类、类方法。

AOP 的多数操作是在 -forwardInvocation: 中完成的，一般会分为2个阶段，一个是方法注册，一个是方法调用。

首先会把类里面的某个要切片的方法的 IMP 加入到 Aspect 中，类方法里面如果有 -forwardingTargetForSelector: 方法的IMP，也要加入到 Aspect 中，然后对类要切片的方法和 -forwardingTargetForSelector: 方法的 IMP 进行替换，要切片的方法的 IMP 替换为 objc_msgForward() （消息转发函数）的 IMP，这样方法都注册完了。

当执行方法的时候，会去查找它的IMP，找到的 IMP 方法是 objc_msgForward() 的 IMP ，进入消息转发，由于未动态添加方法，于是开始查找备援接收对象。
查找备援接受者调用 -forwardingTargetForSelector: 这个方法，由于此方法 hook 过了，IMP 指向的是 hook 过的 -forwardingTargetForSelector: 方法。在 hook 过的 -forwardingTargetForSelector: 方法中设置返回 Aspect  作为备援接受者。

有了 Aspect 作为备援接受者之后，就会重新 objc_msgSend，从消息发送阶段重头开始。
objc_msgSend 找不到指定的IMP，再进行 -class_resolveMethod，这里也没有找到，forwardingTargetForSelector:这里也不做处理，接着就会methodSignatureForSelector。在methodSignatureForSelector方法中创建一个NSInvocation对象，传递给 Aspect 的 -forwardInvocation 方法。

Aspect 里面的 -forwardInvocation: 方法会干所有切面的事情，这里方法调用就完全由我们自定义了。注册方法的时候我们也加入了原来方法中的 method() 和 -forwardingTargetForSelector: 方法的IMP，这里我们可以在forwardInvocation方法中去执行这些 IMP。在执行这些 IMP 的前后都可以任意的插入任何 IMP 以达到切面的目的。

#### KVO 观察者
> KVO 是通过一种叫做is a-swizzling的技术实现的，isa指针指向维护分派表的对象的类。这个分派表实质上包含指向类实现的方法的指针以及其他数据。当一个观察者为一个对象的属性注册时，观察对象的 isa 指针被修改，指向一个中间类而不是真正的类。因此，isa指针的值不一定反映实例的实际类，不应该依赖isa指针来确定类成员。相反，应该使用类方法来确定对象实例的类。

在属性值发生变化的时候，肯定会调用其 setter 方法。因此，KVO的本质就是监听对象有没有调用被监听属性对应的 setter 方法。具体方法应是需要重写 setter 方法，如何重写的？实验如下：
```Objective-C
A *a = [[A alloc]init];
[a addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
```
打印观察isa指针的指向
```Objective-C
// a 注册 KVO 监听前
NSLog(@"Printing description of a->isa = %@",ClassMethodNames(object_getClass(a)));

Printing description of a->isa:
A

// a 注册 KVO 监听后
NSLog(@"Printing description of a->isa = %@",ClassMethodNames(object_getClass(a)));

Printing description of a->isa:
NSKVONotifying_A
```
在这个过程，被观察对象的 isa 指针从指向原来的 A 类，被KVO 机制修改为指向系统新创建的子类 NSKVONotifying_A 类，来实现当前类属性值改变的监听；
所以当我们从应用层面上看来，完全没有意识到有新的类出现，这是系统“隐瞒”了对 KVO 的底层实现过程，让我们误以为还是原来的类。但是此时如果我们创建一个新的名为“ NSKVONotifying_A ”的类，就会发现系统运行到注册 KVO 的那段代码时程序就会报 “ KVO failed to allocate class pair for name NSKVONotifying_A, automatic key-value observing will not work for this class ”，因为系统在注册监听的时候会动态创建了名为 NSKVONotifying_A 的中间类，并把 isa 指针指向这个中间类，但是由于我们创建一个NSKVONotifying_A 类，导致系统在注册监听的时候无法再创建一个NSKVONotifying_A 。


在新的类中会重写对应的 setter 方法，是为了在 setter方法中增加另外两个方法的调用，回调值的更改状态。
```Objective-C
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
//
// 值改变回调方法，-didChangeValueForKey:(NSString *)key 中调用
- (void)observeValueForKeyPath:(NSString *)keyPath
ofObject:(id)object
change:(NSDictionary *)change
context:(void *)context;

```
如果访问属性用的是 KVC，那么 runtime 会在 -setValue:forKey 中调用 -will/didChangeValueForKey: 方法。


#### Associated Object 关联对象
有时需要在对象中存放相关信息(增加属性)时，我们会从对象所属的类中继承一个子类，然后改用该子类。除此外我们可以使用“关联对象”将对象关联其他对象，对象通过 “键” 来区分，存储对象值时，可以指明“存储策略”维护“内存管理语义”。
```Objective-C
OBJC_ASSOCIATION_ASSIGH   =>  assign
OBJC_ASSOCIATION_RETAIN_NONATOMIC  => nonatomic,retain
OBJC_ASSOCIATION_COPY_NONATOMIC  => nonatomic,copy
OBJC_ASSOCIATION_RETAIN  => retain
OBJC_ASSOCIATION_COPY  => copy

// 关联对象方法
void objc_setAssociatedObject (id object, void*key, id value, objc_AssociationPolicy policy)

// 根据key获取关联对象值
id objc_getAssociatedObject(id object, void*key)

// 移除对象的全部关联对象
void objc_removeAssociatedObjects(id object)
```
这种方法可以为已有类增加属性
```Objective-C
#import "objc/runtime.h"

@interface NSObject (AssociatedObject)
@property (nonatomic, strong) id associatedObject;

@end

@implementation NSObject (AssociatedObject)
@dynamic associatedObject;

- (void)setAssociatedObject:(id)object {
  objc_setAssociatedObject(self, @selector(associatedObject), object, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (id)associatedObject {
  return objc_getAssociatedObject(self, @selector(associatedObject));
}
```

#### class_addMethod 动态增加方法
```Objective-C
class_addMethod(Class cls, SEL name, IMP imp, const char *types) 
```
- cls 对象
- name 要添加的方法名
- imp 要添加的方法实现
- types 方法类型字符串，如 "v@:" 表示返回类型为 void 无参数，"i@:" 表示返回类型为 int 无参数，"i@:@" 表示返回类型为 int 一个参数。


#### NSCoding 自动归档和自动解档
当类的属性多起来时，手写对应属性的归档和解档代码就多起来了，并且都是一样的代码。可以用 runtime 去获取类的所有成员属性，遍历属性，利用 KVC 去完成归档和解档。
```Objective-C
#import <objc/message.h>

- (void)encodeWithCoder:(NSCoder *)aCoder {

  unsigned int outCount = 0;
  Ivar *vars = class_copyIvarList([self class], &outCount);

  for (int i = 0; i < outCount; i ++) {

    Ivar var = vars[i];
    const char *name = ivar_getName(var);
    NSString *key = [NSString stringWithUTF8String:name]
    id value = [self valueForKey:key];
    [aCoder encodeObject:value forKey:key];
  }
}

- (id)initWithCoder:(NSCoder *)aDecoder {

  if (self = [super init]) {

    unsigned int outCount = 0;
    Ivar *vars = class_copyIvarList([self class], &outCount);

    for (int i = 0; i < outCount; i ++) {

      Ivar var = vars[i];
      const char *name = ivar_getName(var);
      NSString *key = [NSString stringWithUTF8String:name];
      id value = [aDecoder decodeObjectForKey:key];
      [self setValue:value forKey:key];
    }
  }
  return self;
}
```

#### 字典和模型互相转换
##### 字典转模型
```Objective-C
- (void)objectWithDictionary:(NSDictionary *)dictionary {

  for (NSString *key in dictionary.allKeys) {
    // 1. 根据key 从 dictionary中获取值
    id value = dictionary[key];
    // 2. 根据 key 从模型中获取属性
    objc_property_t property = class_getProperty([self class], key.UTF8String);
    // 3. 获取属性的类型，用于处理二级嵌套，此处未处理。
    unsigned int outCount = 0;
    objc_property_attribute_t *attributeList = property_copyAttributeList(property, &outCount);
    objc_property_attribute_t attribute = attributeList[0];
    NSString *typeString = [NSString stringWithUTF8String:attribute.value];

    // 4. 生成 setter 方法，使用 objc_msgSend 调用生成的 setter 方法，或者用 KVC，设置属性的值
    NSString *methodName = [NSString stringWithFormat:@"set%@%@:",[key substringToIndex:1].uppercaseString,[key substringFromIndex:1]];
    SEL setter = sel_registerName(methodName.UTF8String);
    if ([self respondsToSelector:setter]) {
      ((void (*) (id,SEL,id)) objc_msgSend) (self,setter,value);
    }
    free(attributeList);
  }
}
```

##### 模型转字典
```Objective-C
- (NSDictionary *)keyValuesWithObject {
  // 1. 获取属性列表
  unsigned int outCount = 0;
  objc_property_t *propertyList = class_copyPropertyList([self class], &outCount);
  NSMutableDictionary *dict = [NSMutableDictionary dictionary];
  // 2. 遍历属性列表
  for (int i = 0; i < outCount; i ++) {

    // 3. 生成getter方法，并用objc_msgSend调用
    objc_property_t property = propertyList[i];
    const char *propertyName = property_getName(property);
    SEL getter = sel_registerName(propertyName);

    if ([self respondsToSelector:getter]) {
     id value = ((id (*) (id,SEL)) objc_msgSend) (self,getter);

     // 4. 判断当前属性是不是Model
     if ([value isKindOfClass:[self class]] && value) {
        value = [value keyValuesWithObject];
     }

     // 5. 存入字典中，可以使用 objc_msgSend 调用生成的 getter 方法，或者用 KVC
     if (value) {

        NSString *key = [NSString stringWithUTF8String:propertyName];
        [dict setObject:value forKey:key];
     }
   }
  }
  free(propertyList);
  return dict;
}
```

### 总结
Runtime 能提供很多便利的解决方法，如 Method swizzling 能快速实现埋点统计，可以给分类添加属性等，但是这类应用需慎用，因使用为runtime 造成的 bug 很难定位。因此，若必须使用需记录使用了 runtime 的地方，以供日后调试使用 runtime 造成的 bug 使用，或者交接他人工作。

整理出处:
- [神经病院Objective-C Runtime入院第一天——isa和Class](https://www.jianshu.com/p/9d649ce6d0b8)
- [神经病院Objective-C Runtime住院第二天——消息发送与转发](https://www.jianshu.com/p/4d619b097e20)
- [神经病院Objective-C Runtime出院第三天——如何正确使用Runtime](https://www.jianshu.com/p/db6dc23834e3)
- [iOS Runtime详解](https://www.jianshu.com/p/6ebda3cd8052)
