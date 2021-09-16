---
title: iOS Error
date: 2018-11-28 15:38:08
categories:
- iOS
tags:
- iOS 问题集
keywords: Error,iOS Error,报错,错误
description:
---
### 离开页面后，网络请求使用 delegate 回调崩溃
原因：用的是 assgin 修饰 delegate，像数值类型一样 delegate对象被销毁了但不置 nil。
<!-- more -->
解决方案：
- 使用weak修饰delegate对象，离开页面后，delegate对像销毁并置 nil。
- 离开页面时，手动将delegate对象置nil。
- 离开页面时，取消这个网络请求。

### This application is modifying the autolayout engin from a background thread
原因：未回主线程刷新UI

解决方案： 
```
dispatch_async(dispatch_get_main_queue(), ^{
// 有关UI的操作
});
```
