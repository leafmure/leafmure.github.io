---
title: iOS-事件分发处理-RunLoop
date: 2022-03-08 17:31:02
categories: iOS
tags: RunLoop
keywords: RunLoop,事件分发
description:
images:
---
### 一、初识 RunLoop
#### 1.1  线程
iOS开发是单进程，所以一般来说，一个应用程序就是一个进程，进程之间是相互独立的，每个进程都有独立的内存空间和资源。进程若需处理事件，则需要基本执行单元：线程。进程的所有事件都必须在线程中处理，因此进程在开辟时，会默认创建并开启一条线程，程序默认的事件处理和 UI 绘制都在这条线程处理，因此这条线程被称为主线程或者UI线程。
<!-- more -->
![image](../postImage/iOS-事件分发处理-RunLoop/psb-01.png)

#### 1.2  线程如何常驻处理事件
一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。主线程是程序是重要的事件处理单元，自然不能执行完任务后就退出，所以我们需要一个机制，让线程能随时处理事件但并不退出，通常的代码逻辑是这样的：
```swift
func loop() {
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```
上述代码实现通常被称为 Event Loop，使用 while 语句创建一个死循环，让 CPU 进入忙等待，当程序一接收事件便能立即处理，但是这种方式极其消耗 CPU 资源。

Event Loop 在很多系统和框架里都有实现，比如 Node.js 的事件处理，比如 Windows 程序的消息循环，再比如 OSX/iOS 里的 RunLoop。它们不是通过“死循环”这种“忙等待”方式让线程常驻处理事件，而是采用“闲等待”方式。当没有有事件需要处理时，线程会休眠以避免资源占用，在有消息到来时立刻被唤醒。

#### 1.3  RunLoop
RunLoop 实际上就是一个对象，是事件接收和分发机制的一个实现，让我们来看看它的结构：
```Objective-C
struct __CFRunLoop {
    __CFPort _wakeUpPort;   /// used for CFRunLoopWakeUp 内核向该端口发送消息可以唤醒runloop
    pthread_t _pthread;             /// RunLoop对应的线程
    CFMutableSetRef _commonModes;    /// 存储的是字符串，记录所有标记为common的mode
    CFMutableSetRef _commonModeItems; /// 存储所有commonMode的item(source、timer、observer)
    CFRunLoopModeRef _currentMode;   /// 当前运行的mode
    CFMutableSetRef _modes;          /// 存储的是CFRunLoopModeRef
    ......
```
可见，一个 RunLoop 对象，主要包含了一个线程，若干个Mode，若干个commonMode，还有一个当前运行的Mode。这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 “接受消息->等待->分发处理” 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。

CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。

NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

CFRunLoopRef 的代码是开源的，你可以在这里 http://opensource.apple.com/tarballs/CF/ 下载到整个 CoreFoundation 的源码来查看，Swift 开源后，苹果又维护了一个跨平台的 CoreFoundation 版本：https://github.com/apple/swift-corelibs-foundation/，这个版本的源码可能和现有 iOS 系统中的实现略不一样，但更容易编译，而且已经适配了 Linux/Windows。

#### 1.4 RunLoop 与 线程的关系
在Runloop的 官方文档 中，我们可以看到 Runloop 是一个死循环模型，线程在执行完任务后会进行休眠，有新的任务需要执行时就会被唤醒。如下图所示：

![image](../postImage/iOS-事件分发处理-RunLoop/psb-02.png)

线程提供事件处理的能力，RunLoop 提供线程持续处理事件的能力。RunLoop 在接收到 Input sources、Timer sources 后，根据 sources 类型转化相对应的事件交于线程处理。在这个过程中，RunLoop 体现了它接收事件和分发事件的能力。

在 RunLoop 结构中有定义它相对应线程对象的成员变量 pthread ，可以看出 RunLoop 是基于 pthread 来管理的。iOS 开发中能遇到两个线程对象: pthread_t 和 NSThread。过去苹果有份文档标明了 NSThread 只是 pthread_t 的封装，但那份文档已经失效了，现在它们也有可能都是直接包装自最底层的 mach thread。

苹果并没有提供这两个对象相互转换的接口，但不管怎么样，可以肯定的是 pthread_t 和 NSThread 是一一对应的。比如，你可以通过 pthread_main_thread_np() 或 [NSThread mainThread] 来获取主线程；也可以通过 pthread_self() 或 [NSThread currentThread] 来获取当前线程。

