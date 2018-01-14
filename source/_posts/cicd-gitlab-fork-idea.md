---
title: 持续集成(CI)与持续交付(CD)之gitlab-Fork工作流与IntellijIDEA操作
toc: true   // 在文章侧边增加文章目录
date: 01/08/2018 5:26:15 PM 
updated: 01/08/2018 5:26:20 PM 
categories: [CI/CD,gitlab,IntelliJ IDEA]
tags: [gitlab,docker,linux,IntelliJIDEA]
---

承接上一篇，gitlab在本地部署完后，我们就可以像拥抱github一下来体验一下gitlab的精妙之处，在git中我们针对代码的版本管理，有很多种玩法，如有集中式工作流，功能分支工作流，gitflow工作流，git-fork工作流。不同的玩法，有不同的操作；前三种都是基于集中式代码管理来进行操作的，有点类似于SVN的方式，只不过git的集中式代码管理还是一种分布式的管理，针对开分支，合并分支等相关操作，要比SVN方便很多。
本文主要讲上面提的到第四种gitlab的fork工作流，目前最流行的github中开源团队协作的方式就是这种。Forking工作流的一个主要优势是，贡献的代码可以被集成，而不需要所有人都能push代码到仅有的中央仓库中。 开发者push到自己的服务端仓库，而只有项目维护者才能push到正式仓库。 这样项目维护者可以接受任何开发者的提交，但无需给他正式代码库的写权限。
在安装完git bash工具后,本文主要是以文档的方式做操作，而且主要工具是Intellij idea来开发。所以以下的操作都是在IDEA中进行。


### fork工作流程步骤
FORK远程中央仓库-->
clone从远程中央仓库fork到远程个人仓库到本地git环境-->
管理员在web界面创建功能与BUG任务issues-->
开发人员查看远程中央仓库的issues并针对功能和BUG issues在本地新建分支并开发-->
提交并push分支到个人仓库-->
个人仓库中发送meger request请求，合并个人仓库分支到远程中央仓库的master-->
远程中央仓库的代码维护者进行合并分支-->
个人本地仓库删除已经合并的分支，并从远程中央仓库中pull代码到本地master中-->
本地master push到个人仓库中。
#### FORK远程中央仓库到个人仓库中
用用户名密码登录我们上一章节搭建的gitlab服务web端程序。找到想要fork的项目，点击fork,如下图：
![](/images/cicd/git-idea/15155043228652.jpg)
![](/images/cicd/git-idea/15155043776641.jpg)
![](/images/cicd/git-idea/15155047092481.jpg)
以上是fork远程仓库的步骤。

#### clone个人仓库中fork过来的仓库
复制SSH链接地址，到intellij idea进行clone操作，如下：
![](/images/cicd/git-idea/15155051645082.jpg)
![](/images/cicd/git-idea/15155052697020.jpg)
然后按照IDEA中一步一步操作，就可以把项目clone到本地

#### 远程仓库创建功能及BUG问题的任务
（如果是公司内部用，最好是项目管理者来创建BUG和功能issues，这里我们是想把issues来当做任务与BUG功能的小管理工具来使用）创建功能及BUG问题的任务，以便于在本地开发的时候，新建对应的分支进行开发，在web端的操作如下：
![](/images/cicd/git-idea/15155058237183.jpg)
![](/images/cicd/git-idea/15155063602455.jpg)
![](/images/cicd/git-idea/15155064678746.jpg)

#### 开发人员修改并提交meger request
本地开发人员针对这个以上问题在本地刚刚用idea clone下来的项目，新建一个分支，进行修改
![](/images/cicd/git-idea/15155066713100.jpg)
![](/images/cicd/git-idea/15155067118353.jpg)
![](/images/cicd/git-idea/15155067851203.jpg)
将修改提交本地git环境
![](/images/cicd/git-idea/15155068606051.jpg)
将修改提交到远程个人仓库
![](/images/cicd/git-idea/15155069060448.jpg)
![](/images/cicd/git-idea/15155069797246.jpg)
在登录web上查看个人仓库下是否有push过来的update-a-module分支
![](/images/cicd/git-idea/15155070670436.jpg)
切换分支，并在左侧菜单栏下的meger request中新建一个meger个人update-a-module分支与远程fork的中央仓库的合并请求
![](/images/cicd/git-idea/15155073536145.jpg)
![](/images/cicd/git-idea/15155074565530.jpg)
![](/images/cicd/git-idea/15155082934666.jpg)
![](/images/cicd/git-idea/15155076458896.jpg)

