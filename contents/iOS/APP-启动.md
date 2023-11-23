---
title: App 启动
date: 2021-09-13 19:19:46
categories:
- iOS
tags:
- 启动优化
keywords:
- App启动流程,启动优化
description:
images:
---

### APP 启动流程
App 启动情况有：
- 冷启动：系统里没有任何进程的缓存信息，如：重启手机后直接启动 App，或者杀死App 等待一段时间，缓存被清除后再启动。
- 热启动：系统里还存在进程的缓存信息，如：刚把App 杀死后立即启动App。

启动过程中分为两阶段：pre-main、main
<!-- more -->
#### pre-main
pre-main 是在 main 函数执行前的阶段，在这期间主要有以下几个阶段：
1. Load dylibs
2. Rebase images
3. Bind images
4. Objc SetUp
5. Initializers

> image 指的是一个被编译过的符号、代码等的二进制文件

##### dyld(动态链接器)
dyld 是启动的辅助程序，是 in-process 的，即启动的时候会把 dyld 加载到进程的地址空间里，然后把后续的启动过程交给 dyld。其过程如下：

1. 系统先读取App的可执行文件(Mach-O文件)，从里面获得 dyld 的路径，然后加载 dyld。
2. dyld 去初始化运行环境，开启缓存策略(缓存命中)，加载程序相关依赖库(其中也包含可执行文件)，并对这些库进行链接。
3. 最后调用每个依赖库的初始化方法，在这一步，runtime被初始化。
4. 当所有依赖库的初始化后，轮到最后一位(程序可执行文件)进行初始化，在这时 runtime 会对项目中所有类进行类结构初始化，然后调用所有的load方法。
5. 最后 dyld 返回 main函数地址，main函数被调用。

对于库的链接分为：动态链接、静态链接两种方式，系统 framework 都是动态链接的。

系统级的动态库有：
- UIKit.framework
- Foundation.framework
- CoreFoundation.framework
- libobjc.A.dylib
- libSystem.dylib

libobjc 即 objc 和 runtime，libSystem 中包含了很多系统级别 lib，如：

- libdispatch ( GCD )
- libsystem_c ( C语言库 )
- libsystem_blocks ( Block )
- libcommonCrypto ( 加密库，比如常用的 md5 函数 )

动态链接和静态链接：

- 动态链接：在程序运行时用到该库的时候加载库，库文件在内存和磁盘只存在一份，减少可执行文件的体积。另外采用引用方式，可以对库进行更新，但是外部依赖库出现问题，会造成程序运行异常。
- 静态链接：在编译时就加载完所有库，将库文件复制一份加入可执行文件。这种方式会增加可执行文件的体积，也不能对库进行更新，但其好处在于不与外部环境产生依赖，每个进程中独立存在一份。

##### Rebase images
在dylib的加载过程中，系统为了安全考虑，采用 ASLR(address space layout randomization) 机制，在 mmap 到虚拟内存的时候，Mach-O 会在随机的地址上加载，和之前指针指向的地址会有一个偏差，dyld 需要修正这个偏差，来指向正确的地址并以 page 为单位进行加密验证，保证不会被篡改。

##### Bind images
- Binding：设置指向 image 外部的指针，比如：objc代码中需要使用到NSObject，即符号_OBJC_CLASS_$_NSObject，但是这个符号又不在我们的二进制中，在系统库 Foundation.framework中，因此就需要 binding 这个操作将对应关系绑定到一起。
- LazyBinding：在加载动态库的时候不会立即binding, 当时当第一次调用这个方法的时候再实施binding。实现的方法也很简单： 通过 dyld_stub_binder 这个符号来做。lazyBinding的方法第一次会调用到dyld_stub_binder, 然后dyld_stub_binder负责找到真实的方法，并且将地址bind到桩上，下一次就不用再bind了。

##### Objc SetUp
1. 读取二进制文件的 DATA 段内容，找到与 objc 相关的信息。
2. 注册 Objc 类，ObjC Runtime 需要维护一张映射类名与类的全局表。当加载一个 dylib 时，其定义的所有的类都需要被注册到这个全局表中。
3. 读取 category 的信息，把category的定义插入方法列表。
4. 确保 selector 的唯一性

##### Initializers
1. dyld开始将程序二进制文件初始化
2. 交由ImageLoader读取image，其中包含了我们的类、方法等各种符号。
3. 由于runtime向dyld绑定了回调，当image加载到内存后，dyld会通知runtime进行处理。
4. runtime接手后调用 mapimages 做解析和处理，接下来loadimages中调用 call_load_methods 方法，遍历所有加载进来的Class，按继承层级依次调用Class的 +load() 方法和其 Category 的 +load() 方法

