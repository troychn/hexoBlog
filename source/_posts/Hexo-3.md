---
title: Hexo教程(三)-hexo博客同时托管到github和coding #可以改成中文的，如“新文章”
date: 2016-06-15 22:09:23 #发表日期，一般不改动
updated: 2016-06-23 13:09:23
categories: hexo #文章文类
tags: [hexo,hexo优化] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---
# 前言：
之前已经把hexo部署到github，但是有时候挺慢的，于是就像跟大家一样搞到国内的coding.net上，申请coding的邮箱尽量与github使用同一个邮箱，这样配置ssh就省了事情，不用配置两个ssh了
# 正文
## coding配置
基本流程跟github一样，都是先申请帐号。这里需要说一下的是，申请coding的邮箱尽量与github使用同一个邮箱，这样配置ssh就省了事情，不用配置两个ssh了。
![coding网站](/images/hexo-4/coding.png)
ssh对接基本和github一样，如果之前生成过ssh的话，就可以直接使用，一般存放在~/.ssh/id_rsa.pub。如果没有的话，那就下面几步就可以了,参考：[Hexo教程(一)-hexo环境搭建](http://www.troylc.cc/hexo/2016/05/31/Hexo%E6%95%99%E7%A8%8B2.html#%E9%85%8D%E7%BD%AESSH%E5%AF%86%E9%92%A5)
```bash
$ ssh-keygen -t rsa -b4096 -C "你的邮箱"
```
![coding网站](/images/hexo-4/coding-ssh.png)
**新建项目**
直接新建一个coding项目，需要注意就是：项目名尽量与用户名一样，这样可以省去后续的一些麻烦.设置为公有的项目，其余的都不用动，直接默认就行了。
## hexo配置
新建好了coding的项目之后，修改一下站点的根目录配置文件_config.yml,找到deploy位置，修改如下:
```bash
deploy:
  type: git
  repository: 
      github: git@github.com:xxxx/xxxxx.github.io.git
      coding: git@git.coding.net:xxxx/xxxxx.git
  branch: master
```
如果都是master分支，所以两个写到一起了，如果不是同一个分支的话，可以如下的写法：
```bash
deploy:
  type: git
  repository:
      github: git@github.com:xxxx/xxxxx.github.io.git,分支名称
      coding: git@git.coding.net:xxxx/xxxxx.git,分支名称
```
其实就是在原来的基础再加一个就好了。只需要把repo的地址改成自己对应项目的就好。
还有一步，就是在博客的source/目录下需要创建一个空白文件,至于原因，是因为 coding.net需要这个文件来作为以静态文件部署的标志。就是说看到这个Staticfile就知道按照静态文件来发布。
```bash
$ cd source
$ touch Staticfile  #名字必须是Staticfile 
```
现在可以deploy了:
```bash
$ hexo clean
$ hexo d -g
```
会看到有两个发布。发布完成之后，还需要托管coding.net上。

**coding使用Pages部署**
这种方式的话，可以绑定自己的域名。就跟Github Pages一样。项目名字必须与coding的用户名一致!!!只需要在如下:
![coding网站](/images/hexo-4/coding-pages.png)
因为前面配置的分支是master,因此开启之后，也需要是master。然后看起之后就可访问了,下面两个链接都可以访问
`http://troylc.coding.me/troylc`
`http://troylc.coding.me/`
## 绑定域名：
在万网上面购买了troylc.cc域名，三年只要50多块，个人用的就不用com这种超级贵的域名了。现在要实现国内的走coding，海外的走github，只要配置2个CNAME就行(详细参考：[万网](https://wanwang.aliyun.com/ "购买万网域名")，这里就不做详细说明了).域名解析如下：
![coding网站](/images/hexo-4/wanwang.png)
**coding-pages绑定域名：**  
![coding网站](/images/hexo-4/coding-ww.png)
**github-pages绑定域名：**  
在hexo根目录下的source/目录下需要创建一个CNAME,内容为：troylc.cc
![github-CNAME](/images/hexo-4/github-CNAME.png)
然后在hexo根目录下执行：
```bash
$ hexo clean
$ hexo d -g
```
过十几分钟后检测troylc.cc看到的解析是正确的，国内解析到Coding，国外解析到Github

# 参考：
[hexo同时部署到coding(gitcafe)和github](http://shomy.top/2016/03/03/hexo-in-coding-github/)

---

