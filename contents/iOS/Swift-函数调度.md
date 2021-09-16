---
title: Swift 函数调度
date: 2021-09-13 19:06:54
categories:
- iOS
tags:
- Swift
keywords:
- 函数调度
description:
images:
---

在 OC 中方法的调用是要经过 runtime 消息转发机制找到方法实现，然后调用方法实现，在 Swift 中函数的调度分为两种：静态调度、动态调度。
<!-- more -->
### 函数调度方式
#### 静态调度
静态调度也被称为直接调度（Direct Dispatch），函数在编译、链接完成后，函数的地址就已经确定了，函数的地址存放在代码段。在调用函数时，直接跳转到这个地址来执行，因此它的效率最快。

#### 动态调度
对于动态调度的方法，编译器不知道它的内存地址，所以无法对代码进行优化，可能导致一些性能问题的出现。动态调度可以为覆写父类中的方法提供支持，使得Swift也可以实现多态。
动态调度又可分为现两种：

##### 表调度
表调度是编译语言中最常用的方法之一，在编译时，为每个类构造一个 v-table，其中包含类中实现的函数指针，拥有继承关系的子类会在虚函数表内通过继承顺序去展示虚函数表指针。与静态调度相比，表调度需要两条额外的指令（读和跳转）来确定在运行时函数的地址。

对于值类型来说，并没有构造虚函数表指针，但对于属于结构体的 Protocol，Protocol 可以拥有属性和实现方法，为管理 Protocol 的函数调用，所以加入了 The Protocol Witness Table (PWT) 函数表用于协议类型方法的调度。

##### 消息调度
消息调度是最灵活也是最慢的调度技术，最快是在方法缓存表中找到方法实现，最慢的时候，运行时需要爬遍整个类层次结构，然后分析是否动态添加了方法实现，是否提供了其他对象处理，是否有最后的容错处理。Objective-C 在很大程度上依赖于消息调度，并且还通过 dynamic @objc 向 Swift 提供运行时功能。

### 类的函数调度方式
```
/// main.swift
class Person {
    func eat() {}
    static func eat2() {}
    class func eat3() {}
    @objc func eat4() {}
    dynamic func eat5() {}
    @objc dynamic func eat6() {}
}
extension Person {
    func ex_eat() { }
    static func ex_eat2() {}
    class func ex_eat3() {}
    @objc func ex_eat4() {}
    dynamic func ex_eat5() {}
    @objc dynamic func ex_eat6() {}
}

let p = Person()
p.eat()
Person.eat2()
Person.eat3()
p.eat4()
p.eat5()
p.eat6()

p.ex_eat()
Person.ex_eat2()
Person.ex_eat3()
p.ex_eat4()
p.ex_eat5()
p.ex_eat6()

shell命令： swiftc -emit-sil main.swift 生成SIL源文件，简单处理如下：

// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
  %9 = class_method %8 : $Person, #Person.eat : (Person) -> () -> (), $@convention(method) (@guaranteed Person) -> () // user: %10
  %12 = function_ref @$s4main6PersonC4eat2yyFZ : $@convention(method) (@thick Person.Type) -> () // user: %13
  %15 = function_ref @$s4main6PersonC4eat3yyFZ : $@convention(method) (@thick Person.Type) -> () // user: %16
  %18 = class_method %17 : $Person, #Person.eat4 : (Person) -> () -> (), $@convention(method) (@guaranteed Person) -> () // user: %19
  %21 = class_method %20 : $Person, #Person.eat5 : (Person) -> () -> (), $@convention(method) (@guaranteed Person) -> () // user: %22
  %24 = objc_method %23 : $Person, #Person.eat6!foreign : (Person) -> () -> (), $@convention(objc_method) (Person) -> () // user: %25
  %27 = function_ref @$s4main6PersonC6ex_eatyyF : $@convention(method) (@guaranteed Person) -> () // user: %28
  %30 = function_ref @$s4main6PersonC7ex_eat2yyFZ : $@convention(method) (@thick Person.Type) -> () // user: %31
  %33 = function_ref @$s4main6PersonC7ex_eat3yyFZ : $@convention(method) (@thick Person.Type) -> () // user: %34
  %36 = objc_method %35 : $Person, #Person.ex_eat4!foreign : (Person) -> () -> (), $@convention(objc_method) (Person) -> () // user: %37
  %39 = dynamic_function_ref @$s4main6PersonC7ex_eat5yyF : $@convention(method) (@guaranteed Person) -> () // user: %40
  %42 = objc_method %41 : $Person, #Person.ex_eat6!foreign : (Person) -> () -> (), $@convention(objc_method) (Person) -> () // user: %43
 
  %45 = struct $Int32 (%44 : $Builtin.Int32)      // user: %46
  return %45 : $Int32                             // id: %46
} // end sil function 'main'

sil_vtable Person {
  #Person.eat: (Person) -> () -> () : @$s4main6PersonC3eatyyF	// Person.eat()
  #Person.eat3: (Person.Type) -> () -> () : @$s4main6PersonC4eat3yyFZ	// static Person.eat3()
  #Person.eat4: (Person) -> () -> () : @$s4main6PersonC4eat4yyF	// Person.eat4()
  #Person.eat5: (Person) -> () -> () : @$s4main6PersonC4eat5yyF	// Person.eat5()
  #Person.init!allocator: (Person.Type) -> () -> Person : @$s4main6PersonCACycf// Person.__allocating_init()
  #Person.deinit!deallocator: @$s4main6PersonCfD	// Person.__deallocating_deinit
}

```
从SIL源文件我们可以看到有三种调用调度方式：class_method、function_ref、objc_method。