苹果不允许直接创建 RunLoop，它只提供了两个自动获取的函数：CFRunLoopGetMain() 和 CFRunLoopGetCurrent()。 这两个函数内部的逻辑大概是下面这样:
```Objective-C
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}

/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
     /// 访问 loopsDic 时的锁
    OSSpinLockLock(&loopsLock);
    if (!loopsDic) {
        // 第一次进入时，初始化全局字典，key 是 pthread_t， value 是 CFRunLoopRef，并先为主线程创建一个 RunLoop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }

    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
```
获取 CFRunLoopRef 的过程比较简单

1、首先判断全局RunLoop和线程的映射字典有没有创建，没有的话便为主线程创建一个 RunLoop 并初始化全局字典。

2、全局字典存在的情况下，会直接访问字典取出已创建的 RunLoop 对象。若在字典中没有读取到RunLoop 对象，便创建一个 RunLoop 对象，将映射关系添加到全局字典中。

3、调用 _CFSetTSD 函数注册销毁监听回调，用于线程销毁时，同时销毁关联的 RunLoop对象。

总结：
- RunLoop和线程的一一对应的，对应的方式是以key-value的方式保存在一个全局字典中
- 主线程的RunLoop会在初始化全局字典时创建
- 子线程的RunLoop会在第一次获取的时候创建，如果不获取的话就一直不会被创建
- RunLoop会在线程销毁时销毁

### 二、RunLoop的运行
#### 2.1 RunLoop 相关作用类
在 CoreFoundation 里面关于 RunLoop 有5个类:
- CFRunLoopRef
- CFRunLoopModeRef
- CFRunLoopSourceRef
- CFRunLoopTimerRef
- CFRunLoopObserverRef

CFRunLoopModeRef 类没有对外暴露，只是通过 CFRunLoopRef 的接口进行了封装。他们的关系如下:
![image](../postImage/iOS-事件分发处理-RunLoop/psb-03.png)

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

Source/Timer/Observer 统称为 Mode item，一个 item 可以被同时加入多个 Mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 RunLoop 会直接退出，不进入循环。

#### 2.2 CFRunLoopSourceRef
CFRunLoopSourceRef 是事件产生的地方，Source有两个版本：Source0 和 Source1。
- Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。
- Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息，这种 Source 能主动唤醒 RunLoop 的线程。

#### 2.3 CFRunLoopTimerRef
CFRunLoopTimerRef 是基于时间的触发器，它和 NSTimer 是 toll-free bridged 的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到 RunLoop 时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒以执行那个回调。

#### 2.4 CFRunLoopObserverRef
CFRunLoopObserverRef 是观察者，每个 Observer 都包含了一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。可以观测的时间点有以下几个：
```Objective-C
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```

#### 2.5 RunLoop Mode
程序在运行的过程中，会处理基于时间的、系统的、用户的事件，这些事件在程序运行期间有着不同的优先级，为了满足应用层依据优先级对这些事件的管理，系统采用 RunLoopMode 对这些事件分组，然后交由RunLoop去管理，苹果文档中提到的 Mode 有五个，分别是：
- NSDefaultRunLoopMode /// 默认模式
- NSConnectionReplyMode /// NSConnection 对象监测回应
- NSModalPanelRunLoopMode /// 标识用于模态面板的事件
- NSEventTrackingRunLoopMode /// 用于UI交互事件追踪
- NSRunLoopCommonModes /// 并非常规模式，而是一组可配置的常用模式
iOS 中公开暴露出来的只有 NSDefaultRunLoopMode 和 NSRunLoopCommonModes。 NSRunLoopCommonModes 实际上是一个 Mode 的集合，默认包括 NSDefaultRunLoopMode 和 NSEventTrackingRunLoopMode。并不是说 Runloop 会运行在 kCFRunLoopCommonModes 这种模式下，而是相当于分别注册了 NSDefaultRunLoopMode 和 UITrackingRunLoopMode。当然你也可以通过调用CFRunLoopAddCommonMode() 方法将自定义 Mode 放到 kCFRunLoopCommonModes 组合。

#### 2.6 RunLoop 的运行流程
![image](../postImage/iOS-事件分发处理-RunLoop/psb-04.png)

