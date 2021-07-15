---
title: next主题集成不蒜子统计
date: 2018-11-28 15:14:59
categories: hexo
tags:
- next 主题配置
keywords: 不蒜子统计,next主题
description:
---
打开next主题配置文件_config.yml 搜索busuanzi_count修改配置
```
// next 6.0 的配置样式
busuanzi_count:
enable: true
total_visitors: true
total_visitors_icon: user
total_views: true
total_views_icon: eye
post_views: true
post_views_icon: eye

// next 5.0 的配置样式
busuanzi_count:
enable: true
site_uv: true
site_uv_header: <i class="fa fa-user"></i> 本站访问数
site_uv_footer: 人
site_pv: true
site_pv_header: <i class="fa fa-eye"></i> 本站总访问量
site_pv_footer: 次
page_pv: false
page_pv_header: <i class="fa fa-file-o"></i> 本文总阅读量
page_pv_footer: 次
```
配置完后如发现不能统计, 可能是引用网址更换了，检查 themes\next\layout\_third-party\analytics\busuanzi-counter.swig 文件，更换引用地址为：https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js