首先了解一下汇编指令:
```
adrp: 地址偏移
blr: 带返回的跳转指令，跳转到指令后边跟随寄存器中保存的地址。
mov: 将某一寄存器的值复制到另一寄存器（只能用于寄存器和寄存器或者寄存器与常量之间传值，不能用于内存地址）如 mov x1 , x0 : 将寄存器 x0 的值复制到寄存器 x1 中
ldr: 将内存中的值读取到寄存器中 ldr x0, [x1, x2] : 将内存 [x1 + x2] 处的值放入寄存器 x0 中
str: 将寄存器中的值写入到内存中 str x0, [x0, x8] : 将寄存器 x0 的值保存在内存 [x0 + x8] 处
bl: 跳转到某地址
```
打断点，汇编编译过程如下：
```
    // p.eat()  class_method
    -> 0x100005fcc <+56>:  ldr    x9, [x8, #0x488] 
    0x100005fd0 <+60>:  ldr    x10, [x9]
    0x100005fd4 <+64>:  ldr    x10, [x10, #0x50]
    0x100005fd8 <+68>:  mov    x20, x9
    0x100005fdc <+72>:  str    x8, [sp]
    0x100005fe0 <+76>:  blr    x10
    0x100005fe4 <+80>:  ldr    x20, [sp, #0x8]
    
    // Person.eat2() function_ref
    -> 0x100005fe8 <+84>:  bl     0x1000060e0               ; static SwiftTest.Person.eat2() -> () at main.swift:27
    0x100005fec <+88>:  ldr    x20, [sp, #0x8]
    
    // Person.eat3() function_ref
    -> 0x100005ff0 <+92>:  bl     0x1000060f4               ; static SwiftTest.Person.eat3() -> () at main.swift:28
    0x100005ff4 <+96>:  ldr    x8, [sp]
    
    // p.eat4() class_method
    -> 0x100005ff8 <+100>: ldr    x9, [x8, #0x488]
    0x100005ffc <+104>: ldr    x10, [x9]
    0x100006000 <+108>: ldr    x10, [x10, #0x60]
    0x100006004 <+112>: mov    x20, x9
    0x100006008 <+116>: blr    x10
    0x10000600c <+120>: ldr    x8, [sp]
    
    // p.eat5() class_method
    -> 0x100006010 <+124>: ldr    x9, [x8, #0x488]
    0x100006014 <+128>: ldr    x10, [x9]
    0x100006018 <+132>: ldr    x10, [x10, #0x68]
    0x10000601c <+136>: mov    x20, x9
    0x100006020 <+140>: blr    x10
    0x100006024 <+144>: ldr    x8, [sp]
    
    // p.eat6() objc_method
    -> 0x100006028 <+148>: ldr    x0, [x8, #0x488]
    0x10000602c <+152>: adrp   x9, 18
    0x100006030 <+156>: ldr    x9, [x9, #0xa40]
    0x100006034 <+160>: mov    x1, x9
    0x100006038 <+164>: bl     0x100011bd0               ; symbol stub for: objc_msgSend
    0x10000603c <+168>: ldr    x8, [sp]
    
    // p.ex_eat() function_ref
    -> 0x100006040 <+172>: ldr    x20, [x8, #0x488]
    0x100006044 <+176>: bl     0x1000062ac               ; SwiftTest.Person.ex_eat() -> () at main.swift:34
    0x100006048 <+180>: ldr    x20, [sp, #0x8]
    
    // Person.ex_eat2() function_ref
    -> 0x10000604c <+184>: bl     0x1000062c0               ; static SwiftTest.Person.ex_eat2() -> () at main.swift:35
    0x100006050 <+188>: ldr    x20, [sp, #0x8]
    
    // Person.ex_eat3() function_ref
    -> 0x100006054 <+192>: bl     0x1000062d4               ; static SwiftTest.Person.ex_eat3() -> () at main.swift:36
    0x100006058 <+196>: ldr    x8, [sp]
    
    // p.ex_eat4() objc_method
    -> 0x10000605c <+200>: ldr    x0, [x8, #0x488]
    0x100006060 <+204>: adrp   x9, 18
    0x100006064 <+208>: ldr    x1, [x9, #0xa48]
    0x100006068 <+212>: bl     0x100011bd0               ; symbol stub for: objc_msgSend
    0x10000606c <+216>: ldr    x8, [sp]
    
    // p.ex_eat5() function_ref
    -> 0x100006070 <+220>: ldr    x20, [x8, #0x488]
    0x100006074 <+224>: bl     0x100006338               ; SwiftTest.Person.ex_eat5() -> () at main.swift:38
    0x100006078 <+228>: ldr    x8, [sp]
    
    // p.ex_eat6() objc_method
    -> 0x10000607c <+232>: ldr    x0, [x8, #0x488]
    0x100006080 <+236>: adrp   x9, 18
    0x100006084 <+240>: ldr    x1, [x9, #0xa50]
    0x100006088 <+244>: bl     0x100011bd0               ; symbol stub for: objc_msgSend
```
可以看出：
- function_ref 静态调度
- class_method 表调度
- objc_method 消息调度

