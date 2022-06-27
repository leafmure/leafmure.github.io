---
title: AutoreleasePool详解
date: 2022-03-11 16:25:19
categories: iOS
tags: AutoreleasePool
keywords: AutoreleasePool,Autorelease,内存管理
description:
images:
---
### 一、前言
iOS 的内存管理是通过引用计数管理的，当对象的引用计数为 0 时，说明该对象要释放回收内存了。在 iOS 5 之前，内存管理是需要开发者负责的，需要手动调用 retain、release、autorelease 去操作对象的引用计数，这种手动管理模式称为 MRC（Manual Reference Counting）。iOS 5 后引入 ARC（Automatic Reference Counting），LLVM 编译器会在编译时自动插入 retain、release、autorelease 的调用方法。autorelease 标记的对象会添加到 AutoreleasePool，由 AutoreleasePool 管理内存释放。
<!-- more -->
### 二、AutoreleasePool 是什么？
#### 2.1  创建自动释放池
在 MRC 下，我们可以用 NSAutoreleasePool 或者@autoreleasepool 去手动创建一个 AutoreleasePool
```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
Class *obj = [[Class alloc] init];
/// 加入 pool
[obj autorelease];
[pool release]
```
在 ARC 下，已经禁止使用 NSAutoreleasePool 类以及 retain、release、autorelease 等调用方法，只能通过 @autoreleasepool 创建自动释放池。
```
@autoreleasepool {
    /// 括号中，存在 autorelease 对象时，自动加入 pool 中。
}
```
#### 2.2  @autoreleasepool 源码分析
我们估计平时很少使用 @autoreleasepool 显式创建 AutoreleasePool，在工程的 Main 文件，倒是能看到 @autoreleasepool 使用，mian 函数如下：
```
int main(int argc, const char * argv[]) {
    @autoreleasepool {}
    return 0;
}
```
通过 Clang 命令将上述代码转换成 C++ 源码：
```
/// -> clang -rewrite-objc main.m
/// -> main.cpp
......
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
......
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ 
    { __AtAutoreleasePool __autoreleasepool;  }
    return 0;
}
......
```
我们可以看，@autoreleasepool 实际上是创建了一个  __AtAutoreleasePool 结构体对象，在创建__AtAutoreleasePool结构体时会在构造函数中调用objc_autoreleasePoolPush()函数，并返回一个atautoreleasepoolobj (POOL_BOUNDARY存放的内存地址，下面会讲到)。在释放__AtAutoreleasePool结构体时会在析构函数中调用objc_autoreleasePoolPop()函数，并将atautoreleasepoolobj传入。

