---
title: Swift 指针
date: 2021-09-13 19:59:34
categories:
- iOS
tags:
- Swift
keywords:
- 指针
description:
images:
---
在 OC 中指针的使用采用 C 语法方式，通过 *、& 操作指针，在 Swift 需要初始化相应的的指针类，其中分为两类：
- typed pointer 指向的内存分配了数据类型
- raw pointer 指向的内存未分配数据类型
<!-- more -->

swift | OC | 说明 |
---|---|---|
unsafePointer<T> | const T * | 指针及所指向的内容都不可变 |
unsafeMutablePointer | T * | 指针及其所指向的内存内容均可变 |
unsafeRawPointer|const void *|指针指向未知类型|
unsafeMutableRawPointer|void *|指针指向未知类型|

### typed pointer 
#### unsafePointer<T> / UnsafeMutablePointer <T>
unsafePointer<T> 对于 T 可以指定指针指向的类型，没有直接的初始化方法，只能通过 withUnsafePointer 转换获得。
```swift
var a = 7
let pointer: UnsafePointer<Int> = withUnsafePointer(to: &a) { $0}

```

对于 UnsafeMutablePointer<T> 需要我们手动进行内存的申请和释放，因此它提供了初始化方法和释放方法。
```swift
// 申请 1 个 Int 类型的内存空间
let pointer = UnsafeMutablePointer<Int>.allocate(capacity: 1)

// 初始化内存，初始值为：7
pointer.initialize(to: 7)

// 初始化后，就可以通过 pointee 来操作指针指向的内容
pointer.pointee = 8

// 当该指针不需要使用了，可以用 deinitialize 销毁指针指向的对象
pointer.deinitialize()

// 指针指向的内容被清空后，需要 deallocate 回收内存空间，回收 1 个 Int 类型的内存空间
pointer.deallocate(capacity: 1)
```
对于 Int 这样分配在常量段上的对象，deinitialize 并不是必要的，但是对于像类的对象或者结构体实例来说，如果不保证初始化和摧毁配对的话，是会出现内存泄露的。所以没有特殊考虑的话，不论内存中到底是什么，保证 initialize: 和 deinitialize 配对会是一个好习惯。

对于一个 UnsafePointer<T> 类型，我们可以通过 pointee 属性对其进行取值，如果这个指针是可变的 UnsafeMutablePointer<T> 类型，我们还可以通过 pointee 对它进行赋值。
```swift
func update(p: UnsafeMutablePointer<Int>) {
    ptr.pointee = 12
}

var a = 10
update(&a)
```
这里和 C 的指针使用类似，我们通过在变量名前面加上 & 符号就可以将指向这个变量的指针传递到接受指针作为参数的方法中去。与这种做法类似的是使用 Swift 的 inout 关键字。我们在将变量传入 inout 参数的函数时，同样也使用 & 符号表示地址。不过区别是在函数体内部我们不需要处理指针类型，而是可以对参数直接进行操作。

虽然 & 在参数传递时表示的意义和 C 中一样，是某个“变量的地址”，但是在 Swift 中我们没有办法直接通过这个符号获取一个 UnsafePointer 的实例。

#### UnsafeBufferPointer / UnsafeMutableBufferPointer
UnsafeBufferPointer 和 UnsafeMutableBufferPointer 用来表达像是数组或者字典这样的集合类型，其元素指针为声明类型指针，带 Mutable 和不带 Mutable 字样的区别遵循 Cocoa 可变和不可变原则。

```swift
var array = [1, 2, 3, 4, 5]
var arrayPtr: UnsafeMutableBufferPointer? = UnsafeMutableBufferPointer<Int>(start: &array, count: array.count)
// baseAddress 是第一个元素的指针，类型为 UnsafeMutablePointer<Int>
if let basePtr = arrayPtr?.baseAddress {
    print(basePtr.pointee)  // 1
    basePtr.pointee = 10
    print(basePtr.pointee) // 10
    
    //下一个元素
    let nextPtr = basePtr.successor()
    print(nextPtr.pointee) // 2
}
arrayPtr = nil
```
不过上述代码会有警告，提示结果可能是“悬空指针”，指针所指内存被释放后，这个指针就被悬空了。由于编译器无法确定该指针悬空后是否释放了，故此有该提示。为了消除该提示，可以采用 withUnsafeMutableBufferPointer 转换方法：
```swift
var array = [1, 2, 3, 4, 5]
array.withUnsafeMutableBufferPointer { p in
    if let basePtr = p.baseAddress {
        print(basePtr.pointee)  // 1
        basePtr.pointee = 10
        print(basePtr.pointee) // 10
        
        //下一个元素
        let nextPtr = basePtr.successor()
        print(nextPtr.pointee) // 2
    }
}
```

### raw pointer
#### UnsafeRawPointer / UnsafeMutableRawPointer
raw pointer 指向的内存地址，不注明数据类型，只对内存空间进行管理，通过 storeBytes 存值，load 取值。
```swift
//定义一个未知类型的指针：本质是分配32字节大小的空间，指定对齐方式是8字节对齐
let p = UnsafeMutableRawPointer.allocate(byteCount: 32, alignment: 8)

//存值
for i in 0..<4 {
    //指定当前移动的步数 
    p.advanced(by: i * 8).storeBytes(of: i + 1, as: Int.self)
}

//取值
for i in 0..<4 {
    //p是当前内存的首地址，通过内存平移来获取值
    let value = p.load(fromByteOffset: i * 8, as: Int.self)
    print("index: \(i), value: \(value)")
}

//使用完需要手动释放
p.deallocate()
```