在类中，对于 static、class 修饰的函数采用的是静态调度，@objc dynamic 组合修饰的函数采用消息调度，其他的函数采用表调度。在类的 extension 中，objc 和 objc dynamic 修饰的函数采用的是消息调度，其他的函数采用静态调度。

#### sil_vtable
sil_vtable SIL源码实现如下：
```
// swift-main/docs/SIL.rst
decl ::= sil-vtable
sil-vtable ::= 'sil_vtable' identifier '{' sil-vtable-entry* '}'
 
sil-vtable-entry ::= sil-decl-ref ':' sil-linkage? sil-function-name

SIL represents dynamic dispatch for class methods using the `class_method`_,
`super_method`_, `objc_method`_, and `objc_super_method`_ instructions.

The potential destinations for `class_method`_ and `super_method`_ are
tracked in ``sil_vtable`` declarations for every class type. The declaration
contains a mapping from every method of the class (including those inherited
from its base class) to the SIL function that implements the method for that
class::

  class A {
    func foo()
    func bar()
    func bas()
  }

  sil @A_foo : $@convention(thin) (@owned A) -> ()
  sil @A_bar : $@convention(thin) (@owned A) -> ()
  sil @A_bas : $@convention(thin) (@owned A) -> ()

  sil_vtable A {
    #A.foo: @A_foo
    #A.bar: @A_bar
    #A.bas: @A_bas
  }

  class B : A {
    func bar()
  }

  sil @B_bar : $@convention(thin) (@owned B) -> ()

  sil_vtable B {
    #A.foo: @A_foo
    #A.bar: @B_bar
    #A.bas: @A_bas
  }

  class C : B {
    func bas()
  }

  sil @C_bas : $@convention(thin) (@owned C) -> ()

  sil_vtable C {
    #A.foo: @A_foo
    #A.bar: @B_bar
    #A.bas: @C_bas
  }
```
从源码注释中可得知，在类中 class_method、super_method、objc_method 和 objc_super_method 都采用动态调度。类的每个函数和继承的函数，都在 sil_vtable 中映射相对应的函数实现。swift 的 AST 记录了重载关系，vtable中的函数声明是指向最后的衍生类的函数。为了标明重载函数的所属，使用了原始类名作为前缀。

