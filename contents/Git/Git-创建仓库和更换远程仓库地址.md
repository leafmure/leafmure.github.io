
# Git 创建仓库和更换远程仓库地址

#### 创建仓库
##### 本地初始化git仓库
```
// cd 至项目目录下，初始化本地仓库
$ git init
```

##### 添加远程仓库
```
// 首先需在github、码云创建远程仓库（项目）
// 复制远程仓库地址
$ git remote add origin 仓库地址
```
##### 本地仓库与远程仓库关联
```
$ git branch --set-upstream-to=origin/master
$ git branch --set-upstream-to=origin/<branch> master
```

##### 拉取远程仓库内容到本地仓库
```
$ git pull

// 若git pull提示：fatal: refusing to merge unrelated histories
// 改用以下命令
$ git pull origin master --allow-unrelated-histories

```

##### 将最新新内容推送至远程仓库
```
// 使用git status 查看是否有内容未添加至暂存区
$ git status

// 将内容添加至暂存区
$ git add -A  // 全部添加

// 将暂存区内容提交至本地仓库
$ git commit -m "本次提交内容信息"

// 将本地仓库推送至远程仓库
$ git push
```

#### 更换远程仓库地址
- 通过命令直接修改地址
```
// 查看仓库地址
$ git remote -v

// 更换仓库地址
$ git remote set-url origin 新仓库地址
```
- 删除仓库地址然后添加新的仓库地址
```
// 删除仓库地址
git remote rm origin

// 添加新的仓库地址
git remote add origin 新仓库地址
```
- 直接修改本地的.git/config 文件
```
// git 文件是隐藏文件
// 显示隐藏文件
$ defaults write com.apple.finder AppleShowAllFiles Yes && killall Finder 
// 不显示隐藏文件
$ defaults write com.apple.finder AppleShowAllFiles No && killall Finder 

// .git/config 文件内容
[remote "origin"]
url = 更换为新仓库地址
fetch = +refs/heads/*:refs/remotes/origin/*

```