远程中央仓库管理员，查看meger request请求，检查并合并，用管理员的身份登录gitlab，web界面。
![](/images/cicd/git-idea/15155078116367.jpg)
打开，并点击合并，合并后会删除原分支
![](/images/cicd/git-idea/15155083660048.jpg)
![](/images/cicd/git-idea/15155084344884.jpg)
![](/images/cicd/git-idea/15155084741351.jpg)
新建的问题和功能任务也自动关闭了
![](/images/cicd/git-idea/15155085956790.jpg)

#### 从个人本地仓库更新远程中央仓库
个人本地仓库在idea中更新远程master分支，然后push到人个仓库的master分支，以做到更新远程仓库中的所有修改。
![](/images/cicd/git-idea/15155470922065.jpg)
![](/images/cicd/git-idea/15155473821411.jpg)
![](/images/cicd/git-idea/15155514907571.jpg)
![](/images/cicd/git-idea/15155515422837.jpg)
本地个人仓库已经和远程中央仓库同步了，现在把本地的更新推到个人远程仓库中
![](/images/cicd/git-idea/15155520138332.jpg)
![](/images/cicd/git-idea/15155520322469.jpg)
到web端查看个人仓库下的mater分支的代码，可以看到和远程仓库一样了
![](/images/cicd/git-idea/15155627363690.jpg)
至此一个完成的fork-git工作流已经基本操作完成，下面我们讲讲fork工作流下的冲突怎么解决

### git冲突在idea中的操作
冲突主要是两人同时操作同一个文件的同一行，比如用A用户操作一下a-module.md这个文件第二行，B用户也操作了a-module.md这个文件第二行
A用户操作：
本地先开分支，在分支上做修改操作
![](/images/cicd/git-idea/15155632305526.jpg)
提交到本地
![](/images/cicd/git-idea/15155632787905.jpg)
push到远程个人仓库
![](/images/cicd/git-idea/15155634089430.jpg)

![](/images/cicd/git-idea/15155637661113.jpg)
![](/images/cicd/git-idea/15155638141886.jpg)
一步一步住下操作，因为此处A用户来此仓库的管理员，所以A用户可以直接meger
![](/images/cicd/git-idea/15155641181541.jpg)
![](/images/cicd/git-idea/15155640795951.jpg)

B用户登录开新分支，并修改同一文件的同一行操作，
![](/images/cicd/git-idea/15156399438941.jpg)
![](/images/cicd/git-idea/15156401255810.jpg)
![](/images/cicd/git-idea/15156401696299.jpg)
![](/images/cicd/git-idea/15156402109597.jpg)

提交成功后，在web界面发送meger request请求远程中央仓库管理中A用户来合并，这时合并，就会提示冲突，修改B用户先解决冲突后再提交
![](/images/cicd/git-idea/15156403774357.jpg)
![](/images/cicd/git-idea/15156404351545.jpg)
![](/images/cicd/git-idea/15156404735390.jpg)
![](/images/cicd/git-idea/15156406262524.jpg)
远程仓库管理员(A用户)这时就有可以先跟提交在这个合并请求的讨论区进行讨论，让A用户自己先解决冲突后再push分支上来，然后本次提交会自动打开meger request按钮，如下是A用户查看请求合并的界面
![](/images/cicd/git-idea/15156409283782.jpg)
![](/images/cicd/git-idea/15156409577421.jpg)

然后B用户在本地先pull，远程中央仓库的最新版本，和本地的update-b-5分支进行合并操作
![](/images/cicd/git-idea/15156412763743.jpg)
![](/images/cicd/git-idea/15156413012578.jpg)
在pull远程仓库的更新时，在idea中会自动弹出冲突界面
![](/images/cicd/git-idea/15156414180679.jpg)
![](/images/cicd/git-idea/15156415094759.jpg)
![](/images/cicd/git-idea/15156415898700.jpg)
push把本地update-b-5的分支到远程个人仓库，然后远程个人仓库会自动把之前提交到远程中央仓库的的合并请求的meger按钮打开，仓库管理就可以看到
![](/images/cicd/git-idea/15156417663811.jpg)
到web界面，给远程中央仓库管理员发一个讨论，告诉管理员冲突已经解决了
![](/images/cicd/git-idea/15156423668278.jpg)
远程中央仓库管理员A用户登录后就看到之前提交的meger request的按钮已经可以点击了，如下：
![](/images/cicd/git-idea/15156425231210.jpg)
![](/images/cicd/git-idea/15156426775275.jpg)/images/cicd/git-idea/
至此有关冲突的操作，基本上结束，后续的操作和上面正常操作方式一样，需要B用户把远程中央仓库的更新拉取到本地master分支，让本地master分支和远程仓库保持一致，然后push到远程的个人仓库的master分支，后再开发新功能都是基于master分支，再来开新分支进行功能开发，代码提交等。

文章写得比较繁琐，需要细看，如有问题请留言，或者发邮件troylc@163.com，一起交流学习。谢谢。