再看一下类的 v-table 的初始化流程
```
static void initClassVTable(ClassMetadata *self) {
  const auto *description = self->getDescription();
  auto *classWords = reinterpret_cast<void **>(self);

  if (description->hasVTable()) {
    auto *vtable = description->getVTableDescriptor();
    auto vtableOffset = vtable->getVTableOffset(description);
    auto descriptors = description->getMethodDescriptors();
    for (unsigned i = 0, e = vtable->VTableSize; i < e; ++i) {
      auto &methodDescription = descriptors[i];
      swift_ptrauth_init_code_or_data(
          &classWords[vtableOffset + i], methodDescription.Impl.get(),
          methodDescription.Flags.getExtraDiscriminator(),
          !methodDescription.Flags.isAsync());
    }
  }
  ...
}
```
可以看出 v-table 是一个数组结构，里面存放着函数指针数组，通过 vtableOffset + i 偏移获取函数。


### 结构体的函数调度方式
```
struct TestStruct {
    func test() {}
    static func test2() {}
}
extension TestStruct {
    func ex_test() { }
    static func ex_test2() {}
}

let s = TestStruct()
s.test()
TestStruct.test2()

s.ex_test()
TestStruct.ex_test2()

// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):

  %9 = function_ref @$s4main10TestStructV4testyyF : $@convention(method) (TestStruct) -> () // user: %10

  %12 = function_ref @$s4main10TestStructV5test2yyFZ : $@convention(method) (@thin TestStruct.Type) -> () // user: %13

  %15 = function_ref @$s4main10TestStructV7ex_testyyF : $@convention(method) (TestStruct) -> () // user: %16

  %18 = function_ref @$s4main10TestStructV8ex_test2yyFZ : $@convention(method) (@thin TestStruct.Type) -> () // user: %19

  %21 = struct $Int32 (%20 : $Builtin.Int32)      // user: %22
  return %21 : $Int32                             // id: %22
} // end sil function 'main'
```
由上可以知道，在结构体中的函数都是静态调度。

### 协议的函数调度方式
```
// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
 
  // function_ref RunProtocol.run()
  %9 = open_existential_addr immutable_access %3 : $*RunProtocol to $*@opened("5B499E10-11DF-11EC-A870-1E00231ED685") RunProtocol // users: %11, %11, %10
  %10 = witness_method $@opened("5B499E10-11DF-11EC-A870-1E00231ED685") RunProtocol, #RunProtocol.run : <Self where Self : RunProtocol> (Self) -> () -> (), %9 : $*@opened("5B499E10-11DF-11EC-A870-1E00231ED685") RunProtocol : $@convention(witness_method: RunProtocol) <τ_0_0 where τ_0_0 : RunProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %9; user: %11
  
  // function_ref RunProtocol.ex_run()
  %13 = function_ref @$s4main11RunProtocolPAAE6ex_runyyF : $@convention(method) <τ_0_0 where τ_0_0 : RunProtocol> (@in_guaranteed τ_0_0) -> () // user: %14
  %14 = apply %13<@opened("5B499FBE-11DF-11EC-A870-1E00231ED685") RunProtocol>(%12) : $@convention(method) <τ_0_0 where τ_0_0 : RunProtocol> (@in_guaranteed τ_0_0) -> () // type-defs: %12
  
  %16 = struct $Int32 (%15 : $Builtin.Int32)      // user: %17
  return %16 : $Int32                             // id: %17
} // end sil function 'main'
```
从上面的SIL源码看，对于 protocol 的扩展函数 ex_run() 调用采用的是静态调度，而 run() 采用的是 witness_method 的调度方式。

witness_method 采用的是 Protocol Witness Table(简称PWT)调度，和 v-table 一样, PWT 内存储的是方法数组，里面包含了函数实现的指针地址，调度函数时，通过获取对象的内存地址和函数的 offset 去查找的。