#### UnsafeRawPointer / UnsafeMutableRawPointer
UnsafeRawPointer 和 UnsafeMutableRawPointer 用来表达像是数组或者字典这样的集合类型，其元素指针为未声明类型指针，带 Mutable 和不带 Mutable 字样的区别遵循 Cocoa 可变和不可变原则。
```swift
var array = [1,2,3]
let p = UnsafeMutableRawBufferPointer.allocate(byteCount: 32, alignment: 4)
p.initializeMemory(as: Int.self, from: array)
let v = p.baseAddress?.load(as: Int.self)
print(v)
p.deallocate()
```

### 指针转换
#### withUnsafePointer / withUnsafeMutablePointer
在 Swift 中不能像 C 里那样使用 & 符号直接获取地址来进行操作，如果想对某个变量进行指针操作，可以借助 withUnsafePointer / withUnsafeMutablePointer 辅助方法。这两个方法接受两个参数，第一个是 inout 的任意类型，第二个是一个闭包。Swift 会将第一个输入转换为指针，然后将这个转换后的 Unsafe 的指针作为参数，去调用闭包。
```swift
// UnsafePointer
let ptr = withUnsafePointer(to: &a) { p in
}

// UnsafeMutablePointer
let ptr = withUnsafeMutablePointer(to: &a) { p in
    p.pointee += 1
}
```

#### unsafeBitCast
unsafeBitCast 是非常危险的操作，它会将一个指针指向的内存强制按位转换为目标的类型。因为这种转换是在 Swift 的类型管理之外进行的，因此编译器无法确保得到的类型是否确实正确，你必须明确地知道你在做什么，如：
```swift
let arr = ["moonlight"]
let str = unsafeBitCast(CFArrayGetValueAtIndex(arr as CFArray, 0), to: CFString.self)
print(str)
```
CFArrayGetValueAtIndex 函数用于对 arr 指定下标取值，返回的类型是 UnsafePointer<Void>，由于我们很明白其中存放的是 String 对象，因此可以直接将其强制转换为 CFString，但若类型不一致则会导致转换失败，会崩溃。

unsafeBitCast 常用于不同类型的指针之间的转换，使用如下：
```swift
var count = 100
let voidPtr = withUnsafePointer(to: &count, { (a: UnsafePointer<Int>) -> UnsafePointer<Void> in
    return unsafeBitCast(a, to: UnsafePointer<Void>.self)
})
// voidPtr 是 UnsafePointer<Void>。相当于 C 中的 void *

// 转换回 UnsafePointer<Int>
let intPtr = unsafeBitCast(voidPtr, to: UnsafePointer<Int>.self)
intPtr.pointee //100
```

#### bindMemory
bindMemory(to: T, capacity: Int) 将内存地址绑定成指定类型分布，新绑定类型必须与旧绑定类型布局兼容，此函数只绑定内存为指定类型分布，并不执行分配或初始化内存空间。
```swift
let p = UnsafeMutableRawPointer.allocate(byteCount: 32, alignment: 8)
let p2: UnsafeMutablePointer<Int> = p.bindMemory(to: Int.self, capacity: 8)
```

#### assumingMemoryBound
assumingMemoryBound 返回一个指向的内存区域已绑定数据类型的指针，
```swift
let p = UnsafeMutableRawPointer.allocate(byteCount: 32, alignment: 8)
let p2: UnsafeMutablePointer<Int> = p.assumingMemoryBound(to: Int.self)
p2[2] = 1
print(p2[2])
```

### 指针访问
#### MemoryLayout
MemoryLayout 可以计算类型的大小(size)、内存对齐大小(alignment)以及实际占用的内存大小(stride)，其细节如下：
```swift
public enum MemoryLayout<T> {

    public static var size: Int { get }

    public static var stride: Int { get }

    public static var alignment: Int { get }

    public static func size(ofValue value: T) -> Int

    public static func stride(ofValue value: T) -> Int

    public static func alignment(ofValue value: T) -> Int
}

MemoryLayout<Int>.size // return 8 (on 64-bit)
MemoryLayout<Int>.alignment // return 8 (on 64-bit)
MemoryLayout<Int>.stride // return 8 (on 64-bit)
```
一般在移动指针的时候，对于特定类型，指针一次移动一个stride（步长）,一般情况下stride是alignment的整数倍，即符合内存对齐原则；实际分配的内存空间大小也是alignment的整数倍，但是实际实例大小可能会小于实际分配的内存空间大小。

#### 访问内存空间
```swift
let ptr = UnsafeMutablePointer<Int>.allocate(capacity: 2)
// 初始化第一个空间
ptr.initialize(to: 7)
```
上面定义了 两个 Int 大小的空间，如何访问第二个空间并初始化呢？对于 可指定类型的 UnsafeMutablePointer 可以通过数组下标方式访问或者内存平移。
```swift
ptr[1] = 8
print(ptr[0])
print(ptr[1])

(ptr+1).pointee = 8
print(ptr.pointee)
print((ptr+1).pointee)
```
也可使用 successor() 函数，该函数返回指向下一个连续实例的指针，类似于 ptr + 1
```swift
// 初始化第2个空间
ptr.successor().initialize(to: 8)
print(ptr.pointee)
print(ptr.successor().pointee)
```
还可以通过 advanced(by: MemoryLayout) 指定步长移动
```swift
ptr.advanced(by: 1).initialize(to: 8)
print(ptr.pointee)
print(ptr.advanced(by: 1).pointee)
```
为何这里是 advanced(by: 1) 而不是 advanced(by: MemoryLayout<Int>.stride) ，是因为 ptr 是指定类型的 UnsafeMutablePointer，传 1 代表 1 个 Int 类型步长。
