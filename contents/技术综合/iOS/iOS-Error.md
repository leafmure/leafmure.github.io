---
title: iOS Error
date: 2018-11-28 15:38:08
categories: [技术综合, iOS]
tags: [iOS 问题集]
keywords: [Error, iOS Error, 报错, 错误]
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
```Objective-C
dispatch_async(dispatch_get_main_queue(), ^{
// 有关UI的操作
});
```

### Target 编译报错: Build input file cannot be found
Build 构建信息中提示:
None of the architectures in ARCHS (arm64) are valid. Consider setting ARCHS to $(ARCHS_STANDARD) or updating it to include at least one value from VALID_ARCHS (arm64, arm64e, armv7, armv7s) which is not in EXCLUDED_ARCHS (arm64, armv7).

需要检查 [Build Settings] 以下配置, 
- [Architectures] 表示使用的架构, 
- [Excluded Architectures] 表示需要排除的架构, 

此处问题在于: 
- [Architectures]: arm64
- [Excluded Architectures]: arm64、armv7

[Architectures] 配置被无效了, 导致无法编译. 需将 [Excluded Architectures] 中的 arm64 删除.
