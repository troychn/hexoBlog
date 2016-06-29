---
title: Hexo教程(七)-hexo-利用git多PC间同步博客 #可以改成中文的，如“新文章”
date: 6/29/2016 10:22:16 AM   #发表日期，一般不改动
updated: 6/29/2016 10:22:22 AM  
categories: hexo #文章文类
tags: [hexo] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog
---
# 前言：
如果是两PC，比如单位和家，同时都想更新blog。而由于hexo没有动态的后台，而且全部文件都在本地生成，所以如果在A电脑上发表了A1文章后，在B电脑上又写了篇B1文章，在B电脑上上传后你会发现只有B1文章而A1文章没了（因为B电脑上没有A1文章的md文件），所以多台电脑同时用来写文章的时候，需要解决文章同步问题。
# 正文
下面介绍的是如何利用第三方代码托管平台进行同步，以github为例：
## 准备工作
在A电脑和B电脑上都安装git以及ssh密钥配置与连接，如果不会，可参考[GIT安装](http://www.troylc.cc/hexo/2016/05/31/Hexo-1.html#%E5%AE%89%E8%A3%85Git "GIT安装")和[配置SSH密钥](http://www.troylc.cc/hexo/2016/05/31/Hexo-1.html#%E9%85%8D%E7%BD%AESSH%E5%AF%86%E9%92%A5 "配置SSH密钥")
## 同步最新的blog到github
建议先将拥有最新blog相关的文件的电脑上的文件上传到github上，否则在另一台电脑上下载时会有版本冲突，解决也比较麻烦。一般建议blog静态文件和blog源码文件分库存放，在PC上建立git ssh密钥连接和建立新库respo在此略过-参考：[注册Github账户创建代码库](http://www.troylc.cc/hexo/2016/05/31/Hexo-1.html#%E6%B3%A8%E5%86%8CGithub%E8%B4%A6%E6%88%B7%E5%88%9B%E5%BB%BA%E4%BB%A3%E7%A0%81%E5%BA%93 "注册Github账户创建代码库")
- 编辑.gitignore文件：.gitignore文件作用是声明不被git记录的文件，blog根目录下的.gitignore是hexo初始化是创建的，可以直接编辑，建议.gitignore文件包括以下内容：
public内的文件可以根据source文件夹内容自动生成的，不需要备份。其他日志、压缩、数据库等文件也都是调试等使用，也不需要备份。
```bash
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```
- 初始化仓库：
```bash
git init    
git remote add origin <server>
```
**server**是仓库的在线目录地址，可以从github上直接复制过来，origin是本地分支，remote add会将本地仓库映射到托管服务器的github仓库上。
- 添加本地文件到仓库并同步到git上：
```bash
git add . #添加blog目录下所有文件，注意有个'.'(.gitignore里面声明的文件不在此内)    
git commit -m "hexo source first add" #添加更新说明    
git push -u origin master  #将本地origin分支推送更新到github上的master分支 
```
至此，github库上已完成最新blog的同步上传。
## 将github库的内容同步到另一台电脑
之前已经将最新的blog源码内容同步到了github仓库上，现在另一台电脑准备同步源码内容。注意，在同步前也要先建好[hexo的环境](http://www.troylc.cc/hexo/2016/05/31/Hexo-1.html#hexo%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85 "hexo环境安装")，不然把blog同步到本地后，无法运行hexo服务。在建好的环境的主目录(F:\Blog\hexo\)运行以下命令：
```bash
git init  #将目录添加到版本控制系统中    
git remote add origin <server>  #同上    
git fetch --all  #将git上所有文件拉取到本地    
git reset --hard origin/master  #强制将本地内容指向刚刚同步git云端内容
```
**reset**对所拉取的文件不做任何处理，不用**pull**是因为本地尚有许多文件，使用pull会有一些版本冲突，解决起来也麻烦，而本地的文件都是初始化生成的文件，较github库里面的blog文件而言基本无用，所以可以直接丢弃覆盖。
至此在另一台电脑上也同步了最新的blog文件。
## 实现博客在多电脑之间同步
假设上面的操作做完后，你相当于在公司电脑和家里电脑上，都拥有了最新blog文件，现在需要在不同的电脑间，更新同步blog，以实现多电脑之间blog同步：
比如我在这家里的电脑上创建了一篇新的文章C1，编写了一些内容，还没有编写完，如果想在单位电脑上
上继续在C1的基础上写，我们必需先在家里电脑上把C1上传到github上，运行以下命令，提交文章：
```bash
git add .   #将所有更新的本地文件添加到版本控制系统中
git status  #查看本地文件的状态。然后对更改添加说明更推送到git托管库上
git commit -m '更新信息说明'
git push  #把本地blog文件同步到github对应的库上。
```
至此，家里电脑的blog最新的更新同步完成。
如果在公司电脑上再次对C1进行编辑，需先运行:
```bash
git pull #下载同步github上的文件到本地。
```
然后就可以在本地找到C1这篇文章了，至此多台电脑间同步博客内容就已经完成了
# 参考：
[利用git解决hexo博客多PC间同步问题](http://rainnie.me/2016/03/13/利用git-解决hexo博客多PC-间同步问题/)


---
