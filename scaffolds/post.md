---
layout:  #文章布局 ---------#默认文章的模板-hexo new "title" 就会以这个模板创建文
title: {{ title }} #文章标题
date: {{ date }} #时间，一般不用改
updated: {{ date }}  # 更新时间
comments:  #是否开启评论，默认为true
categories:  #目录分类
tags:  #标签，格式可以是[Hexo,总结]，中间用英文逗号分开 hexo new "text"  报错YAMLException: duplicated mapping key at line 6, column 1: 因为模板中有两个tags:
permalink:  #文章永久链接，一般不用写，默认就行 
---
