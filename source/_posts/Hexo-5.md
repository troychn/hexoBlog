---
title: Hexo教程(五)-hexo博客jacman主题首页优化 #可以改成中文的，如“新文章”
date: 2016-06-27 22:09:23 #发表日期，一般不改动
updated: 2016-06-27 13:09:23
categories: hexo #文章文类
tags: [hexo,hexo优化,jacman] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---
# 前言：
上一篇说到了hexo的博客被百度和google收录的方法，虽然搜索引擎是收录了，但是我在百度上搜索我文章的标题，还是没有搜索结果，结果只有首页一个。这个不知道是什么原因，有可能是百度索引量的问题，后续再观察，再折腾吧。下面来分享一下优化jacman主题，包括首页和文章页等，
# 正文
jacman主题优化之首页优化,顺序从上到下，从左到右进行优化
## 首页头部优化
![头部header](/images/hexo-5/header.png)
1. 修改网站小图标、博客logo及文字描述，修改hexo(F:\Blog\hexo\_config.yml)主配置文件_config.yml中的
```bash
# Site 网站
title: troylc博客                   #网站标题
subtitle: 爱生活爱编程，一起来进步吧！  #网站副标题
description:  hello,every body!         #网站描述
author: troylc                          #您的名字 
language: zh-CN                         #网站使用的语言
timezone:                               #网站时区。Hexo 默认使用您电脑的时区
```
2. 修改jacman(F:\Blog\hexo\themes\jacman\_config.yml)主题目录下的配置文件_config.yml,找到以下配置,指定favicon，imglogo的图片位置，或者把原本对应位置下的图片名称不变，直接换成自己喜欢的图片
```bash
#### Image
imglogo:
  enable: true             ## display image logo true/false.
  src: img/logo.png        ## `.svg` and `.png` are recommended,please put image into the theme folder `/jacman/source/img`.
favicon: img/favicon.ico   ## size:32px*32px,`.ico` is recommended,please put image into the theme folder `/jacman/source/img`.     
apple_icon: img/jacman.jpg ## size:114px*114px,please put image into the theme folder `/jacman/source/img`.
author_img: img/author.jpg ## size:220px*220px.display author avatar picture.if don't want to display,please don't set this.
banner_img: #img/banner.jpg ## size:1920px*200px+. Banner Picture
### Theme Color 
theme_color:
    theme: '#2ca6cb'    ##the defaut theme color is blue
```

# 参考：




---
