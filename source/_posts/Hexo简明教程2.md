---
title: Hexo -简明教程(一)-环境安装 #可以改成中文的，如“新文章”
date: 2016-05-31 13:50:23 #发表日期，一般不改动
categories: hexo #文章文类
tags: [hexo] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog

---

# hexo静态博客环境安装
## 安装Node.js
在 Windows7 64 环境下安装 Node.js 非常简单，仅须到[官网](https://nodejs.org/en/download/ "nodejs官网")下载安装文件并执行即可完成安装
![nodejs官网下载](/images/nodejs.jpg)
## 安装Git
下载 msysgit 并执行即可完成安装。(上官网要翻墙，如果你的是64位，可以点击此处下载)
怎么打开Git？安装完后，右击鼠标选择 git bash here
![git容器](/images/git.png)
- Git教程：[Pro Git（中文版）](http://git.oschina.net/progit/ "git教程")
- git基本操作：

命令 | 说明
---|---
git init |  git init 在目录中创建新的 Git 仓库。 你可以在任何时候、任何目录中这么做，完全是本地化的
git clone | 使用 git clone 拷贝一个 Git 仓库到本地，让自己能够查看该项目，或者进行修改。
git add | git add 命令可将该文件添加到缓存
git status | git status 以查看在你上次提交之后是否有修改。加 -s 参数，以获得简短的结果输出。
git commit | 执行 git commit 将缓存区内容添加到本地仓库中
git push | 推送本地仓库的更新到远程仓库，语法git push [远程仓库名][本地分支][远程分支]
git pull | 抓取远程仓库所有分支更新并合并到本地仓库  
  

##hexo环境安装
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

### Hexo安装
输入npm命令即可安装 如在你喜爱的文件夹下（如E:\Hexo），执行以下指令(在E:\Hexo内点击鼠标右键，选择Git Bash)，Hexo 即会自动在目标文件夹建立网站所需要的所有文件。




---

