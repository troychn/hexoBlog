---
title: Hexo教程(一)-hexo搭建 #可以改成中文的，如“新文章”
date: 2016-05-31 13:50:23 #发表日期，一般不改动
updated: 6/2/2016 10:18:20 AM 
categories: hexo #文章文类
tags: [hexo] #文章标签，多于一项时用这种格式，只有一项时使用tags: blog

---

## hexo-Github Pages静态博客
### 安装Node.js
在 Windows7 64 环境下安装 Node.js 非常简单，仅须到[官网](https://nodejs.org/en/download/ "nodejs官网")下载安装文件并执行即可完成安装
![nodejs官网下载](/images/hexo-1/nodejs.jpg)
### 安装Git
从[git官网](https://git-scm.com/download "git下载")下载git并执行即可完成安装。[安装过程](http://jingyan.baidu.com/article/90895e0fb3495f64ed6b0b50.html "windows安装过程")。安装完后，右击鼠标选择 git bash here
![git窗口](/images/hexo-1/git.png)
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
  

### hexo环境安装
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

#### Hexo安装
桌面右键鼠标，点击Git Bash Here，输入npm命令即可安装
```shell    
npm install hexo-cli -g
npm install hexo --save
#如果命令无法运行，可以尝试更换taobao的npm源，[http://npm.taobao.org](http://npm.taobao.org "淘宝npm源说明") ,请进入查看相关说明
npm install -g cnpm --registry=https://registry.npm.taobao.org
```
#### Hexo初始化配置
安装完成后，根据自己喜好建立目录（如F:\Blog\Hexo），直接进入F:\Blog\Hexo文件夹下右键鼠标，点击Git Bash Here，进入Git命令框，执行以下操作。
```shell
$ hexo init
$ npm install
```
安装 Hexo 完成后，Hexo 将会在指定文件夹中新建所需要的文件。Hexo文件夹下的目录如下：
![hexo初始化目录](/images/hexo-1/hexo-dir.png)
下面依次介绍上面各个文件或者目录的用途：
- _config.yml 站点配置文件，很多全局配置都在这个文件中。
- package.json 应用数据。从它可以看出hexo版本信息，以及它所默认或者说依赖的一些组件。
- scaffolds 模版文件。当你创建一篇新的文章时，hexo会依据模版文件进行创建，主要用在你想在每篇文章都添加一些共性的内容的情况下。
- source 这个文件夹就是放文章的地方了，除了文章还有一些主要的资源，比如文章里的图片，文件等等东西。这个文件夹最好定期做一个备份，丢了它，整个站点就废了。
- themes 主题文件夹。

#### 安装Hexo插件
如果想不出错，就将下面的插件都安装完。（如果用淘宝源，请把npm换成cnpm）
```shell
npm install hexo-generator-index --save
npm install hexo-generator-archive --save
npm install hexo-generator-category --save
npm install hexo-generator-tag --save
npm install hexo-server --save
npm install hexo-deployer-git --save
npm install hexo-deployer-heroku --save
npm install hexo-deployer-rsync --save
npm install hexo-deployer-openshift --save
npm install hexo-renderer-marked@0.2 --save
npm install hexo-renderer-stylus@0.2 --save
npm install hexo-generator-feed@1 --save
npm install hexo-generator-sitemap@1 --save
```
#### 本地查看效果
执行下面语句，执行完即可登录localhost:4000查看效果
```
hexo generate
hexo server
```
登录localhost:4000，即可看到本地的效果如下：
![hexo初始化目录](/images/hexo-1/hexo-view.png)

### 将博客部署Github Pages上
本地的博客已经搭建起来了，但是目前只可以通过本地服务查看我们的博客。现在需要做的就是把本地的博客发布到Github服务器上，通过Github Pages这个功能让别人也可以访问我们的博客，而Github Pages就帮我完成了这件事情。但是Github Pages的代码就是寄存在Github上面的。所以需要在Github上面创建一个新的项目。
#### 注册Github账户创建代码库
1. 访问[Github](http://www.github.com/)首页
2. 点击右上角的 Sign Up，注册自己的账户
3. 注册完登陆后，我们就创建一个我们自己的Github Pages项目。点击New repository.创建要点如下：
![github reposity](/images/hexo-1/github-reposity.png)

#### 配置SSH密钥
1. 看看是否存在SSH密钥(keys)
首先，我们需要看看是否看看本机是否存在SSH keys,打开Git Bash,并运行:
```
$ cd ~/. ssh
```
检查你本机用户home目录下是否存在.ssh目录
如果，不存在此目录，则进行第二步操作，否则，你本机已经存在ssh公钥和私钥，可以略过第二步，直接进入第三步操作。
2. 创建一对新的SSH密钥(keys)
```shell
#输入 ssh-keygen -t rsa -C "xxxxx@126.com" 这将按照你提供的邮箱地址，创建一对密钥
$ ssh-keygen -t rsa -C "xxxxx@126.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/Administrator/.ssh/id_rsa): 回车，则将密钥按默认文件进行存储 c/Users/you/.ssh/github_rsa
Created directory '/c/Users/Administrator/.ssh'.
Enter passphrase (empty for no passphrase): 输入密码，
Enter same passphrase again: 确认密码
Your identification has been saved in /c/Users/Administrator/.ssh/id_rsa.
Your public key has been saved in /c/Users/Administrator/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:j9GCdXhgo+Up1af6XB/E9yZDv8XyZsZNXIA917q/72U xxxxx@126.com
The key's randomart image is:
+---[RSA 2048]----+
|        *.   o  .|
|       * =. o + o|
|      o = oo . = |
|       + +.   = o|
|      . S..  o *o|
|        .=  . * O|
|        .o.. . @E|
|          o   .oX|
|               *=|
+----[SHA256]-----+
```
3. 在GitHub账户中添加你的公钥
直接用文本打开当前用户目录下的.ssh/id_rsa.pub文件，复制文件中的内容
登陆GitHub,进入你的Account Settings.
![github settings](/images/hexo-1/github-settings.png)
选择New SSH Keys
![github sshkey](/images/hexo-1/github-sshkey.png)
粘贴密钥，添加即可
![github sshkey](/images/hexo-1/ithub-SSH-OK.png)
4. 测试SSH密钥
可以输入下面的命令，看看设置是否成功，git@github.com的部分不要修改：
```shell
$ ssh -T git@github.com
```
如果是下面的反馈，表示成功：
```shell
$ ssh -T git@github.com
The authenticity of host 'github.com (192.30.252.123)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no)? yes  不要紧张，输入yes就好，然后会看到：
Warning: Permanently added 'github.com,192.30.252.123' (RSA) to the list of known hosts.
Enter passphrase for key '/c/Users/Administrator/.ssh/id_rsa': 输入上面的创建SSHKEY的密码
Hi troychn! You've successfully authenticated, but GitHub does not provide shell access.
```
如果出现以下问题，没有权限
```shell
$ ssh -T git@github.com
The authenticity of host 'github.com (192.30.252.120)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,192.30.252.120' (RSA) to the list of known hosts.
**Permission denied (publickey).**
```
解决办法：
重新执行:第二步、创建一对新的SSH密钥(keys) 最好是输入密码,然后一步一步往下走，直至测试这步
如还有问题，请参考：
GitHub Help - [Generating SSH Keys](https://help.github.com/articles/generating-an-ssh-key/ "Generating an SSH key")
Error: [Permission denied (publickey)](https://help.github.com/articles/error-permission-denied-publickey/ "Permission denied (publickey)")

#### 将本地Hexo文件更新到Github库中
登录Github打开上面创建的的项目 username.github.io 复制地址
![github sshkey](/images/hexo-1/cp-githubaddr.png)
打开你一开始创建的Hexo文件夹（如F:\Blog\Hexo），用记事本打开刚文件夹下的_config.yml文件
![github sshkey](/images/hexo-1/git-gitml_4.png)
在配置文件里作如下修改，保存
![github sshkey](/images/hexo-1/cp-githubaddr-2.png)
在Hexo文件夹下执行，右击打开git bash here 执行以下命令 ：

```
hexo g -d
```

执行完之后会让你输入github的账号和密码，输入完后就可以登录我们自己的部署在Github Pages服务器上的博客了。对应的地址是 username.github.io。
![github sshkey](/images/hexo-1/username-passw.png)
在浏览器上输入自己的主页地址 （https://username.github.io/）username为你的github账号
在浏览器上输入即可看到我们自己的博客，别人电脑输入也可以哦。
![github sshkey](/images/hexo-1/view-cg.png)

到此利用github Pages 搭建hexo静态博客就成功了，下一篇准备美化我们的博客。

 





---