#### 2.3  AutoreleasePoolPage
objc_autoreleasePoolPush() 和 objc_autoreleasePoolPop() 这两个函数做了什么呢，让我们去 objc4 源码 查看：
```
/// -> NSObject.mm
void * objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```
从上述代码可以看出，objc_autoreleasePoolPush() 和 objc_autoreleasePoolPop() 内部是通过 AutoreleasePoolPage 去调用 push() 和 pop() 函数，AutoreleasePoolPage 明显是 @autoreleasepool 基础实现类。AutoreleasePoolPage 是如何管理 autorelease 对象的呢？让我们来看一下 AutoreleasePoolPage 的实现结构：
```
/// -> NSObject.mm
class AutoreleasePoolPage : private AutoreleasePoolPageData
{
	friend struct thread_data_t;

public:
	static size_t const SIZE =
#if PROTECT_AUTORELEASEPOOL
		PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
		PAGE_MIN_SIZE;  // size and alignment, power of 2
#endif
    
private:
	static pthread_key_t const key = AUTORELEASE_POOL_KEY;
	static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
	static size_t const COUNT = SIZE / sizeof(id);
    static size_t const MAX_FAULTS = 2;

    // EMPTY_POOL_PLACEHOLDER is stored in TLS when exactly one pool is 
    // pushed and it has never contained any objects. This saves memory 
    // when the top level (i.e. libdispatch) pushes and pops pools but 
    // never uses them.
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)
#   define POOL_BOUNDARY nil
......
    AutoreleasePoolPage(AutoreleasePoolPage *newParent) :
		AutoreleasePoolPageData(begin(),
								objc_thread_self(),
						      newParent,
								newParent ? 1+newParent->depth : 0,
								newParent ? newParent->hiwat : 0)
{
        if (objc::PageCountWarning != -1) {
            checkTooMuchAutorelease();
        }


        if (parent) {
            parent->check();
            ASSERT(!parent->child);
            parent->unprotect();
            parent->child = this;
            parent->protect();
        }
        protect();
    }
......
}


/// -> NSObject-internal.h
struct AutoreleasePoolPageData
{
#if SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS
    struct AutoreleasePoolEntry {
        uintptr_t ptr: 48;
        uintptr_t count: 16;
        static const uintptr_t maxCount = 65535; // 2^16 - 1
    };
    static_assert((AutoreleasePoolEntry){ .ptr = MACH_VM_MAX_ADDRESS }.ptr == MACH_VM_MAX_ADDRESS, "MACH_VM_MAX_ADDRESS doesn't fit into AutoreleasePoolEntry::ptr!");
#endif

	magic_t const magic;  // 用来校验 Page 的结构是否完整
	__unsafe_unretained id *next;  // 指向下一个可存放 autorelease 对象地址的位置，初始化指向 begin()
	pthread_t const thread;  // 指向当前线程
	AutoreleasePoolPage * const parent;  // 指向父结点，首结点的 parent 为 nil
	AutoreleasePoolPage *child;  // 指向子结点，尾结点的 child  为 nil
	uint32_t const depth;  // Page 的深度，从 0 开始递增
	uint32_t hiwat;
......
};
```
AutoreleasePoolPage 类继承自 AutoreleasePoolPageData，从 AutoreleasePoolPageData 的实现结构可得知：
- AutoreleasePool 是以 AutoreleasePoolPage 为节点组合的“双向链表”，AutoreleasePoolPage 创建时，将新创建的 Page 的 parent 指针指向parentPage，将 parentPage 的 child 指针指向自己。
- AutoreleasePoolPage 的 thread 成员变量指向当前 AutoreleasePool 所在的线程，说明 AutoreleasePool 和线程是一一对应的
- AutoreleasePoolPage 的最大 Size 为一页虚拟内存页的大小-4096 字节，其中 56 个字节用来存放它内部的成员变量，剩下的空间（4040个字节）用来存放 autorelease 对象 的地址，它的内存分布图如下：
![image](https://lianghuii.com/postImage/AutoreleasePool详解/psb-01.png)

AutoreleasePoolPage 拥有 begin、end、empty、full 这些方法来描述 Page 的容量情况，实现代码如下：
```
id * begin() {
   /// Page自己的地址+Page对象的大小56个字节；
   return (id *) ((uint8_t *)this+sizeof(*this));
}

id * end() {
   /// Page自己的地址+4096个字节；
   return (id *) ((uint8_t *)this+SIZE);
}

bool empty() {
   /// 判断Page是否为空的条件
   return next == begin();
}

bool full() { 
   /// 判断Page是否已满的条件
   return next == end();
}
```

#### 2.4  AutoreleasePoolPage 的 push() 和 pop()
##### 2.4.1  push()
AutoreleasePoolPage 是如何操纵 autorelease 对象的添加呢，让我们先看一下 push() 的源码实现：
```
/// NSObject.mm
static inline void *push() {
   id *dest;
   if (slowpath(DebugPoolAllocation)) {
      // Each autorelease pool starts on a new pool page.
      dest = autoreleaseNewPage(POOL_BOUNDARY);
   } else {
      dest = autoreleaseFast(POOL_BOUNDARY);
   }
   ASSERT(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
   return dest;
}
```
当 AutoreleasePool 创建后，AutoreleasePoolPage 会调用 push() 方法加入 POOL_BOUNDARY 哨兵对象。push() 内部加入对象时，会判断是否 AutoreleasePoolPage 对象存在，没有的话，调用  autoreleaseNewPage() ，让我们看一下 autoreleaseNewPage 的实现：
```
static __attribute__((noinline))
id *autoreleaseNewPage(id obj)
{
   AutoreleasePoolPage *page = hotPage();
   if (page) return autoreleaseFullPage(obj, page);
   else return autoreleaseNoPage(obj);
}

static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
     // The hot page is full. 
     // Step to the next non-full page, adding a new page if necessary.
     // Then add the object to that page.
     ASSERT(page == hotPage());
     ASSERT(page->full()  ||  DebugPoolAllocation);

     do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
     } while (page->full());

     setHotPage(page);
     return page->add(obj);
}

static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    // "No page" could mean no pool has been pushed
    // or an empty placeholder pool has been pushed and has no contents yet
    ASSERT(!hotPage());

    bool pushExtraBoundary = false;
    if (haveEmptyPoolPlaceholder()) {
       // We are pushing a second pool over the empty placeholder pool
       // or pushing the first object into the empty placeholder pool.
       // Before doing that, push a pool boundary on behalf of the pool 
       // that is currently represented by the empty placeholder.
       pushExtraBoundary = true;
    }
    else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
       // We are pushing an object with no pool in place, 
       // and no-pool debugging was requested by environment.
       _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                         "autoreleased with no pool in place - "
                         "just leaking - break on "
                         "objc_autoreleaseNoPool() to debug", 
                         objc_thread_self(), (void*)obj, object_getClassName(obj));
       objc_autoreleaseNoPool(obj);
       return nil;
    }
    else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
       // We are pushing a pool with no pool in place,
       // and alloc-per-pool debugging was not requested.
       // Install and return the empty pool placeholder.
       return setEmptyPoolPlaceholder();
    }

    // We are pushing an object or a non-placeholder'd pool.
    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);
        
    // Push a boundary on behalf of the previously-placeholder'd pool.
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY);
    }
        
    // Push the requested object or pool.
    return page->add(obj);
 }
```
autoreleaseNewPage() 内部逻辑如下：
- 首先，调用 hotPage() 获取 AutoreleasePool 双向链表最后一个 Page。
- 如果 Page 存在，调用  autoreleaseFullPage()：
   - 1、当前 Page 未满时，通过 page→add(obj) 将 autorelease 对象加到当前 Page 中。
   - 2、当前 Page 已满时，通过 while 循环查找未满的Page，若查找到，将 autorelease 对象添加进去。若未查找到，创建一个新的 Page，并将 autorelease 对象添加进去。
- 如果 Page 不存在，调用 autoreleaseNoPage() 创建第一个 AutoreleasePoolPage，并将 autorelease 对象添加进去。

再让我们看一下 push() 方法中，当存在 AutoreleasePoolPage 时 autoreleaseFast()  函数实现：
```
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```
autoreleaseFast() 里处理的事情跟 autoreleaseNewPage() 差不多，都是判断 hotPage()的存在以及是否满容量做相对应的处理。

对象要添加到 AutoreleasePool 中，就要调用 autorelease()，其内部也是通过 autoreleaseFast() 方法添加的，代码实现如下：
```
static inline id autorelease(id obj)
{
    ASSERT(!obj->isTaggedPointerOrNil());
    id *dest __unused = autoreleaseFast(obj);
#if SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS
    ASSERT(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  (id)((AutoreleasePoolEntry *)dest)->ptr == obj);
#else
    ASSERT(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
#endif
    return obj;
}
```
##### 2.4.2  pop()
前面我们已经了解了 autorelease 对象的添加，接下来让我们了解 autorelease 对象的释放：
```
static inline void
pop(void *token)
{
    AutoreleasePoolPage *page;
    id *stop;
    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
    // Popping the top-level placeholder pool.
    page = hotPage();
    if (!page) {
       // Pool was never used. Clear the placeholder.
       return setHotPage(nil);
    }
       // Pool was used. Pop its contents normally.
       // Pool pages remain allocated for re-use as usual.
       page = coldPage();
       token = page->begin();
    } else {
       page = pageForPointer(token);
    }

    stop = (id *)token;
    if (*stop != POOL_BOUNDARY) {
       if (stop == page->begin()  &&  !page->parent) {
          // Start of coldest page may correctly not be POOL_BOUNDARY:
          // 1. top-level pool is popped, leaving the cold page in place
          // 2. an object is autoreleased with no pool
       } else {
          // Error. For bincompat purposes this is not 
          // fatal in executables built with old SDKs.
          return badPop(token);
       }
    }

    if (slowpath(PrintPoolHiwat || DebugPoolAllocation || DebugMissingPools)) {
       return popPageDebug(token, page, stop);
    }
    return popPage<false>(token, page, stop);
}
```
pop() 方法有个入参数 token，该 token 其实就是 AutoreleasePool 创建时 push 的 POOL_BOUNDARY 哨兵对象。当 AutoreleasePool 销毁时，pop() 方法会依次向 AutoreleasePool 中的对象发送 release 消息，直到遇到 POOL_BOUNDARY 哨兵对象时，说明 AutoreleasePool 内的对象已全部释放。

pop() 方法内部流程如下：
- 判断 token 是不是 EMPTY_POOL_PLACEHOLDER，如是的话，判断 hotPage 存不存在：
   - 1、若不存在，证明该 AutoreleasePool 是空池，清除 EMPTY_POOL_PLACEHOLDER 标识，销毁该池。
   - 2、若存在，调用 coldPage() 重新分配 Page 返回第一个 Page对象，将 Page 的首对象赋值给 token。
- 如果不是的话 EMPTY_POOL_PLACEHOLDER，就通过 pageForPointer(token) 拿到 token 所在的Page（自动释放池的首个 Page）
- 若 token 对象非 POOL_BOUNDARY，说明 Page 的首对象不正确，如果 token 是 page 的首对象且 page 是第一页，说明这是  coldPage 或者对象已经自动释放。如果不是前面的情况的话，说明 AutoreleasePool 异常，调用 badPop() 函数销毁 AutoreleasePool。
- 调用 popPage() 函数，内部会调用 releaseUntil() 释放 autorelease 对象。

让我们来看一下 releaseUntil() 是如何释放对象的：
```
void releaseUntil(id *stop) 
{
    // Not recursive: we don't want to blow out the stack 
    // if a thread accumulates a stupendous amount of garbage
        
    while (this->next != stop) {
       // Restart from hotPage() every time, in case -release 
       // autoreleased more objects
       AutoreleasePoolPage *page = hotPage();

       // fixme I think this `while` can be `if`, but I can't prove it
       while (page->empty()) {
          page = page->parent;
          setHotPage(page);
       }

       page->unprotect();
#if SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS
       AutoreleasePoolEntry* entry = (AutoreleasePoolEntry*) --page->next;

       // create an obj with the zeroed out top byte and release that
       id obj = (id)entry->ptr;
       int count = (int)entry->count;  // grab these before memset
#else
       id obj = *--page->next;
#endif
       memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
       page->protect();

       if (obj != POOL_BOUNDARY) {
#if SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS
         // release count+1 times since it is count of the additional
         // autoreleases beyond the first one
         for (int i = 0; i < count + 1; i++) {
             objc_release(obj);
         }
#else
         objc_release(obj);
#endif
       }
    }

    setHotPage(this);

#if DEBUG
    // we expect any children to be completely empty
    for (AutoreleasePoolPage *page = child; page; page = page->child) {
       ASSERT(page->empty());
    }
#endif
}
```
releaseUntil() 内通过 while 循环，从最后一个 autorelease 对象开始往前依次调用 release() 释放对象，直到 POOL_BOUNDARY 哨兵对象退出循环。

### 三、  POOL_BOUNDARY 的作用
从  AutoreleasePoolPage pop() 的过程，我们可以知道 POOL_BOUNDARY 明显的一点就是作为一个边界对象，这样 AutoreleasePoolPage 可以保证将该释放的对象释放，这一作用在 @autoreleasepool 嵌套使用中更能体现，先让我们来看一下单个 @autoreleasepool 管理 autorelease 对象的内存分布。
```
extern void _objc_autoreleasePoolPrint(void);

_objc_autoreleasePoolPrint();
@autoreleasepool {
    _objc_autoreleasePoolPrint();
    NSObject *obj = [[[NSObject alloc] init] autorelease];
    _objc_autoreleasePoolPrint();
    NSObject *obj2 = [[[NSObject alloc] init] autorelease];
    _objc_autoreleasePoolPrint();
}
```
_objc_autoreleasePoolPrint() 可以在控制台输出当前 AutoreleasePool 中的对象信息，_objc_autoreleasePoolPrint() Runtime 中的隐匿函数，所以得
extern void _objc_autoreleasePoolPrint(void)，以下是输出信息：
```
objc[1529]: ##############
objc[1529]: AUTORELEASE POOLS for thread 0x10008c580
objc[1529]: 0 releases pending.
objc[1529]: [0x100810000]  ................  PAGE  (hot) (cold)
objc[1529]: ##############
objc[1529]: ##############
objc[1529]: AUTORELEASE POOLS for thread 0x10008c580
objc[1529]: 1 releases pending.
objc[1529]: [0x100810000]  ................  PAGE  (hot) (cold)
objc[1529]: [0x100810038]  ################  POOL 0x100810038 // POOL_BOUNDARY
objc[1529]: ##############
objc[1529]: ##############
objc[1529]: AUTORELEASE POOLS for thread 0x10008c580
objc[1529]: 2 releases pending.
objc[1529]: [0x100810000]  ................  PAGE  (hot) (cold)
objc[1529]: [0x100810038]  ################  POOL 0x100810038
objc[1529]: [0x100810040]       0x1007b6db0  NSObject
objc[1529]: ##############
objc[1529]: ##############
objc[1529]: AUTORELEASE POOLS for thread 0x10008c580
objc[1529]: 3 releases pending.
objc[1529]: [0x100810000]  ................  PAGE  (hot) (cold)
objc[1529]: [0x100810038]  ################  POOL 0x100810038
objc[1529]: [0x100810040]       0x1007b6db0  NSObject
objc[1529]: [0x100810048]       0x1007b5760  NSObject
objc[1529]: ##############
Program ended with exit code: 0
```
从输出信息来看，它的内存分布图如下：
![image](https://lianghuii.com/postImage/AutoreleasePool详解/psb-02.png)

再来看看多层 @autoreleasepool 嵌套情况：
```
int main(int argc, const char * argv[]) {
    _objc_autoreleasePoolPrint();
    @autoreleasepool {
        _objc_autoreleasePoolPrint();
        NSObject *obj = [[[NSObject alloc] init] autorelease];
        _objc_autoreleasePoolPrint();
        NSObject *obj2 = [[[NSObject alloc] init] autorelease];
        _objc_autoreleasePoolPrint();
        @autoreleasepool {
            _objc_autoreleasePoolPrint();
            NSObject *obj3 = [[[NSObject alloc] init] autorelease];
            _objc_autoreleasePoolPrint();
            NSObject *obj4 = [[[NSObject alloc] init] autorelease];
            _objc_autoreleasePoolPrint();
        }
    }
    return 0;
}
```
每个 AutoreleasePool 创建后都会 push 一个 POOL_BOUNDARY 对象，区别其他 AutoreleasePool 管理的对象，嵌套后分布图如下
![image](https://lianghuii.com/postImage/AutoreleasePool详解/psb-03.png)

### 四、AutoreleasePool 的创建时机和释放时机
AutoreleasePool 创建一般是这两种方式：

开启 RunLoop，RunLoop 会自动管理 AutoreleasePool 的创建和释放，Runloop 中注册了两个 Observer，回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。两个 Observer 如下：
- 1、监测 Entry 事件，回调里自动创建自动释放池，order为 -214748364， 优先级最高，保证创建释放池发生在其他所有回调之前；
- 2、监测 BeforeWaiting 及 Exit 事件，BeforeWaiting 时调用 _objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池。Exit 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

通过 @autoreleasepool 手动创建，在离开 @autoreleasepool 所处的作用域时， AutoreleasePool 释放
主线程因为会自动开 RunLoop，因此 AutoreleasePool 会自动创建，但其他线程的 RunLoop 是默认未创建的，那它们的  autorelease 对象如何管理呢？子线程中如果没有开启 RunLoop，当存在 autorelease 对象时，就会创建 AutoreleasePool 并添加到 AutoreleasePool 中，等线程销毁时，AutoreleasePool 就会释放。

@autoreleasepool 的手动创建，一般用于避免内存峰值，比如在 for 循环中创建了大量的临时对象，我们需要在循环体内创建 AutoreleasePool ，当一次循环结束后，释放临时对象，数组系统遍历方法内部为避免这种情况，内部会自动创建 AutoreleasePool，但请注意只有 Autorelease类型的对象才会交给AutoreleasePool去管理。
```
[array enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
    // 这里被一个局部@autoreleasepool包围着
}];
```

### 五、什么对象会通过 autorelease 管理
只有 Autorelease 类型的对象才会交给AutoreleasePool去管理，那什么样的对象才是 Autorelease 类型的呢？

编译器会检查方法名是否以alloc, new, copy, mutableCopy 开始，如果不是则自动将返回值的对象注册到 AutoreleasePool 中，比如一些类方法；
iOS 5 及之前的编译器，关键字 __weak 修饰的对象，会自动加入AutoreleasePool。iOS 5 及之后的编译器，则直接调用的 release，不会加入 AutoreleasePool；
id 指针 (id *) 和对象指针（NSError *），会自动加上关键字 __autorealeasing，加入 AutoreleasePool。

我们其实可以通过 objc_autoreleaseReturnValue 函数来标识一个对象是否加入到 AutoreleasePool 中去。同时该方法通过 TLS（Thread Local Storage）线程局部存储做了些优化，符号优化情况的不会加入到 AutoreleasePool 中。系统将一块内存作为某个线程专有的存储，以key-value的形式进行读写，比如在非arm架构下，使用 pthread 提供的方法实现：

void* pthread_getspecific(pthread_key_t);
int pthread_setspecific(pthread_key_t , const void *);
在返回值身上调用 objc_autoreleaseReturnValue 方法时，runtime 将这个返回值 object 储存在TLS中，然后直接返回这个object（不调用autorelease），同时，在外部接收这个返回值的 objc_retainAutoreleasedReturnValue里，发现TLS中正好存了这个对象，那么直接返回这个object（不调用retain）。
于是乎，调用方和被调方利用TLS做中转，很有默契的免去了对返回值的内存管理。