### 三、RunLoop的实现
#### 3.1 RunLoop 运行函数实现
在Core Foundation中我们可以通过以下2个API来让RunLoop运行：
```Objective-C
/// 默认 kCFRunLoopDefaultMode 运行当前线程的RunLoop
void CFRunLoopRun(void) {  
    int32_t result;
    do {
        //默认在kCFRunLoopDefaultMode下运行runloop
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

/// 在指定mode下运行当前线程的RunLoop
SInt32 CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {  
    CHECK_FOR_FORK();
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
```
两种启动运行方式都共同的调用了 CFRunLoopRunSpecific 函数，那么 CFRunLoopRunSpecific 函数里干了什么呢？
```Objective-C
/// RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
    
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
    /// 如果mode里没有source/timer/observer, 直接返回。
    if (__CFRunLoopModeIsEmpty(currentMode)) return;
    
    /// 1. 通知 Observers: RunLoop 即将进入 loop。
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
    
    /// 内部函数，进入loop
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
        
        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {
 
            /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
            /// 4. RunLoop 触发 Source0 (非port) 回调。
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }
            
            /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
            if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }
            
            /// 7. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            /// • 一个基于 port 的Source 的事件。
            /// • 一个 Timer 到时间了
            /// • RunLoop 自身的超时时间到了
            /// • 被其他什么调用者手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }
 
            /// 8. 通知 Observers: RunLoop 的线程刚刚被唤醒了。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
            
            /// 收到消息，处理消息。
            handle_msg:
 
            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
 
            /// 9.2 如果有dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
 
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
            
            /// 执行加入到Loop的block
            __CFRunLoopDoBlocks(runloop, currentMode);
            
 
            if (sourceHandledThisLoop && stopAfterHandle) {
                /// 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout) {
                /// 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(runloop)) {
                /// 被外部调用者强制停止了
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                /// source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }
            
            /// 如果没超时，mode里没空，loop也没被停止，那继续loop。
        } while (retVal == 0);
    }
    
    /// 10. 通知 Observers: RunLoop 即将退出。
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}
```
CFRunLoopRunSpecific 具体函数实现在官方文档中可查阅，它的内部逻辑如下：
- 首先判断 RunLoop 对象是否存在，否则返回 kCFRunLoopRunFinished 结束运行。
- 根据传入的 Mode name 找到本次运行的 mode，若未找到该 Mode 或者 Mode 中未注册任何事件，则返回 kCFRunLoopRunFinished 结束运行。
- 上述步骤正常的话，通知 observer RunLoop 即将运行，然后调用 __CFRunLoopRun 函数开启 RunLoop 运行。
- 在 CFRunLoopRun 函数回调中，处理 Timer、Source0、Source1 等事件，若无事件则进入休眠。
- 直到接收到 kCFRunLoopExit 信号或者超时，通知 observer RunLoop 退出运行。

