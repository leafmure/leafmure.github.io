---
title: 上传封装工具到Cocoapods
date: 2018-11-28 14:58:55
categories: 
- iOS
tags:
- Cocoapods
keywords: cocoapods,上传到Cocoapods
description:
---
##### 准备
在github、码云上创建项目，并把项目代码上传到项目仓库。

注册cocoapods
```
// 验证是否注册
$ pod trunk me
$ pod trunk register 邮箱号 昵称
// 然后该邮箱会收到一封cocoapods的邮件，只需打开查看邮件即可验证。
```

##### 创建Pod所对应的podspec文件
```
$ cd 项目下
$ pod spec create 项目名
```
配置podspec文件
```

Pod::Spec.new do |s|


s.name         = "LHCategoryTool"
s.version      = "0.0.1"
s.summary      = "LHCategoryTool is foundation category methods"

s.description  = <<-DESC
you can select some category methods of foundation that you want,this methods can help own improve develop quickly
DESC

s.homepage     = "http://gitee.com/meanmouse/LHCategoryTool"
s.license      = "MIT"

s.author             = { "MeanMouse" => "1871382624@qq.com" }

s.platform     = :ios, "8.0"

s.source       = { :git => "http://gitee.com/meanmouse/LHCategoryTool.git", :tag => "0.0.1" }

s.source_files  = "LHCategoryTool/**/*.{h,m}", "LHCategoryTool/**/*.{h,m}"

s.frameworks = "Foundation"

s.requires_arc = true

end
```
##### 添加至私有的Spec Repo
```
$ pod repo add 项目名 项目仓库地址
$ open ~/.cocoapods/repos/  // 执行可看见repos文件下有项目文件
```
##### 本地测试podspec文件是否可用
```
// 切换到repos中项目文件路径下
$ cd ~/.cocoapods/repos/项目名

// 测试本地库是否可用
$ pod lib lint

如果只有warning可执行下面这条命令：
$ pod lib lint --allow-warnings
// 之后的push操作也需加 --allow-warnings

```
创建新项目，pod 本地podspec文件
```
platform :ios, '8.0'

pod 项目名, :podspec => '~/.cocoapods/repos/项目名/项目名.podspec'  # 指定podspec文件

```
如果报错：Could not find remote branch 0.1(版本号) to clone
解决方案：
在github、码云仓库添加相对应版本的tag
##### 提交podspec到私有Spec Repo
```
$ cd ~/.cocoapods/repos/项目名/
$ pod repo push 项目名 项目名.podspec  #前面是本地Repo名字 后面是podspec名字
$ pod search 项目名
```
##### 测试Spec Repo中的的 podspec 是否可用
新建工程pod 导入
```
pod '项目名'
```

##### 私有库关联trunk 账号
私有库关联trunk 账号,将其上传至cocoapods库上，可供公共搜索
```
$ pod trunk push

// 执行下面命令搜索自己上传的工具库
$ pod search 项目名
// 如果出现如下错误：
/*
An unexpected version directory was encountered for the'<xxx>'Pod in the `xxx`
可能是因为和私有库中的项目文件冲突，
执行 open ~/.cocoapods/repos/
删除私有库中的项目文件，再次执行pod search 即可搜索成功
*/

```
