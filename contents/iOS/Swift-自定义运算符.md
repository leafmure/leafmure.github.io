---
title: Swift-自定义运算符
date: 2022-02-10 16:44:17
categories:
- iOS
tags:
- 自定义运算符
keywords:
- Swift,自定义运算符
description:
images:
---
##### 运算符定义类型
```
var a = 0
var b = 1
a += 1
a ^ b
a--
--a
```
上面代码的运算是如何实现的呢？
在 Swift 中提供三种类型运算符：
- prefix：前置运算符，像 --a、++a
- infix：中间运算符，像 a += 1、a + b
- postfix：后置运算符，像 a++、a--
<!-- more -->
具体应用如下：

```
prefix func --(v: inout Int) {
    v = v - 1
}
var a = 0
--a
```
```
infix operator ++
func ++(leftv: Int, rightv: Int) -> Int {
    return leftv + rightv + rightv
}
a ++ 1

```
```
postfix func --(v: inout Int) {
    v = v - 1
}
a--
```
可以对系统的运算符的实现函数重载，但是对于“=” 类型的运算符是无法重载的。(注意：重载和重写的区别，运算符实现不支持重写)


##### 自定义运算符
Swift 支持声明和实现自定义运算符，自定义运算符需要在全局作用域，通过关键字 operator 进行定义，同时要指定运算符类型，运算符命名只能包含这些字符：/ = - + * % < >！& | ^。~
```
/ = - + * % < >！& | ^。~

postfix operator ~
```