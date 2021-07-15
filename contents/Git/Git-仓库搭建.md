---
title: Git 仓库搭建
date: 2018-11-28 15:29:59
categories: 服务器
tags:
- Git仓库搭建
keywords: Git 仓库搭建,Git,服务器
description:
---
#### 创建git新用户管理git仓库
1. 创建git用户并配置
```
# adduser git  // 创建git用户
# passwd git  // 设置git用户的密码
# chmod -v u+w /etc/sudoers  // 赋予读写权限
// 添加新用户信息至/etc/sudoers
# vim /etc/sudoers

// 添加新用户信息如下
// Allow root to run any commands anywhere 
root ALL=(ALL) ALL
git ALL=(ALL) ALL
```
2. 生成密钥（在电脑的shell中执行）
```
$ ssh-keygen -t rsa  // 一直回车会出现 key’s randomart image
$ pbcopy < ~/.ssh/id_rsa.pub // 复制id_rsa.pub中的密钥
```
3. 密钥配置（在服务器配置）
```
# su git  // 切换至git用户，如果当前是不必执行
# mkdir ~/.ssh // 创建.ssh文件
# vim ~/.ssh/authorized_keys // 将上步骤获取的密钥复制到authorized_keys文件里

# cd ～ // git用户下的根目录
// 设置权限
# chmod 600 .ssh/authorzied_keys
# chmod 700 .ssh
```
4. 测试是否能连接服务器
```
$ ssh -v git@SERVERIP // SERVERIP 为服务器公网ip
```
- 如果不用输入密码直接进入：能连接服务器并且rsa密钥配置成功
- 如果需要输入密码进入：未配置密钥或者密钥无效
- 如果出现 Host key verifocation failed：检查本地密钥是否存在 known_hosts文件，存在的话将其删除
- 其他未知结果：无法访问服务器

5. 建立git空库，也叫裸库，只有一个.git文件。在git用户下执行，因为在上步骤配置的权限只有git用户拥有所有权。
```
# cd /home/git  // 进入git用户目录下
# git init --bare 仓库名称.git // 创建git仓库
# chown git:git -R 仓库名称.git // 配置新建的git仓库的权限
```
6. 测试是否创建成功
```
// clone git仓库
$ git clone git@SERVERIP:/home/git/仓库名称.git
```

#### 使用 git-hooks 同步到其他目录下
将git仓库的内容自动同步至其他目录
```
# vim 仓库名称.git/hooks/post-receive

在post-receive文件下加以下内容：
git --work-tree=目标目录 --git-dir=git仓库地址 checkout -f
如：git --work-tree=/home/hexo --git-dir=/home/git/blog.git checkout -f
```
配置post-receive的可执行权限
```
# chmod +x /home/git/blog.git/hooks/post-receive
```
