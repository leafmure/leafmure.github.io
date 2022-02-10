---
title: cocoapods-私有库和公开库
date: 2022-02-10 17:40:13
categories:
- iOS
tags:
- cocoapods
keywords:
- cocoapods,私有库,公开库
description:
images:
---
> 私有库：git 仓库设置为私有不对外公开

> 公开库：提交到 Cocoapods Specs 上，可以 pod search 到库
<!-- more -->
### 私有库
#### 私有索引库
私有索引库用于保存私有库的 .podspec 文件，pod install 私有库时会去私有索引库找到对应的 .podspec 文件。

首先去git托管平台(如：Github)创建私有索引库：Private-Specs，然后将该仓库加入到本地索引库。
```
# 加入到本地索引库
$ pod repo add 索引库名 仓库地址
es: pod repo add Private-Specs https://github.com/es/Private-Specs.git

# 查看本地索引库
$ pod repo
# ====== Out-print: ======
master
- Type: git (master)
- URL:  https://github.com/CocoaPods/Specs.git
- Path: /Users/name/.cocoapods/repos/master

Private-Specs
- Type: git (main)
- URL:  https://github.com/es/Private-Specs.git
- Path: /Users/name/.cocoapods/repos/podSpecs

trunk
- Type: CDN
- URL:  https://cdn.cocoapods.org/
- Path: /Users/name/.cocoapods/repos/trunk

```

#### 私有库创建
在git托管平台(如：Github)创建私有库：Tool，然后在本地创建 git 本地仓库。
```
$ pod lib create 私有库名
es: pod lib create Tool

# ====== Out-print: ======

Configuring Tool template.

------------------------------

To get you started we need to ask a few questions, this should only take a minute.

If this is your first time we recommend running through with the guide: 
 - https://guides.cocoapods.org/making/using-pod-lib-create.html
 ( hold cmd and double click links to open in a browser. )
# 库的使用平台
What platform do you want to use?? [ iOS / macOS ]
 > iOS
# 编写语言
What language do you want to use?? [ Swift / ObjC ]
 > Swift
# 是否创建 Example 工程
Would you like to include a demo application with your library? [ Yes / No ]
 > Yes
# 采用什么测试框架
Which testing frameworks will you use? [ Quick / None ]
 > None
# 是否需要视图测试
Would you like to do view based testing? [ Yes / No ]
 > No
```
创建后目录结构如下：
```
- Tool：库实现代码文件
    - Assets：资源文件
    - Classes：代码文件
- Example：库使用的示例工程
- Tool.podspec 索引文件
```
将实现代码文件放入 Tool/Classes 中，cd到Example中执行 pod install，Example 工程就可以访问 Tool/Classes 代码了。

#### 配置索引文件
```
Pod::Spec.new do |s|
  s.name             = '库名'
  # 和仓库的tag名称关联，如：git tag List中要有：0.1.0
  s.version          = '0.1.0'
  # 简介
  s.summary          = ''

# This description is used to generate tags and improve search results.
#   * Think: What does it do? Why did you write it? What is the focus?
#   * Try to keep it short, snappy and to the point.
#   * Write the description between the DESC delimiters below.
#   * Finally, don't worry about the indent, CocoaPods strips it!

  # 描述
  s.description      = 'none'

  s.homepage         = 'https://github.com/es/Tool'
  # s.screenshots     = 'www.example.com/screenshots_1', 'www.example.com/screenshots_2'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'Username' => 'email' }
  s.source           = { :git => 'https://github.com/es/Tool.git', :tag => s.version.to_s }
  # s.social_media_url = 'https://twitter.com/<TWITTER_USERNAME>'

  s.ios.deployment_target = '10.0'

  s.source_files = 'Tool/Classes/**/*'
  
  # s.resource_bundles = {
  #   'Tool' => ['Tool/Assets/*.png']
  # }

  # s.public_header_files = 'Pod/Classes/**/*.h'
  # 以来的框架
  # s.frameworks = 'UIKit', 'MapKit'
  # 依赖的第三方
  # s.dependency 'Result'
  
  # 创建子索引
  s.subspec 'Extension' do |e|
    e.source_files = 'Tool/Classes/Extension/**/*'
  end

end
```
配置好后，关联好 Tool 本地仓库和远程仓库，将内容提交到本地仓库并创建 tag："0.1.0", 将内容和 tag 推送到远程仓库。

#### 私有库索引提交到私有索引库
cd 到 Tool 根目录，分别使用 pod lib lint 和 pod spec lint 命令进行podspec的本地校验和远程校验, 校验通过后便提交到私有索引库。
```
# 提交到私有索引库
$ pod repo push 索引库名 podspec文件名
es: pod repo push Private-Specs Tool.podspec

# 输出执行日志
--verbose 
# 忽略警告，因警告而无法通过校验可在命令后追加
--allow-warnings
es：pod lib lint --allow-warnings
es：pod repo push Private-Specs Tool.podspec --allow-warnings

```
pod 私有库时，需要在 podfile 添加私有索引库地址
```
# 私有索引库
Source 'https://github.com/es/Private-Specs.git'
# 私有库依赖了 cocoapods 公开第三方需再添加这个
Source 'https://github.com/CocoaPods/Specs.git'
```


