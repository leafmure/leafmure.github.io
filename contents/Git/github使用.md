
# github使用

### 什么是github？
github是一个基于git的代码托管平台，付费用户可以建私人仓库，我们一般的免费用户只能使用公共仓库，也就是代码要公开。
> gitHub是一个面向开源及私有软件项目的托管平台，因为只支持git 作为唯一的版本库格式进行托管，故名gitHub。
gitHub于2008年4月10日正式上线，除了git代码仓库托管及基本的 Web管理界面以外，还提供了订阅、讨论组、文本渲染、在线文件编辑器、协作图谱（报表）、代码片段分享（Gist）等功能。目前，其注册用户已经超过350万，托管版本数量也是非常之多，其中不乏知名开源项目 Ruby on Rails、jQuery、python 等。[github百度百科](https://baike.baidu.com/item/github/10145341?fr=aladdin)

### 注册github
点击[github signUp注册](https://github.com/join)，

### 创建仓库
![屏幕快照 2017-12-25 下午9.36.09.png](http://upload-images.jianshu.io/upload_images/3850436-ed7d2689f7eed077.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 配置ssh key
复制下方“显示隐藏文件”命令至终端，，前往Users/lianghui/.ssh查看是否有.ssh文件，如果有建议删除。

显示隐藏文件
```
$ defaults write com.apple.finder AppleShowAllFiles Yes && killall Finder 
```
不显示隐藏文件
```
$ defaults write com.apple.finder AppleShowAllFiles No && killall Finder 
```

终端里输入指令,创建一个 .ssh 文件夹
```
$ mkdir .ssh
```

配置ssh
```
$ ssh-keygen -t rsa -C "your_email@youremail.com"
```
双引号中填写你github注册用的邮箱地址，之后会提示你要输入密码，不用管一直按回车直至出现
```
The key‘s randomart image is:
+--[ xxx xxxx]----+
|                 |
|  x              |
| xx   x          |
|  x     x        |
+-----------------+
```

输入指令，复制ssh key
```
$ pbcopy < ~/.ssh/id_rsa.pub 
```
登陆github，点击头像 -> Settings -> SSH and GPG keys
点击New SSH Key，输入title(自己命名)，将复制到的ssh key复制到key栏中，然后add，界面钥匙显示灰色。

为了验证是否成功，在终端中下输入
```
$ ssh -T git@github.com
```
如果是第一次会出现Are you sure you want to continue connecting (yes/no)?
输入yes回车，出现You've successfully authenticated, but GitHub does not provide shell access，返回github界面钥匙显示绿色 ，这就表示已成功连上github。

### 上传文件至github
如果是第一次提交没有git仓库，cd到本地项目根目录创建
```
$ git init
```
如果有git仓库，检出仓库
```
$ git clone  仓库地址
```
将要上传的文件移至检出的仓库文件夹

查看仓库文件状态
```
$ git status
```
将状态为未添加的文件添加至git
单个添加
```
$ git add 文件名
```
全部添加

```
$ git add . //监控工作区的状态树,提交文件内容修改(modified)、新文件(untracked file),但不包括被删除的文件(delete)
$ git add -u //仅监控已经被add的文件(tracked file),不会提交新文件(untracked file)(git add --update的缩写）
$ git add -A //是上面两个功能的合集(git add --all的缩写)
```

提交文件至本地仓库的HEAD

```
$ git commit -m "此次上传的描述"
```
将改动提交至远端仓库(master为仓库分支)
```
$ git push origin master
```
然后就可以在github上看到自己上传的文件

