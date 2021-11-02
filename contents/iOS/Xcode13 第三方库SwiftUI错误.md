---
title: Xcode13 Kingfisher、RealmSwift Release模式编译报错
date: 2021-11-01 16:38:42
categories: iOS
tags:
- iOS 问题集
keywords: Xcode13,Kingfisher,RealmSwift,Swift UI
description:
images:
---

Xcode 13 发行说明中提到：
> Swift 库可能无法为使用 armv7 的 iOS 目标构建。(74120874)
解决方法：将包的平台依赖性增加到 v12 或更高版本。
> 依赖于 Combine 的 Swift 库可能无法为包括 armv7 和 i386 架构在内的目标构建。(82183186, 82189214)
> 解决方法：使用不受影响的库的更新版本（如果可用）或删除 armv7 和 i386 支持（
例如，将库的部署目标增加到 iOS 11 或更高版本）。
<!-- more -->
Kingfisher、RealmSwift 库处于上述情况，如果项目中使用的是 Swift UI 则需要将最低兼容版本改为 iOS 12。由于个人项目需要兼容 iOS 10 并且项目中也没有使用到 Swift UI，所以修改 pod file，在 pod install 时将Kingfisher、RealmSwift库的 Swift UI 相关代码删除。
```
pre_install do |installer|
  remove_Kingfisher_swiftui()
  remove_RealmSwift_swiftui()
end

def remove_Kingfisher_swiftui
  code_file = "./Pods/Kingfisher/Sources/General/KFOptionsSetter.swift"
  code_text = File.read(code_file)
  code_text.gsub!(/#if canImport\(SwiftUI\) \&\& canImport\(Combine\)(.|\n)+#endif/,'')
  file = File.new(code_file, 'w+')
  file.syswrite(code_text)
  file.close()
end

def remove_RealmSwift_swiftui
  code_file = "./Pods/RealmSwift/RealmSwift/SwiftUI.swift"
  code_text = File.read(code_file)
  code_text.gsub!(/#if canImport\(SwiftUI\) \&\& canImport\(Combine\)(.|\n)+#else/,'')
  code_text = code_text.gsub(/#endif/,'')
  file = File.new(code_file, 'w+')
  file.syswrite(code_text)
  file.close()
end
```

在 Ruby 中 gsub 函数用于替换字符，示例用法如下：
```
'abc'.gsub(/a/,d) => 'dbc'
'abc'.gsub!(/a/,d) => 'dbc'
'abc'.gsub(/d/,e) => 'abc'
'abc'.gsub!(/d/,e) => nil
```
若有其他第三方库遇到这种情况，可以尝试根据上面的示例编写对应函数，删除 Swift UI 代码。