#### main
pre-main 阶段完成后，main 阶段的工作如下：
1. 调用 main() 函数
2. 调用 UIApplicationMain() 
3. 调用 application:didFinishLaunchingWithOptions 方法

##### UIApplicationMain()
1. 读取 Info.plist 文件 NSPrincipalClass 键的值作为 principalClassName 参数值，初始化 UIApplication 应用程序对象或其子类对象。
2. 根据 delegateClassName 将对应类设置为 UIApplication 应用程序对象的代理
3. 启动主RunLoop 并开始接收事件
4. 应用加载完毕，代理对象调用 application:didFinishLaunchingWithOptions 委托函数。

### APP 启动优化
#### pre-main 阶段优化

> 在 Xcode 中 Edit scheme -> Run -> Auguments -> Environment Variables 添加 DYLD_PRINT_STATISTICS 变量值为1，可在控制台看到 pre-main阶段的耗时详情。

##### Load dylibs
该阶段工作量主要是 dyld 加载应用依赖的 dylib，分析所依赖的动态库，找到动态库的 mach-o文件并打开，验证文件的有效性，在系统核心注册文件签名，对动态库的每一个 segment 调用 mmap()。对于这一阶段的优化地方有：
- 减少非系统库的依赖
- 将动态库改为静态库，静态库在编译时便已经加载到可执行文件了。

##### Rebase & Bind images
Rebase 工作量在于：需要重复不断地对 __DATA 段中需要 rebase 的指针加上这个偏移量，I/O操作多；Bind 工作量在于：查阅符号表，设置那些指向 dylib 外部的指针，CPU 计算量大。对于这一阶段的优化地方有：
- 减少Objc类、selector、category 数量，把未使用的类和函数删除。
- 减少C++虚函数数量
- Swift 项目尽量使用 Struct 替代类

##### Initializers
这一阶段会调用类和分类的 +load() 方法，调用C/C++ 中的构造器函数和创建非基本类型的C++静态全局变量。对于这一阶段的优化地方有：
- 减少在 +load() 方法的逻辑，不要有耗时操作。
- 减少 C/C++ 中的构造器函数，不要使用 atribute((constructor)) 将方法显式标记为构造器函数，而是由初始化方法调用。
- 减少非基本类型的C++静态全局变量

#### main 阶段优化
main 阶段的优化主要在 application:didFinishLaunchingWithOptions 方法里的逻辑，以及首页控制器 viewDidLoad()、viewWillApper() 方法里的逻辑。主要优化有：

- 减少主要方法的事件处理，去除多余的代码。
- 优化主要方法的代码逻辑，对于耗时操作尽量采用异步处理，把可以延迟执行的逻辑，做延迟执行处理，可以懒加载的做懒加载处理。

##### APP 启动优化-二进制重排（os：优化不明显）
###### Page Fault
早期在没有虚拟内存概念的时候，都是直接访问真实的物理地址，应用进程加载到内存中是完整加载的并且多个应用程序在内存上是有序排列的，因此在进程中可以通过地址偏移获取其他进程的内存数据，这存在安全隐患。随着软件功能的扩展的，软件所占用的内存也越来越多，然而进程所占用的内存空间并不是完全被利用，所以造成内存浪费。

引用虚拟内存后，虚拟内存会维护一张虚拟地址对应真实物理地址映射表。进程在虚拟内存中内存地址是连续的，但对应的物理地址不一定是连续的。

虚拟内存和物理内存通过映射表进行映射，但是这个映射并不可能是一一对应的，那样就太过浪费内存了。为了提高效率和方便管理，又对虚拟内存和物理内存又进行分页（Page，Mac OS 系统中, 一页为4KB，iOS 系统中，一页为16KB）。当进程被加载到内存中时, 并不会将整个进程加载到内存中，只会加载目前进程用到的数据。

当进程访问一个虚拟内存Page而对应的物理内存却不存在时，会触发一次缺页中断（Page Fault），分配物理内存，有需要的话会从磁盘 mmap 读入数据，通过App Store渠道分发的App，Page Fault还会进行签名验证，所以一次Page Fault的耗时比想象的要多。

