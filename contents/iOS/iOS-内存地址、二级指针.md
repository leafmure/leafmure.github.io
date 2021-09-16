---
title: iOS 内存地址、二级指针
date: 2021-09-13 19:16:48
categories:
- iOS
tags:
- 内存管理
keywords:
- 内存地址,二级指针
description:
images:
---
### %p
用于输出变量内存地址
```
NSString *a = @"six god";
NSLog(@"%p", a);
NSLog(@"%p", &a);

print: 0x100004008
print: 0x16fdff418
```
在日志输出中，a 和 &a 的区别是：
- 0x100004008 是字符串 @"six god" 内存首地址
- 0x16fdff418 是指针 *a 内存首地址，*a 保存着 0x100004008 内容地址
<!-- more -->
### 二级指针
在上面 *a 是一级指针，而二级指针就是指向指针的指针如：
```
// 下面代码只支持在 MRC 模式
// 在MRC下，二级指针类型定义为属性、成员变量或者参数都是可行的
// 在ARC下，二级指针类型则不能定义为属性或成员变量，但是可以作为参数传递

NSString *a = @"six god";
NSString** p = &a;
*p = @"six six"
```
可以通过*号来访问、操作内存，访问的过程为：二级指针访问一级指针的内容。一级指针存的是内容地址，系统会根据该地址访问一级指针指向的内容。二级指针 p 存的是 一级指针 a 的地址。

```
示例1：
NSString *a = @"six god";
NSLog(@"修改前: %@", a);
[self update:&a];
NSLog(@"修改后：%@", a);

- (void)update:(NSString **)v {
    *v = @"hi! six";
}

print:
修改前: six god
修改后：hi! six

示例2：
NSString *a = @"six god";
NSLog(@"修改前: %@", a);
[self update:a];
NSLog(@"修改后：%@", a);

- (void)update:(NSString *)v {
    *v = @"hi! six";
}

print:
修改前: six god
修改后：six god
```
示例1 中 update 方法中传的参数是指针，因此能访问原参数 a 修改内容，而示例2中传递的是内容地址(@"six god"存放的地址)，参数 v 是个新的参数即新的指针，指向传递的内容地址，随后 *v = @"hi! six"，访问的是指针 v 而不是原来的 a 了。 