#### 3.2 添加 RunLoop Mode
```Objective-C
/// 向当前RunLoop的common modes中添加一个mode。
CFRunLoopAddCommonMode(CFRunLoopRef rl, CFStringRef mode)

/// 返回当前运行的mode的name
CFStringRef CFRunLoopCopyCurrentMode(CFRunLoopRef rl)

/// 返回当前RunLoop的所有mode
CFArrayRef CFRunLoopCopyAllModes(CFRunLoopRef rl)
```
我们没有办法直接创建一个CFRunLoopMode对象，但是我们可以调用 CFRunLoopAddCommonMode 传入 Mode 字符串名，函数内部会创建 CFRunLoopMode 对象并添加到 RunLoop 中。但是得注意以下几点：
- modeName不能重复，modeName是mode的唯一标识符
- RunLoop的_commonModes数组存放所有被标记为common的mode的名称
- 添加commonMode会把commonModeItems数组中的所有source同步到新添加的mode中
- CFRunLoopMode对象在CFRunLoopAddItemsToCommonMode函数中调用CFRunLoopFindMode时被创建
相关函数实现如下：
```Objective-C
CFRunLoopAddCommonMode
void CFRunLoopAddCommonMode(CFRunLoopRef rl, CFStringRef modeName) {
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return;
    __CFRunLoopLock(rl);
    //看rl中是否已经有这个mode，如果有就什么都不做
    if (!CFSetContainsValue(rl->_commonModes, modeName)) {
        CFSetRef set = rl->_commonModeItems ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModeItems) : NULL;
        //把modeName添加到RunLoop的_commonModes中
        CFSetAddValue(rl->_commonModes, modeName);
        if (NULL != set) {
            CFTypeRef context[2] = {rl, modeName};
            /* add all common-modes items to new mode */
            //这里调用CFRunLoopAddSource/CFRunLoopAddObserver/CFRunLoopAddTimer的时候会调用
            //__CFRunLoopFindMode(rl, modeName, true)，CFRunLoopMode对象在这个时候被创建
            CFSetApplyFunction(set, (__CFRunLoopAddItemsToCommonMode), (void *)context);
            CFRelease(set);
        }
    } else {
    }
    __CFRunLoopUnlock(rl);
}
```
#### 3.3 添加Run Loop Source（ModeItem）
我们可以通过以下接口添加/移除各种事件:
```Objective-C
void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef mode)
void CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef mode)
void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef mode)
void CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef * mode)
void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode)
void CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode)
```
CFRunLoopAddSource 的代码如下：
```Objective-C
//添加source事件
void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName) {    /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rls)) return;
    Boolean doVer0Callout = false;
    __CFRunLoopLock(rl);
    //如果是kCFRunLoopCommonModes
    if (modeName == kCFRunLoopCommonModes) {
        //如果runloop的_commonModes存在，则copy一个新的复制给set
        CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
       //如果runl _commonModeItems为空
        if (NULL == rl->_commonModeItems) {
            //先初始化
            rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
        }
        //把传入的CFRunLoopSourceRef加入_commonModeItems
        CFSetAddValue(rl->_commonModeItems, rls);
        //如果刚才set copy到的数组里有数据
        if (NULL != set) {
            CFTypeRef context[2] = {rl, rls};
            /* add new item to all common-modes */
            //则把set里的所有mode都执行一遍__CFRunLoopAddItemToCommonModes函数
            CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
            CFRelease(set);
        }
        //以上分支的逻辑就是，如果你往kCFRunLoopCommonModes里面添加一个source，那么所有_commonModes里的mode都会添加这个source
    } else {
        //根据modeName查找mode
        CFRunLoopModeRef rlm = __CFRunLoopFindMode(rl, modeName, true);
        //如果_sources0不存在，则初始化_sources0，_sources0和_portToV1SourceMap
        if (NULL != rlm && NULL == rlm->_sources0) {
            rlm->_sources0 = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
            rlm->_sources1 = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
            rlm->_portToV1SourceMap = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, NULL);
        }
        //如果_sources0和_sources1中都不包含传入的source
        if (NULL != rlm && !CFSetContainsValue(rlm->_sources0, rls) && !CFSetContainsValue(rlm->_sources1, rls)) {
            //如果version是0，则加到_sources0
            if (0 == rls->_context.version0.version) {
                CFSetAddValue(rlm->_sources0, rls);
                //如果version是1，则加到_sources1
            } else if (1 == rls->_context.version0.version) {
                CFSetAddValue(rlm->_sources1, rls);
                __CFPort src_port = rls->_context.version1.getPort(rls->_context.version1.info);
                if (CFPORT_NULL != src_port) {
                    //此处只有在加到source1的时候才会把souce和一个mach_port_t对应起来
                    //可以理解为，source1可以通过内核向其端口发送消息来主动唤醒runloop
                    CFDictionarySetValue(rlm->_portToV1SourceMap, (const void *)(uintptr_t)src_port, rls);
                    __CFPortSetInsert(src_port, rlm->_portSet);
                }
            }
            __CFRunLoopSourceLock(rls);
            //把runloop加入到source的_runLoops中
            if (NULL == rls->_runLoops) {
                rls->_runLoops = CFBagCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeBagCallBacks); // sources retain run loops!
            }
            CFBagAddValue(rls->_runLoops, rl);
            __CFRunLoopSourceUnlock(rls);
            if (0 == rls->_context.version0.version) {
                if (NULL != rls->_context.version0.schedule) {
                    doVer0Callout = true;
                }
            }
        }
        if (NULL != rlm) {
            __CFRunLoopModeUnlock(rlm);
        }
    }
    __CFRunLoopUnlock(rl);
    if (doVer0Callout) {
        // although it looses some protection for the source, we have no choice but
        // to do this after unlocking the run loop and mode locks, to avoid deadlocks
        // where the source wants to take a lock which is already held in another
        // thread which is itself waiting for a run loop/mode lock
        rls->_context.version0.schedule(rls->_context.version0.info, rl, modeName); /* CALLOUT */
    }
}
```
CFRunLoopAddSource 函数内部工作内容如下：
- 如果 modeName 传入 kCFRunLoopCommonModes，则该source 会被保存到 RunLoop 的 _commonModeItems 中，同时也会添加到 commonMode 下的所有 Mode 中。
- 如果modeName传入的不是kCFRunLoopCommonModes，则会先查找该Mode，如果没有，会创建一个新的 Mode 对象，将 Source 添加到该 Mode 保存。
同一个source在一个mode中只能被添加一次
```Objective-C
CFRunLoopRemoveSource
//移除source
void CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName) { /* DOES CALLOUT */
    CHECK_FOR_FORK();
    Boolean doVer0Callout = false, doRLSRelease = false;
    __CFRunLoopLock(rl);
    //如果是kCFRunLoopCommonModes，则从_commonModes的所有mode中移除该source
    if (modeName == kCFRunLoopCommonModes) {
        if (NULL != rl->_commonModeItems && CFSetContainsValue(rl->_commonModeItems, rls)) {
            CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
            CFSetRemoveValue(rl->_commonModeItems, rls);
            if (NULL != set) {
                CFTypeRef context[2] = {rl, rls};
                /* remove new item from all common-modes */
                CFSetApplyFunction(set, (__CFRunLoopRemoveItemFromCommonModes), (void *)context);
                CFRelease(set);
            }
        } else {
        }
    } else {
        //根据modeName查找mode，如果不存在，返回NULL
        CFRunLoopModeRef rlm = __CFRunLoopFindMode(rl, modeName, false);
        if (NULL != rlm && ((NULL != rlm->_sources0 && CFSetContainsValue(rlm->_sources0, rls)) || (NULL != rlm->_sources1 && CFSetContainsValue(rlm->_sources1, rls)))) {
            CFRetain(rls);
            //根据source版本做对应的remove操作
            if (1 == rls->_context.version0.version) {
                __CFPort src_port = rls->_context.version1.getPort(rls->_context.version1.info);
                if (CFPORT_NULL != src_port) {
                    CFDictionaryRemoveValue(rlm->_portToV1SourceMap, (const void *)(uintptr_t)src_port);
                    __CFPortSetRemove(src_port, rlm->_portSet);
                }
            }
            CFSetRemoveValue(rlm->_sources0, rls);
            CFSetRemoveValue(rlm->_sources1, rls);
            __CFRunLoopSourceLock(rls);
            if (NULL != rls->_runLoops) {
                CFBagRemoveValue(rls->_runLoops, rl);
            }
            __CFRunLoopSourceUnlock(rls);
            if (0 == rls->_context.version0.version) {
                if (NULL != rls->_context.version0.cancel) {
                    doVer0Callout = true;
                }
            }
            doRLSRelease = true;
        }
        if (NULL != rlm) {
            __CFRunLoopModeUnlock(rlm);
        }
    }
    __CFRunLoopUnlock(rl);
    if (doVer0Callout) {
        // although it looses some protection for the source, we have no choice but
        // to do this after unlocking the run loop and mode locks, to avoid deadlocks
        // where the source wants to take a lock which is already held in another
        // thread which is itself waiting for a run loop/mode lock
        rls->_context.version0.cancel(rls->_context.version0.info, rl, modeName);   /* CALLOUT */
    }
    if (doRLSRelease) CFRelease(rls);
}
```
添加observer和timer的内部逻辑和添加source大体类似。