#### sil-witness-table
```
// swift-main/docs/SIL.rst

decl ::= sil-witness-table
sil-witness-table ::= 'sil_witness_table' sil-linkage?
 normal-protocol-conformance '{' sil-witness-entry* '}'
```
SIL 将通用类型动态分派所需的信息编码为 witness table，这些信息用于在生成二进制码时产生运行时分配表 (runtime dispatch table)。也可以用于对特定通用函数的 SIL 优化。每个明确的一致性声明都会产生 witness table。通用类型的所有实例共享一个通用 witness table，衍生类会继承基类的 witness table。


看下面一段代码，结果会输出什么呢？
```
protocol RunProtocol {
    func run()
}
extension RunProtocol {
    func run() {
        print("extension run")
    }
}
class ProtocolStructTest: RunProtocol { }

class ProtocolStructTestSuper: ProtocolStructTest {
    func run() {
        print("sadda")
    }
}

let structTestSuper: ProtocolStructTestSuper = ProtocolStructTestSuper()
let structTestSuper2: RunProtocol = ProtocolStructTestSuper()
structTestSuper.run()
structTestSuper2.run()

输出：
sadda
extension run
```

structTestSuper.run() 调用者是 ProtocolStructTestSuper 类型，采用 class_method 方式调度，所以打印：sadda。structTestSuper2.run() 调用
者是 RunProtocol 协议类型，采用 witness_method 方式调度，由于父类 ProtocolStructTest 接受 RunProtocol 协议并采用协议的默认实现方式，协议的默认实现函数存放在 RunProtocol 的 witness Table ，因此 ProtocolStructTest 的虚函数表中没有该函数，子类 ProtocolStructTestSuper 也无法继承到该函数。

```
sil_vtable ProtocolStructTest {
  #ProtocolStructTest.init!allocator: (ProtocolStructTest.Type) -> () -> ProtocolStructTest : @$s4main18ProtocolStructTestCACycfC	// ProtocolStructTest.__allocating_init()
  #ProtocolStructTest.deinit!deallocator: @$s4main18ProtocolStructTestCfD	// ProtocolStructTest.__deallocating_deinit
}

sil_vtable ProtocolStructTestSuper {
  #ProtocolStructTest.init!allocator: (ProtocolStructTest.Type) -> () -> ProtocolStructTest : @$s4main23ProtocolStructTestSuperCACycfC [override]	// ProtocolStructTestSuper.__allocating_init()
  #ProtocolStructTestSuper.run: (ProtocolStructTestSuper) -> () -> () : @$s4main23ProtocolStructTestSuperC3runyyF	// ProtocolStructTestSuper.run()
  #ProtocolStructTestSuper.deinit!deallocator: @$s4main23ProtocolStructTestSuperCfD	// ProtocolStructTestSuper.__deallocating_deinit
}

sil_witness_table hidden ProtocolStructTest: RunProtocol module main {
  method #RunProtocol.run: <Self where Self : RunProtocol> (Self) -> () -> () : @$s4main18ProtocolStructTestCAA03RunB0A2aDP3runyyFTW	// protocol witness for RunProtocol.run() in conformance ProtocolStructTest
}

```

子类 ProtocolStructTestSuper 中实现的 run() 函数并不是对 RunProtocol 协议方法实现，只是 ProtocolStructTestSuper 的成员函数而已，所以 witness_method 调度时，发现子类 ProtocolStructTestSuper 没有 sil_witness_table，到父类 ProtocolStructTest 时，从父类的 sil_witness_table 找到 RunProtocol.run() 函数实现地址调用，输出：extension run。

