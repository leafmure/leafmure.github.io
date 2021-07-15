---
title: Hexo项目同步github
date: 2018-11-28 15:04:50
categories: hexo
tags:
- hexo 文件管理
keywords: hexo,hexo同步github,hexo托管到github
description:
---
#### hexo文件托管到github

##### 克隆gitHub上的XXX.github.io项目的文件到本地 
```
git clone https://github.com/yourname/xxx.github.io.git
```
##### 删除文件夹里除了.git的其他所有文件 
##### 把hexo项目文件下的所有文件全部复制过来(除了.git)
##### 里面应该有一个叫.gitignore的文件，如果没有就输入 touch .gitignore，创建一个,并输入以下内容
```
.DS_Store 
Thumbs.db 
db.json 
*.log 
node_modules/ 
public/ 
.deploy*/ 
```
##### 创建一个叫hexo的分支并切换到这个分支上 
```
git checkout -b hexo 
```
##### 提交复制过来的文件到暂存区
```
$ git add --all
```
##### 因为thme/next是一个git仓库，所以git add 会失败
###### 解决方案一
删除主题next中的.git文件，然后就可以add了。

###### 解决方案二
新建一个github仓库，将next本地仓库所关联的远程仓库更改为自己新建的仓库地址并上传文件，然后将新建的仓库作为hexo分支的子模块
```
// 查看所有远程仓库
$ git remote  
// 删除所关联的远程仓库地址
$ git remote rm origin  
// 添加新仓库地址
$ git remote add origin 新仓库地址
// 提交至新仓库
$ git push orign master 
// cd hexo分支文件下，添加next仓库作为hexo分支的子模块
$ git submodule add 仓库地址 themes/next
```
##### cd到主项目路径，将修改文件添加到暂存区，提交本地仓库并同步到GitHub上
```
$ cd ../../ //切到仓库的根目录
$ git add .
$ git commit -m "update next settings in blog sources branch"
$ git push origin hexo //注意hexo分支
```

##### 使用
项目的master分支的内容是由hexo分支生成的，所以使用的时候只需将项目的hexo分支clone下来就行，另外如果theme/next是用子模块管理的，需cd到next文件下执行以下命令获取next主题文件
```
$ git submodule init  // 初始化本地配置文件
$ git submodule update  // 抓取所有数据并检出项目中列出的合适的提交
```
每次更改了next主题文件，记得将更新内容提交至next主题的远程仓库并提交hexo分支！

#### 建议
hexo 项目是用于生成 静态网页资源的，里面包含着你的网页配置信息和文章，如果不想让人知道你的hexo 项目内容，应将 hexo 文件存放于私有仓库，将静态网页资源放公开仓库也就是XXX.github.io仓库，让别人通过这个公开仓库访问你的网页。上面的做法是将静态网页资源和 hexo项目文件 放到同一仓库的不同分支，由于XXX.github.io仓库是公开的，别人可以看到你的 hexo 项目文件的内容。