区别在于observer和timer只能被添加到一个RunLoop的一个或者多个mode中，比如一个timer被添加到主线程的RunLoop中，则不能再把该timer添加到子线程的RunLoop，而source没有这个限制，不管是哪个RunLoop，只要mode中没有，就可以添加。

这一点从实现结构上就能体现，CFRunLoopSource结构体中有保存RunLoop对象的数组，而CFRunLoopObserver和CFRunLoopTimer只有单个RunLoop对象。
```Objective-C
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;
    CFMutableBagRef _runLoops;
    union {
        CFRunLoopSourceContext version0;          
        CFRunLoopSourceContext1 version1;    
    } _context;
}

struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;
    CFIndex _order;  
    CFRunLoopObserverCallBack _callout; 
    CFRunLoopObserverContext _context; 
};

struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop; 
    CFMutableSetRef _rlModes;   
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;   
    CFTimeInterval _tolerance;    
    uint64_t _fireTSR; 
    CFIndex _order;      
    CFRunLoopTimerCallBack _callout;    
    CFRunLoopTimerContext _context; 
};
```

#### 3.4 RunLoop 的底层实现
RunLoop 的核心是基于 mach port 的，其进入休眠时调用的函数是 mach_msg()。为了解释这个逻辑，下面稍微介绍一下 OSX/iOS 的系统架构。
![image](../postImage/iOS-事件分发处理-RunLoop/psb-05.png)