###### 重排原理
函数编译在 mach-o 中的位置是根据 ld ( Xcode 的链接器) 的编译顺序并非调用顺序来的（即：Build Phases中Compile Sources 里文件的排列顺序）。当程序启动时如果需要调用 method2 和 method4，为了执行对应的代码，系统必须进行两个Page Fault。但如果我们把 method2 和 method4排布到一起，那么只需要一个Page Fault即可，这就是二进制文件重排的核心原理。
![image](https://meanmouse.github.io/pic/postImage/AppStart/psb-1.png)
###### System Trace
检测 page fault 次数以及测试优化结果，可以采用 Instruments 的 System Trace 工具，选择App冷启动作为测试环境。

![image](https://meanmouse.github.io/pic/postImage/AppStart/psb-2.png)

File Backed Page In 即为 page fault 次数

###### Linkmap
Linkmap是iOS编译过程的中间产物，记录了二进制文件的布局，需要在 Xcode 的 Build Settings 里开启Write Link Map File。linkmap主要包括三大部分：
- Object Files 生成二进制用到的link单元的路径和文件编号
- Sections 记录Mach-O每个Segment/section的地址范围
- Symbols 按顺序记录每个符号的地址范围

通过 Symbols 的顺序可以知道是否重排成功。
> Linkmap file 默认地址是：/Users/用户名/Library/Developer/Xcode/DerivedData/项目名/Build/Intermediates.noindex/项目名.build/Debug-iphoneos/项目名.build/App名称-LinkMap-normal-arm64.txt

###### Order File
Xcode 的链接器叫做 ld, ld 有一个参数叫 Order File（Build Settings）, 可以通过这个参数配置一个 order 文件的路径，ld 会根据 order 文件里的符号顺序去进行生成对应的 mach-O，所以需要收集启动时用到的符号并写入到 order 文件里。对于 Order File 里不存在的符号，ld 会忽略。

###### clang SanitizerCoverage 插桩
在 [Clang documentation ](https://clang.llvm.org/docs/SanitizerCoverage.html#tracing-pcs) 中可以看到 LLVM 官方对 SanitizerCoverage 的详细介绍，包含了示例代码。简单来说 SanitizerCoverage 是 Clang 内置的一个代码覆盖工具。它把一系列以 __sanitizer_cov_trace_pc_ 为前缀的函数调用插入到用户定义的函数里，借此实现了全局 AOP。其覆盖之广，包含 Swift/Objective-C/C/C++ 等语言，Method/Function/Block 全支持。

开启 SanitizerCoverage 的方法是：在 build settings 里的 “Other C Flags” 中添加 -fsanitize-coverage=func,trace-pc-guard。如果含有 Swift 代码的话，还需要在 “Other Swift Flags” 中加入 -sanitize-coverage=func 和 -sanitize=undefined。所有链接到 App 中的二进制都需要开启 SanitizerCoverage，这样才能完全覆盖到所有调，另外，设置了 -fsanitize-coverage=func,trace-pc-guard 则必须实现 __sanitizer_cov_trace_pc_guard_init 和 __sanitizer_cov_trace_pc_guard 函数。

```Objective-C
官方示例
/// 插入 __sanitizer_cov_trace_pc_guard 函数的符号范围回调
void __sanitizer_cov_trace_pc_guard_init(uint32_t *start,
                                                    uint32_t *stop) {
  static uint64_t N;  // Counter for the guards.
  if (start == stop || *start) return;  // Initialize only once.
  printf("INIT: %p %p\n", start, stop);
  for (uint32_t *x = start; x < stop; x++)
    *x = ++N;  // Guards should start from 1.
}

void __sanitizer_cov_trace_pc_guard(uint32_t *guard) {
  if (!*guard) return;  // Duplicate the guard check.

  void *PC = __builtin_return_address(0);
  char PcDescr[1024];
  //__sanitizer_symbolize_pc(PC, "%p %F %L", PcDescr, sizeof(PcDescr));
  printf("guard: %p %x PC %s\n", guard, *guard, PcDescr);
}
```
编译时在每一个函数内部第一行代码处插入 __sanitizer_cov_trace_pc_guard 函数，所以可以通过该函数获取函数符号。函数嵌套时 , 在跳转子函数时，都会保存下一条指令的地址在 x30 (又叫 lr 寄存器) 里。

例如：A函数中调用了B函数，在 arm 汇编中的体现为 bl + 0x**** 指令，该指令会首先将 A函数下一条汇编指令的地址保存在 x30 寄存器中。然后在跳转到 bl 后面传递的指定地址去执行。
bl 能实现跳转到某个地址的汇编指令，其原理就是修改 pc 寄存器的值来指向到要跳转的地址，而且实际上 B 函数中也会对 x29 / x30 寄存器的值做保护，防止子函数又跳转其他函数会覆盖掉 x30 的值 , 当然叶子函数除外。
当 B 函数执行 ret 也就是返回指令时，就会去读取 x30 寄存器的地址，跳转过去，因此也就回到了 A函数的下一步。

```Objective-C
void *PC = __builtin_return_address(0); 
```
它的作用其实就是去读取 x30 中所存储的要返回到下一条指令的地址，也就是说可在 __sanitizer_cov_trace_pc_guard 获取到被插桩函数地址。

在 dlfcn.h 中有一个方法如下:
```Objective-C
typedef struct dl_info {
        const char      *dli_fname;     /* 所在文件 */
        void            *dli_fbase;     /* 文件地址 */
        const char      *dli_sname;     /* 符号名称 */
        void            *dli_saddr;     /* 函数起始地址 */
} Dl_info;

//这个函数能通过函数内部地址找到函数符号
int dladdr(const void *, Dl_info *);
```
由于 __sanitizer_cov_trace_pc_guard 在不同函数里调用，所以涉及多线程调用，考虑到这个方法调用频率高，使用锁会影响性能，这里使用苹果底层的原子队列 (底层实际上是个栈结构，利用队列结构 + 原子性来保证顺序 ) 来实现。

```Objective-C
#import <dlfcn.h>
#import <libkern/OSAtomic.h>
#include <sanitizer/coverage_interface.h>

//原子队列
static OSQueueHead symbolList = OS_ATOMIC_QUEUE_INIT;
//定义符号结构体
typedef struct{
    void * pc;
    void * next;
}SymbolNode;

#pragma mark - 静态插桩代码
void __sanitizer_cov_trace_pc_guard_init(uint32_t *start,
                                         uint32_t *stop) {
    static uint64_t N;  // Counter for the guards.
    if (start == stop || *start) return;  // Initialize only once.
    printf("INIT: %p %p\n", start, stop);
    for (uint32_t *x = start; x < stop; x++)
        *x = ++N;  // Guards should start from 1.
}

void __sanitizer_cov_trace_pc_guard(uint32_t *guard) {
    // 有load 方法时，guard 是 0，所以注释 if (!*guard) return;
    //if (!*guard) return;  // Duplicate the guard check.
    
    void *PC = __builtin_return_address(0);
    
    SymbolNode * node = malloc(sizeof(SymbolNode));
    *node = (SymbolNode){PC,NULL};
    
    //入队
    // offsetof 用在这里是为了入队添加下一个节点找到 前一个节点next指针的位置
    OSAtomicEnqueue(&symbolList, node, offsetof(SymbolNode, next));
}

/// 获取符号并写入 order 文件
+ (void)readSymbolAndWriteFile {
    
    NSMutableArray<NSString *> * symbolNames = [NSMutableArray array];
    while (YES) {
        //offsetof 就是针对某个结构体找到某个属性相对这个结构体的偏移量
        SymbolNode * node = OSAtomicDequeue(&symbolList, offsetof(SymbolNode, next));
        if (node == NULL) break;
        Dl_info info;
        dladdr(node->pc, &info);
        
        NSString * name = @(info.dli_sname);
        
        // 添加 _
        BOOL isObjc = [name hasPrefix:@"+["] || [name hasPrefix:@"-["];
        NSString * symbolName = isObjc ? name : [@"_" stringByAppendingString:name];
        
        //去重
        if (![symbolNames containsObject:symbolName]) {
            [symbolNames addObject:symbolName];
        }
    }

    //取反
    NSArray * symbolAry = [[symbolNames reverseObjectEnumerator] allObjects];
    NSLog(@"%@",symbolAry);
        
    //将结果写入到文件
    NSString * funcString = [symbolAry componentsJoinedByString:@"\n"];
    NSString * filePath = [NSTemporaryDirectory() stringByAppendingPathComponent:@"binary.order"];
    NSData * fileContents = [funcString dataUsingEncoding:NSUTF8StringEncoding];
    BOOL result = [[NSFileManager defaultManager] createFileAtPath:filePath contents:fileContents attributes:nil];
    if (result) {
        NSLog(@"%@",filePath);
    }else{
        NSLog(@"文件写入出错");
    }
}
```

cocoapod 工程引入的库，会产生多 个target，在主target添加的配置是不会生效的，我们需要针对需要的target做对应的设置。
对于直接手动导入到工程里的 sdk，不管是 静态库 .a 还是 动态库，会默认使用主工程的设置，也就是可以拿到符号的。
