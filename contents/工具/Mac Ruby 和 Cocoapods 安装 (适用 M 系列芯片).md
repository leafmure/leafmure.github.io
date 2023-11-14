---
title: Mac Ruby 和 Cocoapods 安装 (适用 M 系列芯片)
date: 2023-07-06 17:35:47
categories:
- 工具
tags:
- Ruby
- Cocoapods
keywords:
- Ruby,Cocoapods,HomeBrew,M1,M2,Mac
description:
images:
---
# Mac Ruby 和 Cocoapods 安装 (适用 M 系列芯片)
## 前言
Ruby 和 Cocoapods 是 iOSer 必接触的工具, Cocoapods 是 Ruby 程序包, 因此安装 Cocoapods 前置条件要有一个适用的 Ruby 版本, 系统有自带 Ruby, 但版本比较低且其执行目录无权限访问, 所以我们需要去下载安装一个比较高版本的 Ruby
<!-- more -->

## Ruby 安装
### RVM
Ruby 的下载和管理, 一般会通过 rvm 包管理工具, 如何检查 rvm 是否安装? 终端执行以下命令, 有输出版本号即为成功.
```bash
rvm --version
 
output:
rvm 1.29.12 (latest) by Michal Papis, Piotr Kuczynski, Wayne E. Seguin [https://rvm.io]
```
如果无安装, 可通过以下步骤进行安装
```bash
# 安装 rvm
\curl -sSL https://get.rvm.io | bash -s stable
 
# 更新 rvm
rvm get stable
```
安装 rvm 成功后, 即可通过 rvm 去安装 ruby, 如何安装可参考以下命令
```bash
# 列出 ruby 可安装的版本信息
rvm list known
 
# 安装指定版本
rvm install 版本名
 
# 查看已安装版本
rvm list
 
# 当前 shell 指定使用版本
rvm use 版本名
 
# 设置默认使用版本
rvm use 版本名 --default
```
一般来说, 通过 rvm 安装 ruby 一般会顺利完成, 但是最近在 M 系列 Mac 上一直卡在一个问题报错上:  “Error running __rvm_make -j8”, 网上查询好多资料和博文都未得到有效解决, 最终放弃 rvm 方式, 通过 homebrew 安装 ruby.

## HomeBrew 方式安装 Ruby
homebrew 是 Mac 常用的包管理器，用于安装和管理各种开源软件. 
```bash

# 安装命令
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
 
# 卸载命令, 如安装的版本异常, 建议卸载重装
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/HomebrewUninstall.sh)"
 
# 升级 brew
brew upgrade
```
homebrew 安装后, 就可以开始安装 Ruby了
```bash
# 安装 Ruby, 此命令会安装最新版本
brew install ruby
```
安装之后, 此时查询的 ruby 版本还是系统自带的, 需要配置环境变量, 才能让新版本的生效
```bash
vi ~/.bash_profile
 
# ruyb
# intel 芯片
if [ -d "/usr/local/opt/ruby/bin" ]; then
  export PATH=/usr/local/opt/ruby/bin:$PATH
  export LDFLAGS="-L/usr/local/opt/ruby/lib"
  export CPPFLAGS="-I/usr/local/opt/ruby/include"
  export PATH=`gem environment gemdir`/bin:$PATH
fi
# m1 silicon 芯片
if [ -d "/opt/homebrew/opt/ruby/bin" ]; then
  export PATH=/opt/homebrew/opt/ruby/bin:$PATH
  export LDFLAGS="-L/opt/homebrew/opt/ruby/lib"
  export CPPFLAGS="-I/opt/homebrew/opt/ruby/include"
  export PATH=`gem environment gemdir`/bin:$PATH
fi
```
编辑完后记得执行: source ~/.bash_profile, 让配置生效, 查询 ruby 版本即为最新版本啦
```bash
# 查询 ruby 版本
ruby -v
```

## Cocoapods 安装
### RubyGems
Ruby 安装完后, 需要检查 RubyGems 源, RubyGems 是一个用于管理 Ruby 程序包（也称为 gem）的软件包管理系统.
```bash
# 查看当前 ruby 的源
gem sources -l
 
# 移除 ruby 的源
gem sources --remove 源地址
 
# 添加 ruby 源: https://gems.ruby-china.com  (国内源, 不会被墙)
gem sources -a https://gems.ruby-china.com
```
接下我们需要检查 gem 版本, 如果太旧需要更新.
```bash
# 查看版本
gem -v
 
# 更新版本
sudo gem update --system
```

### 开始安装Cocoapods
为防止旧版本影响, 可先移除旧版本再安装. 可以通过 sh 脚本进行卸载
```bash
# remove.sh
for i in $( gem list --local --no-version | grep cocoapods );
do
    sudo gem uninstall $i;
done
 
# 终端执行 remove.sh, 文件拖入终端, 直接回车
./remove.sh (这是remove.sh文件路径)
 
# 配置目录 ~/.cocoapods 删除
rm -rf ~/.cocoapods
 
# 开始安装 cocoapods
sudo gem install -n /usr/local/bin cocoapods
```
好啦, 到这步 Cocoapods 已安装完成, 如未完成可留言, 祝你好运!