苹果官方将整个系统大致划分为上述4个层次： 应用层包括用户能接触到的图形应用，例如 Spotlight、Aqua、SpringBoard 等。 应用框架层即开发人员接触到的 Cocoa 等框架。 核心框架层包括各种核心框架、OpenGL 等内容。 Darwin 即操作系统的核心，包括系统内核、驱动、Shell 等内容，这一层是开源的，其所有源码都可以在 opensource.apple.com 里找到。

我们再深入看一下 Darwin 这个核心的架构：
![image](../postImage/iOS-事件分发处理-RunLoop/psb-06.png)

其中，在硬件层上面的三个组成部分：Mach、BSD、IOKit (还包括一些上面没标注的内容)，共同组成了 XNU 内核。 XNU 内核的内环被称作 Mach，其作为一个微内核，仅提供了诸如处理器调度、IPC (进程间通信)等非常少量的基础服务。 BSD 层可以看作围绕 Mach 层的一个外环，其提供了诸如进程管理、文件系统和网络等功能。 IOKit 层是为设备驱动提供了一个面向对象(C++)的一个框架。

Mach 本身提供的 API 非常有限，而且苹果也不鼓励使用 Mach 的 API，但是这些API非常基础，如果没有这些API的话，其他任何工作都无法实施。在 Mach 中，所有的东西都是通过自己的对象实现的，进程、线程和虚拟内存都被称为”对象”。和其他架构不同， Mach 的对象间不能直接调用，只能通过消息传递的方式实现对象间的通信。“消息”是 Mach 中最基础的概念，消息在两个端口 (port) 之间传递，这就是 Mach 的 IPC (进程间通信) 的核心。

Mach 的消息定义在 <mach/message.h> 文件：
```Objective-C
typedef struct {
  mach_msg_header_t header;
  mach_msg_body_t body;
} mach_msg_base_t;
 
typedef struct {
  mach_msg_bits_t msgh_bits;
  mach_msg_size_t msgh_size;
  mach_port_t msgh_remote_port;
  mach_port_t msgh_local_port;
  mach_port_name_t msgh_voucher_port;
  mach_msg_id_t msgh_id;
} mach_msg_header_t;

mach_msg_return_t mach_msg(
   mach_msg_header_t *msg,
   mach_msg_option_t option,
   mach_msg_size_t send_size,
   mach_msg_size_t rcv_size,
   mach_port_name_t rcv_name,
   mach_msg_timeout_t timeout,
   mach_port_name_t notify);
```
一条 Mach 消息实际上就是一个二进制数据包 (BLOB)，其头部定义了当前端口 msgh_remote_port 和目标端口 msgh_remote_port， 发送和接受消息是通过同一个 API 进行的，其 option 标记了消息传递的方向。

为了实现消息的发送和接收，mach_msg() 函数实际上是调用了一个 Mach 陷阱 (trap)，即函数mach_msg_trap()，陷阱这个概念在 Mach 中等同于系统调用。当你在用户态调用 mach_msg_trap() 时会触发陷阱机制，切换到内核态；内核态中内核实现的 mach_msg() 函数会完成实际的工作，如下图：
![image](../postImage/iOS-事件分发处理-RunLoop/psb-07.png)

RunLoop 的核心就是一个 mach_msg() ，RunLoop 调用这个函数去接收消息，如果没有别人发送 port 消息过来，内核会将线程置于等待状态。例如你在模拟器里跑起一个 iOS 的 App，然后在 App 静止时点击暂停，你会看到主线程调用栈是停留在 mach_msg_trap() 这个地方。

### 四、RunLoop的应用
#### 4.1 根据 RunLoop 的状态 Observer ，优化性能
复杂 Cell 的高度计算比较耗时，因此我们需要对高度缓存，避免重复计算开销。计算高度并缓存的时机应在用户无感知的情况下，所以我们得满足以下条件：

