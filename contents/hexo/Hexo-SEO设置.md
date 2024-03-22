---
title: Hexo SEO设置
date: 2018-11-28 15:09:05
categories: hexo
tags:
- hexo 配置
keywords: hexo,hexo seo,seo
description:
---
> SEO（Search Engine Optimization）:汉译为搜索引擎优化。是一种方式:利用搜索引擎的规则提高网站在有关搜索引擎内的自然排名。目的是：为网站提供生态式的自我营销解决方案，让其在行业内占据领先地位，获得品牌收益；SEO包含站外SEO和站内SEO两方面；为了从搜索引擎中获得更多的免费流量，从网站结构、内容建设方案、用户互动传播、页面等角度进行合理规划，还会使搜索引擎中显示的网站相关信息对用户来说更具有吸引力  [=>百度百科](https://baike.baidu.com/item/搜索引擎优化/3132?fromtitle=seo&fromid=102990&fr=aladdin)
<!-- more -->
### 百度收录站点
登录[百度站长平台](http://zhanzhang.baidu.com)，在用户中心 => 站点管理添加你的站点网址

![Hexo-SEO设置01](http://upload-images.jianshu.io/upload_images/3850436-42f832ac05001ce7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

配置完站点属性后，进入最后一步：验证网站。有三种方式：文件验证、HTML标签验证、CNAME验证，文件验证和CNAME验证都比较简单，也有相对应的帮助文本，在此我选择的是HTML标签验证。

![Hexo-SEO设置01](http://upload-images.jianshu.io/upload_images/3850436-2947491429808daf?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 在主题的_config.yml文件中，设置：baidu_site_verification: true，如果没有该字段就手动添加。
2. 在themes/next/layout/_partials/head.swig文件中添加下列代码
```
// 每个人的content值都不一致，请注意更换成你的content值

{% if theme.baidu_site_verification %}
<meta name="baidu-site-verification" content="6K5YmdKWEx" />
{% endif %}
```
3. 配置好后，重新发布站点，在百度站长页面完成验证。

### 百度链接提交

> 链接提交工具是网站主动向百度搜索推送数据的工具，本工具可缩短爬虫发现网站链接时间，网站时效性内容建议使用链接提交工具，实时向搜索推送数据。本工具可加快爬虫抓取速度，无法解决网站内容是否收录问题

由于github屏蔽百度的爬虫，所以使用github page服务的站点的链接无法被抓取，可用[coding](https://coding.net/)的page服务。

#### 主动推送

> 最为快速的提交方式，建议您将站点当天新产出链接立即通过此方式推送给百度，以保证新链接可以及时被百度收录。

1. 安装百度链接提交插件
```
npm install hexo-baidu-url-submit --save 
```
2. 
```
# 百度链接自动提交
baidu_url_submit:
count: 6 # 提交最新的链接数量
host: http://lianghuii.com # 在百度站长平台中注册的域名
token:  # 请注意这是您的秘钥， 所以请不要把博客源代码发布在公众仓库里!
path: baidu_urls.txt # 文本文档的地址， 新链接会保存在此文本文档里
```
3. 设置deploye
```
deploy:
- type: git
repo:
github: git@github.com:Leafmure/Leafmure.github.io.git
coding: git@git.coding.net:Leafmure/blog.git
- type: baidu_url_submitter
```
#### 自动推送

> 是轻量级链接提交组件，将自动推送的JS代码放置在站点每一个页面源代码中，当页面被访问时，页面链接会自动推送给百度，有利于新页面更快被百度发现。。

1. 在主题配置文件将baidu_push设置为true
2. 在路径themes\next\layout_scripts\下创建baidu_push.swig 文件，文件内容如下
```
{% if theme.baidu_push %}
<script>
(function(){
var bp = document.createElement('script');
var curProtocol = window.location.protocol.split(':')[0];
if (curProtocol === 'https') {
bp.src = 'https://zz.bdstatic.com/linksubmit/push.js';        
}
else {
bp.src = 'http://push.zhanzhang.baidu.com/push.js';
}
var s = document.getElementsByTagName("script")[0];
s.parentNode.insertBefore(bp, s);
})();
</script>
{% endif %}

```

#### sitemap

> 您可以定期将网站链接放到Sitemap中，然后将Sitemap提交给百度。百度会周期性的抓取检查您提交的Sitemap，对其中的链接进行处理，但收录速度慢于主动推送。

使用npm自动生成网站的sitemap，然后将生成的sitemap提交到百度和其他搜索引擎
1. 安装sitemap插件
```
npm install hexo-generator-sitemap --save     
npm install hexo-generator-baidu-sitemap --save
```
2. 修改hexo配置文件
```
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://lianghuii.com
root: /
permalink: :title/
permalink_defaults:
```
执行完之后就会在网站根目录生成sitemap.xml文件和baidusitemap.xml文件,其中sitemap.xml文件是搜索引擎通用的文件，baidusitemap.xml是百度专用的sitemap文件。
3. 将生成的sitemap文件提交到百度站长

![Hexo-SEO设置03](http://upload-images.jianshu.io/upload_images/3850436-47cc13d7d2825cc1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 手动提交

> 如果您不想通过程序提交，那么可以采用此种方式，手动将链接提交给百度。

手动提交方式比较简单，按照官方提示就会使用 - -

![Hexo-SEO设置04](http://upload-images.jianshu.io/upload_images/3850436-34e550704a0e6bfd?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 文章页面配置
#### 缩短页面url路径
hexo 中文章页面url默认为：sitename/year/mounth/day/title，由于url层级过多爬虫不容易爬到文章，所以将页面url缩简为：sitename/title，在hexo 配置文件中修改permalink 配置项。
```
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://lianghuii.com
root: /
permalink: :title/
permalink_defaults:
```

#### 文章增加keywords、description等标识字段
在hexo 根目录中scaffolds文件中的文章模版文件中增加。
