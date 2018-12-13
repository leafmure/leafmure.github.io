---
title: Git 安装
date: 2018-11-28 14:56:29
categories: Git
tags:
- Git安装
keywords: Git 安装,Git
description:
---
### Git是什么？
Git是分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目，它采用了分布式版本库的方式，不必服务器端软件支持。
### Git安装
首先在shell中输入git，看看系统有没有安装Git。
#### Linux
- 如果用的是Debian或Ubuntu Linux可用下面命令安装。
```
$ apt-get install git
```
- 可以通过yum 安装，只不过yum源中的版本比较旧点
```
$ yum install git
```
- 通过[Git官网](https://git-scm.com/)下载源码，然后解压，依次输入：./config，make， make install这几个命令安装就好了。
#### Mac OS X 上安装Git
1.通过[homebrew](http://brew.sh/)安装Git。
```
$ brew install git
```
2.通过Xcode -> Preferences,Downloads Command Line Tools，点击install就可以完成安装。
#### Windows
通过[Git官网](https://git-scm.com/)下载安装程序，在开始菜单找到Git - Git Bash出现 welcome to Git (version x.x.x -xxxx)，说明Git安装成功。

安装Git完成后，进行全局设置，表示你这台机器上所有的Git仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和Email地址。
```
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
$ git config 单个仓库 user.email "email@example.com"
$ git config 单个仓库 user.email "email@example.com"
```
