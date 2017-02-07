---
title: jenkins-registry持续集成-jenkins管理与registry集成打包上传镜像(二)
toc: true
date: 2017-02-06 19:18:23 Sunday
updated: 2017-02-06 19:49:34 Sunday
categories: [docker]
tags: [linux,docker,jenkins,registry]

---

通过上一篇 [jenkins-registry持续集成-jenkins-registry安装与数据迁移(一)](http://www.troylc.cc/docker/2017/01/08/Docker05registry-jenkins.html)，已经把jenkins和registry的环境准备好了。接下来可以通过jenkins来操作怎么做到从版本库中的代码下载->编译->打包docker镜像->上传到docker-registry私有仓库，然后就可以在其它有docker的环境中直接pull打包好的docker镜像。

### 初始化jenkins及安装插件
启动完jenkins后通过浏览器输入地址http://部署jenkins主机IP:端口  
![jenkins初始化界面2-1](/images/docker/jenkins/jenkins2-1.png)  
根据提示从输入administrator password 或者可以通过启动日志查看这个password 如：  
![jenkins初始化界面2-2](/images/docker/jenkins/jenkins2-2.png)  
选择安装插件方式，这里我是默认第一个  
![jenkins初始化界面2-3](/images/docker/jenkins/jenkins2-3.png)  
进入插件安装界面，连网等待插件安装  
![jenkins初始化界面2-4](/images/docker/jenkins/jenkins2-4.png)  
安装完插件后，进入创建管理员界面  
![jenkins初始化界面2-5](/images/docker/jenkins/jenkins2-5.png)  
输入完管理员账号后，点击continue as admin 进入管理界面点击系统管理-插件管理中安装docker构建插件和角色管理插件  
![jenkins初始化界面2-6](/images/docker/jenkins/jenkins2-6.png)  
安装docker构建插件，在可选插件中查找docker build step plugin  
![jenkins初始化界面2-7](/images/docker/jenkins/jenkins2-7.png)   
安装角色管理插件，在可选插件中查找Role-based Authorization Strategy  
![jenkins初始化界面2-8](/images/docker/jenkins/jenkins2-8.png) 

如果在初始化过程中，有些jenkins推荐的插件安装失败，也可以在可选插件中查找再次安装，选择好要安装的插件后，可以选择直接安装和下载后重启安装，我这里下载后重启安装，重启安装完后，之前出现的安装失败的提示就没有了。

### 配置jenkins相关权限及属性

#### 配置maven安装路径和docker remote API接口地址  
点击系统管理->Global Tool Configuration->找到maven点击新增按钮  
![配置jenkins相关权限及属性2-9](/images/docker/jenkins/jenkins2-9.png)   
点击系统管理->系统配置->找到docker builder->在Docker URL栏中输入docker远程访问的地址，如：  
![配置jenkins相关权限及属性2-10](/images/docker/jenkins/jenkins2-10.png)   
设置docker主机可以被远程访问，[请参考：docker系列(二)使用Docker-Remote-API](http://www.troylc.cc/docker/2016/07/31/docker-02.html)

#### 配置用户角色及权限  
1. 选择系统管理->Configuration Global Security->进入选择启用安全：
TCP port for JNLP agents ->禁用，访问控制-安全域->jenkins专有用户数据库，访问控制-授权策略->Role-Based Strategy 如：  
![配置jenkins相关权限及属性2-11](/images/docker/jenkins/jenkins2-11.png)   

2. 选择系统管理->Manage and Assign Roles->Manage Roles：  
- 添加Global Roles(admin、member、ops、others)，  
设置全局角色（全局角色可以对jenkins系统进行设置与项目的操作）  
admin:对整个jenkins都可以进行操作  
ops:可以对所有的job进行管理  
other/member:只有读的权限   
- 添加project Roles(dmp-manager、dmp-view、tsc-manager、tsc-view)并且给添加的角色分配如下权限  
![配置jenkins相关权限及属性2-12](/images/docker/jenkins/jenkins2-12.png)   
**- 注意**：在添加project Roles时，如果想让不同的用户看到不同的job,必须设置Pattern,如上dmp_manager角色就只能查看以dmp开头的job,Pattern规则必须是“dmp.*”，注意是以“.*”结尾的匹配规则，tsc亦是如此。

3. 选择系统管理->管理用户:新建几个管理员用户如：dmpadmin、tscadmin  
![配置jenkins相关权限及属性2-13](/images/docker/jenkins/jenkins2-13.png) 

4. 选择系统管理->Manage and Assign Roles->Assign Relos:把第三步的用户加到user/group中并授于对应的角色权限 如：  
![配置jenkins相关权限及属性2-14](/images/docker/jenkins/jenkins2-14.png) 

### 创建-编译-打包-上传docker镜像任务
进入主界面->新建->输入job名称->选择构建一个maven项目->点击OK->进入项目配置如下图：
![jenkins创建-编译-打包-上传2-15](/images/docker/jenkins/jenkins2-15.png)   
点击保存->选择立即构建->jenkins进入构建流量->build histroy中的时间，可以查看构建过程及错误信息，如果构建一次不成功可以通过日志查看错误原因，来调整job的配置内容。

### 参考
[Jenkins使用教程之用户权限管理（包含插件的安装）](http://www.jianshu.com/p/7e148bcfb96e)  
[Docker学习笔记：Java、Maven、Jenkins Docker插件持续集成简单部署](http://shaofan.org/docker-jenkins/)