### final 关键字
用 final 关键字修饰，表示不允许对其修饰的内容进行继承或者重新操作。当用 final 修饰类时，就无法对类进行继承以及对函数的重写，那对于函数调度有什么样的影响呢？
```
final class Person {
    func eat() {}
    static func eat2() {}
    class func eat3() {}
    @objc func eat4() {}
    dynamic func eat5() {}
    @objc dynamic func eat6() {}
}
extension Person {
    func ex_eat() { }
    static func ex_eat2() {}
    class func ex_eat3() {}
    @objc func ex_eat4() {}
    dynamic func ex_eat5() {}
    @objc dynamic func ex_eat6() {}
    
}

// SIL 源码

// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
  
  // function_ref Person.eat()
  %9 = function_ref @$s4main6PersonC3eatyyF : $@convention(method) (@guaranteed Person) -> () // user: %10
  %10 = apply %9(%8) : $@convention(method) (@guaranteed Person) -> ()
  %11 = metatype $@thick Person.Type              // user: %13
  // function_ref static Person.eat2()
  %12 = function_ref @$s4main6PersonC4eat2yyFZ : $@convention(method) (@thick Person.Type) -> () // user: %13
  %13 = apply %12(%11) : $@convention(method) (@thick Person.Type) -> ()
  %14 = metatype $@thick Person.Type              // user: %16
  // function_ref static Person.eat3()
  %15 = function_ref @$s4main6PersonC4eat3yyFZ : $@convention(method) (@thick Person.Type) -> () // user: %16
  %16 = apply %15(%14) : $@convention(method) (@thick Person.Type) -> ()
  %17 = load %3 : $*Person                        // user: %19
  // function_ref Person.eat4()
  %18 = function_ref @$s4main6PersonC4eat4yyF : $@convention(method) (@guaranteed Person) -> () // user: %19
  %19 = apply %18(%17) : $@convention(method) (@guaranteed Person) -> ()
  %20 = load %3 : $*Person                        // user: %22
  // dynamic_function_ref Person.eat5()
  %21 = dynamic_function_ref @$s4main6PersonC4eat5yyF : $@convention(method) (@guaranteed Person) -> () // user: %22
  %22 = apply %21(%20) : $@convention(method) (@guaranteed Person) -> ()
  %23 = load %3 : $*Person                        // users: %24, %25
  %24 = objc_method %23 : $Person, #Person.eat6!foreign : (Person) -> () -> (), $@convention(objc_method) (Person) -> () // user: %25
  %25 = apply %24(%23) : $@convention(objc_method) (Person) -> ()
  %26 = load %3 : $*Person                        // user: %28
  // function_ref Person.ex_eat()
  %27 = function_ref @$s4main6PersonC6ex_eatyyF : $@convention(method) (@guaranteed Person) -> () // user: %28
  %28 = apply %27(%26) : $@convention(method) (@guaranteed Person) -> ()
  %29 = metatype $@thick Person.Type              // user: %31
  // function_ref static Person.ex_eat2()
  %30 = function_ref @$s4main6PersonC7ex_eat2yyFZ : $@convention(method) (@thick Person.Type) -> () // user: %31
  %31 = apply %30(%29) : $@convention(method) (@thick Person.Type) -> ()
  %32 = metatype $@thick Person.Type              // user: %34
  // function_ref static Person.ex_eat3()
  %33 = function_ref @$s4main6PersonC7ex_eat3yyFZ : $@convention(method) (@thick Person.Type) -> () // user: %34
  %34 = apply %33(%32) : $@convention(method) (@thick Person.Type) -> ()
  %35 = load %3 : $*Person                        // user: %37
  // function_ref Person.ex_eat4()
  %36 = function_ref @$s4main6PersonC7ex_eat4yyF : $@convention(method) (@guaranteed Person) -> () // user: %37
  %37 = apply %36(%35) : $@convention(method) (@guaranteed Person) -> ()
  %38 = load %3 : $*Person                        // user: %40
  // dynamic_function_ref Person.ex_eat5()
  %39 = dynamic_function_ref @$s4main6PersonC7ex_eat5yyF : $@convention(method) (@guaranteed Person) -> () // user: %40
  %40 = apply %39(%38) : $@convention(method) (@guaranteed Person) -> ()
  %41 = load %3 : $*Person                        // users: %42, %43
  %42 = objc_method %41 : $Person, #Person.ex_eat6!foreign : (Person) -> () -> (), $@convention(objc_method) (Person) -> () // user: %43
  %43 = apply %42(%41) : $@convention(objc_method) (Person) -> ()
  %44 = integer_literal $Builtin.Int32, 0         // user: %45
  %45 = struct $Int32 (%44 : $Builtin.Int32)      // user: %46
  return %45 : $Int32                             // id: %46
} // end sil function 'main'

```
可以看出，用 final 修饰后，除 @objc dynamic 修饰的函数还是采用 objc_method 调度，其他的函数都采用静态调度。虽然静态调度效率高，但也不能因此过度使用 final， 
对于不希望被继承和重写的类可以使用该关键词，提高函数的调用效率。