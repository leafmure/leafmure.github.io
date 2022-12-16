---
title: Nginx 安装
date: 2018-11-28 15:26:11
categories: 服务器
tags:
- Nginx
keywords: Nginx 安装,Nginx
description:
---
> Nginx (engine x) 是一个高性能的HTTP和反向代理服务，也是一个IMAP/POP3/SMTP服务。Nginx是由伊戈尔·赛索耶夫为俄罗斯访问量第二的Rambler.ru站点（俄文：Рамблер）开发的，第一个公开版本0.1.0发布于2004年10月4日。
<!-- more -->
#### Nginx安装
yum源中的nginx版本比较旧，包安装可以指定安装版本，安装目录地址为 /usr/local/
##### 包安装
安装nginx所需的依赖库（需在root身份下）

###### pcre库
1. 下载 pcre库 安装包
```shell
# wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.36.tar.gz
```
2. 解压安装包
```shell
# tar -zxvf pcre-8.36.tar.gz
```
3. 编译安装
```shell
# cd pcre-8.36
# ./configure
# make && make install
```
###### zlib库
1. 下载 zlib库 安装包
```shell
# wget http://zlib.net/zlib-1.2.8.tar.gz
```
2. 解压安装包
```shell
# tar -zxvf zlib-1.2.8.tar.gz
```
3. 编译安装
```shell
# cd zlib-1.2.8
# ./configure
# make && make install
```
###### ssl库
1. 下载 zlib库 安装包
```shell
# wget http://www.openssl.org/source/openssl-1.0.1j.tar.gz
```
2. 解压安装包
```shell
# tar -zxvf openssl-1.0.1j.tar.gz
```
3. 编译安装
```shell
# cd openssl-1.0.1j
# ./configure
# make && make install
```
###### 依赖库安装完后就开始安装nginx
1. 下载 zlib库 安装包
```shell
# wget http://nginx.org/download/nginx-1.7.4.tar.gz
```
2. 解压安装包
```shell
# tar -zxvf nginx-1.7.4.tar.gz
```
3. 编译安装
```shell
# cd nginx-1.7.4
# ./configure
# make && make install
```
##### yum 安装
```shell
# yum install pcre pcre-devel
# yum install zlib zlib-devel
# yum install openssl openssl--devel
# yum install nginx
```

#### Nginx 配置
1. 首先查看nginx的安装地址，一般默认安装在 /etc/nginx/nginx.conf
```shell
# nginx -t  // root身份下
// 输出
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
2. 修改配置文件
```shell
# vim /etc/nginx/nginx.conf

// 修改格式如下
server {
listen       80 default_server; # 监听的端口
listen       [::]:80 default_server; # 监听的端口
server_name  lianghuii.com; # 域名
root         /home/hexo; # 网站根目录，需填写自己的目录地址，暂时没有用默认的

# Load configuration files for the default server block.
include /etc/nginx/default.d/*.conf;

location / {
}

error_page 404 /404.html;
location = /40x.html {
}

error_page 500 502 503 504 /50x.html;
location = /50x.html {
}
}

```
3. 修改保存后启动nginx服务
```shell
# systemctl start nginx.service
```
4. 访问自己服务器公网ip -> http://123.123.123.123:80（默认80端口），如果能访问成功证明nginx开启成功，如果访问失败
- 网页403  配置文件的网站根目录下为空
- 网页404  检查云服务器是否配置了 80 端口，然后检查防火墙是否开放了80端口

#### Nginx 命令
- nginx服务状态
```shell
# systemctl status nginx.service
```
- 启动nginx服务
```shell
# systemctl start nginx.service
```
- 停止nginx服务
```shell
# systemctl stop nginx.service
```
- 重启nginx服务
```shell
# systemctl reload nginx.service
```

#### 禁止IP直接访问网站
禁止别人直接通过IP访问网站，在nginx的server配置文件前面加上如下的配置，如果有通过IP直接访问的，直接拒绝连接。在配置前面加一个server，如下
```shell
server {
listen   80 default_server;
listen   [::]:80 default_server;
server_name  _;
return 444;
}
```
添加后：
```shell
server {
listen   80 default_server;
listen   [::]:80 default_server;
server_name  _;
return 444;
}

server {
listen       80;
server_name  lianghuii.com; # 域名
root         /home/hexo; # 网站根目录

# Load configuration files for the default server block.
include /etc/nginx/default.d/*.conf;

location / {
}

error_page 404 /404.html;
location = /40x.html {
}

error_page 500 502 503 504 /50x.html;
location = /50x.html {
}
}


```
