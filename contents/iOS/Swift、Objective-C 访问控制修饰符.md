---
title: Swift、Objective-C 访问控制修饰符
date: 2022-03-10 16:17:14
categories: 
- iOS
tags:
- 访问控制修饰符
keywords: Swift,Objective-C,访问控制,修饰符
---
### 一、引言
Swift、Objective-C 都具有面向对象的三大特征：封装、继承和多态，访问控制修饰符应“封装”特性诞生的，“封装”需将对象的实现信息隐藏在内部，不允许外部直接访问对象内部信息，只能通过向外开放途径去访问对象。对象的那些信息需要隐藏或者开放，这可通过 “访问控制修饰符” 修饰，告知编译器。
<!-- more -->
### 二、Swift 的访问控制修饰符
#### 2.1  private
private 修饰的信息，仅能在当前定义的作用域访问，让我们看一下，Swift 中常使用的 “单例”实现模式：
```swift
class Single {
   static let instance = Single()
   private init() {} 
}
let share = Single.instance
```
将 init() 构造器用 private 修饰后，Single 外部无法通过构造器创建实例，只能通过静态变量 instance 获取 Single 实例

#### 2.2 filePrivate
filePrivate 修饰的信息，只能在当前定义的源文件中才能访问
```swift
/// Person.swift
fileprivate class Person {
   var name: String?
}

class XiaoMing: Person {
   var father: Person
}

/// Family.swift
不能访问 Person
```
#### 2.3  Internal
Internal 默认访问级别，允许定义模块（Target）中的任意源文件访问，但不能被该模块之外的任何源文件访问
```swift
/// TargetOne
class Person {
   var name: String?
}
let p = Person()

/// TargetTwo
import TargetOne
let p2 = Person() -> error: 'Person' initializer is inaccessible due to 'internal' protection level
```
上述代码中，默认的访问级别为 Internal，在 TargetOne 模块内都可访问，当 TargetTwo 跨模块访问  TargetOne 内容时，即使 import TargetOne 也无法访问 Person 内容。这种情况常见于主项目 Target 和 Cocoapods 第三方框架。

#### 2.4 public
public 开放式访问，使我们能够在其定义模块的任何源文件中使用代码，并且可以从另一个源文件访问源文件。但是只能在定义的模块中继承和重写，在 SDK 封装中使用比较多。
```swift
/// TargetOne
public class Person {
  public init() {}
}
let p = Person()

/// TargetTwo
import TargetOne
let p2 = Person()

class Student: Person { -> error: Cannot inherit from non-open class 'Person' outside of its defining module
}
```
2.3 中的 Person 类如果加上 Public 修饰，那么 TargetTwo 中只要 import TargetOne 就能访问 Person 类，但是没法继承 Person。

#### 2.5 open
open 是最高权限的访问级别，它和 public 的区别就在于：可以在任意地方、任意模块间继承、重写。如果将上述 2.4 的 Person 类采用 Open 修饰便不会报错。

#### 2.6 访问控制符权限排序
访问控制权限从高到低的顺序：open > public > internal > filePrivate > private，变量只能接受比自己访问权限高的值，若值的访问权限比定义的变量高，编译器会报错。
```swift
fileprivate class Person {
}
let p1 = Person() -> error: Constant must be declared private or fileprivate because its type 'Person' uses a fileprivate type
/// 下面这样写便不会报错
private let p2 = Person()
fileprivate let p3 = Person()
```
### 三、Objective-C 的访问控制修饰符
相对于 Swift，Objective-C 的访问控制修饰符只有：@private、@package、@protected、@public，一般都是用于修饰成员变量。方法、属性的隐藏和开放，更多是通过声明文件 .h、.m 控制。

#### 3.1  @private
@private 和 private 权限一样，只能在当前类内部（.m 文件）访问。如果类的成员变量使用@private访问控制符来限制，则这个成员变量只能在当前类的内部被访问。这个访问控制符用于彻底隐藏成员变量。在类的实现部分（.m 文件）定义的成员变量相当于默认使用这种访问权限。

#### 3.2  @package
框架类访问权限，这个访问控制符用于部分隐藏成员变量，如果类的成员变量使用 @package 访问控制符来限制，则当前类或者同一框架中的其他类能够访问这个成员变量。

#### 3.3 @protected
子类访问权限，这个访问控制符用于部分隐藏成员变量。如果类的成员变量使用@protected访问控制符来限制，则当前类或者其子类能够访问这个成员变量。

#### 3.4 @public
公共访问权限，如果类的成员变量使用 @public 访问控制符来限制，则任何框架的类都能够访问这个成员变量，这个访问控制符用于彻底暴露成员变量，一般不建议这么做。