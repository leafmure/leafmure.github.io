---
title: AppStrore上架拒审问题
date: 2018-11-28 15:33:13
categories:
- iOS
- iOS 问题集
tags:
- iOS 问题集
keywords: AppStrore上架,上架拒审
description:
---
### Guideline 2.5.1 - Performance - Software Requirement，通过"prefs:root ="调用私有Api
解决方案：
- 通过"prefs:root ="调用私有Api在iOS8之前可用，iOS8之后不再通过私有Api跳转到相应设置，统一跳到设置界面。
```
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:UIApplicationOpenSettingsURLString]];
```
- 将"prefs:root ="字段转化，将转换结果再还原调用。
```
//iOS10
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:@"prefs:root= Bluetooth"] options:@{} completionHandler:nil];
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:@"prefs:root= Bluetooth"]];

// 将字符串转换为16进制
// (unsigned char []){0x70,0x72,0x65,0x66,0x73,0x3a,0x72,0x6f,0x6f,0x74,0x3d,0x4e,0x4f,0x54,0x49,0x46,0x49,0x43,0x41,0x54,0x49,0x4f,0x4e,0x53,0x5f,0x49,0x44}
NSData *encryptString = [[NSData alloc] initWithBytes:(unsigned char []){0x70,0x72,0x65,0x66,0x73,0x3a,0x72,0x6f,0x6f,0x74,0x3d,0x4e,0x4f,0x54,0x49,0x46,0x49,0x43,0x41,0x54,0x49,0x4f,0x4e,0x53,0x5f,0x49,0x44} length:27];
NSString *string = [[NSString alloc] initWithData:encryptString encoding:NSUTF8StringEncoding];
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:string] options:@{} completionHandler:nil];
```