RunLoop 处于“空闲”状态
当 RunLoop 迭代处理完成了所有事件，马上要休眠时 RunLoop 提供了 CFRunLoopObserverCreateWithHandler 函数，让我们可以通过该函数监听 RunLoop 当前的运行状态，代码示例如下：
```Objective-C
CFRunLoopRef runLoop = CFRunLoopGetCurrent();
CFStringRef runLoopMode = kCFRunLoopDefaultMode;
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler
(kCFAllocatorDefault, kCFRunLoopBeforeWaiting, true, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity _) {
    // TODO here
});
CFRunLoopAddObserver(runLoop, observer, runLoopMode);
/// 注意：需在合适的机会调用 CFRunLoopRemoveObserver 移除监听
```

#### 4.2 AutoreleasePool
App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是_wrapRunLoopWithAutoreleasePoolHandler()。
- 第一个 Observer 监视的事件是 Entry (即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。
- 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

#### 4.3 事件响应
苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的 App 进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

#### 4.4 界面更新
当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数： _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

#### 4.5 NSTimer 定时器
NSTimer 其实就是 CFRunLoopTimerRef，他们之间是 toll-free bridged 的。一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。 如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。就比如等公交，如果 10:10 时我忙着玩手机错过了那个点的公交，那我只能等 10:20 这一趟了。

CADisplayLink 是一个和屏幕刷新率一致的定时器（但实际实现原理更复杂，和 NSTimer 并不一样，其内部实际是操作了一个 Source）。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 NSTimer 相似），造成界面卡顿的感觉。在快速滑动TableView时，即使一帧的卡顿也会让用户有所察觉。Facebook 开源的 AsyncDisplayLink 就是为了解决界面卡顿的问题，其内部也用到了 RunLoop。

#### 4.6 PerformSelecter
当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。 当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

#### 4.7 iOS 中的网络请求 NSURLConnection
通常使用 NSURLConnection 时，你会传入一个 Delegate，当调用了 [connection start] 后，这个 Delegate 就会不停收到事件回调。实际上，start 这个函数的内部会会获取 CurrentRunLoop，然后在其中的 DefaultMode 添加了4个 Source0 (即需要手动触发的Source)。CFMultiplexerSource 是负责各种 Delegate 回调的，CFHTTPCookieStorage 是处理各种 Cookie 的。

当开始网络传输时，我们可以看到 NSURLConnection 创建了两个新线程：com.apple.NSURLConnectionLoader 和 com.apple.CFSocket.private。其中 CFSocket 线程是处理底层 socket 连接的。NSURLConnectionLoader 这个线程内部会使用 RunLoop 来接收底层 socket 的事件，并通过之前添加的 Source0 通知到上层的 Delegate。

NSURLConnectionLoader 中的 RunLoop 通过一些基于 mach port 的 Source 接收来自底层 CFSocket 的通知。当收到通知后，其会在合适的时机向 CFMultiplexerSource 等 Source0 发送通知，同时唤醒 Delegate 线程的 RunLoop 来让其处理这些通知。CFMultiplexerSource 会在 Delegate 线程的 RunLoop 对 Delegate 执行实际的回调。

#### 4.8 RunLoop 在 AFNetworking 的应用
AFNetworking 是第三方框架，基于 NSURLConnection 系统网络请求框架封装的。AFNetworking 希望能在后台线程接收 Delegate 回调。为此 AFNetworking 单独创建了一个线程，并在这个线程中启动了一个 RunLoop：
```Objective-C
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}

+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
```
RunLoop 启动前内部必须要有至少一个 Timer/Observer/Source，所以 AFNetworking 在 [runLoop run] 之前先创建了一个新的 NSMachPort 添加进去了。通常情况下，调用者需要持有这个 NSMachPort (mach_port) 并在外部线程通过这个 port 发送消息到 loop 内；但此处添加 port 只是为了让 RunLoop 不至于退出，并没有用于实际的发送消息。
```Objective-C
- (void)start {
    [self.lock lock];
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;
        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}
```
当需要这个后台线程执行任务时，AFNetworking 通过调用 [NSObject performSelector:onThread:..] 将这个任务扔到了后台线程的 RunLoop 中。

整理出处:
[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)
