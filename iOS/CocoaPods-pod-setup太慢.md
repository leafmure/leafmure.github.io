---
title: CocoaPods pod setup太慢
date: 2018-11-28 15:39:50
categories: 
- iOS
- iOS 问题集
tags:
- CocoaPods
keywords: CocoaPods,pod setup,pod setup太慢
description:
---
#### 更换镜像pod setup
1.升级Ruby环境
```
$ sudo gem update --system
```
2.更换Ruby镜像
```
$ gem sources --remove https://rubygems.org/   移除现有镜像
$ gem sources -a https://gems.ruby-china.org/  添加国内最新镜像
$ gem sources -l  查看当前镜像
$ sudo gem install cocoapods  安装cocoapods
$ sudo gem install -n /usr/local/bin cocoapods
$ pod setup  挂vpn，耐心等待
```
3.pod setup后，pod search 第三方框架名，结果如下则为成功
![image.png](https://upload-images.jianshu.io/upload_images/3850436-fd1eaaafdc4f8cbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4.如果pod search 出现Unable to find a pod with name, author, summary, or descriptionmatching，删除~/Library/Caches/CocoaPods目录下的search_index.json文件
```
$ rm ~/Library/Caches/CocoaPods/search_index.json
$ pod search '第三方框架名'
```
#### 手动pod setup (未尝试)
```
$ cd ~/.cocoapods/repos  该文件夹pod setup会创建
$ git clone https://github.com/CocoaPods/Specs.git  克隆git仓库
```
下载完后，将Specs 改名为master，然后执行
```
$ pod repo
```

#### import pod第三方库不提示
1. 选择target —> BuildSettings —> search Paths 下的 User Header Search Paths。
2. 双击User Header Search Paths后面的空白区域。
3. 点击“+”号添加一项：并且输入：“$(PODS_ROOT)”（没有引号），选择：recursive（会在相应的目录递归搜索文件）。
