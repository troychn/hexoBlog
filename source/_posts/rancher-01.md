---
title: RANCHER容器平台管理(一)-环境构建与配置
toc: true   // 在文章侧边增加文章目录
date: 6/2/2017 5:26:15 PM 
updated: 6/2/2017 5:26:20 PM 
categories: [rancher]
tags: [linux,rancher,docker,swarm]

---

rancher是什么，它能为我们做些什么？rancher是一个开源的软件平台，使企业能够在生产环境中运行容器。使用Rancher，我们不再需要使用不同的开源技术从头开始构建容器服务平台。Rancher提供管理生产环境中的容器所需的整个软件堆栈。  

如下为rancher的功能图：  
![rancher功能图](/images/rancher/rancher-base/1.png)

## 安装Rancher服务器  
Rancher被部署为一组Docker容器。运行的牧场主是简单的启动两个容器。一个容器作为管理服务器，另一个容器在节点上作为代理。

**运行Rancher的要求：**  
- 任何具有受支持版本的Docker的现代Linux发行版。如：RancherOS，Ubuntu，RHEL / CentOS 7，这些都进行了更严格的测试。
  -  对于RHEL / CentOS，Docker不推荐使用默认存储驱动程序，即使用环回的devicemapper 。请参考Docker文档，了解如何更改。
  -  对于RHEL / CentOS，如果要启用SELinux，则需要安装其他SELinux模块。
- 1GB RAM
- MySQL服务器应该有一个max_connections设置> 150
  - MYSQL配置要求
    - 选项1：用Antelope文件格式运行，默认值为 COMPACT
    - 选项2：使用Barracuda运行MySQL5.7，默认ROW_FORMAT值为Dynamic  

**rancher目前的版本情况：**  
Rancher服务器端的容器有2个不同的版本。

- rancher/server:latest标签将是我们最新的开发版本。这些构建将通过我们的CI自动化框架进行验证。这些版本不适用于部署在生产中。
- rancher/server:stable标签将是我们最新的稳定发布版本。此标签是我们推荐用于生产的版本。  

请不要使用任何带有rc{n}后缀的版本。这些rc构建意味着Rancher团队测试构建。

### 安装一个单容器rancher service
- 运行一个单容器的rancher service，  
  如果要将容器内的数据库保存到主机上的卷上，请通过绑定MySQL卷来启动Rancher服务器  
  ```bash
  $ docker run -d -v <宿主机目录>:/var/lib/mysql --restart=unless-stopped -p 8080:8080 rancher/server:stable
  ```

### 用户界面  
UI和API将在暴露端口上可用8080。在Docker镜像下载完成之后，Rancher成功启动后可能需要一两分钟的时间才能查看。
导航到以下网址：http://<SERVER_IP>:8080。该<SERVER_IP>是运行牧场主服务器主机的公网IP地址。

一旦UI启动并运行，您可以通过添加主机或从基础架构目录中选择一个容器编排。默认情况下，如果不选择不同的容器编排类型，环境将使用rancher默认的编排模型Cattle。将主机添加到Rancher之后，您可以从Rancher目录开始添加服务或启动模板。  

用浏览器打开http://hostsname:8080/，如果你看到如下页面，则说明你的Rancher Server搭建成功了  

![rancher](/images/rancher/rancher-base/2.png)  

## RANCHER管理
### 设置权限访问  
第一次启动rancher后，它本身是没有给权限控制，所有人都可以访问，并且访问的权限是一样的。在RANCHER UI中我们可以看到系统管理旁边有一个红色的“！”，其实际标识当前的RANCHER是没有权限控制的，需要我们在系统管理下的二级菜单中选择访问控制，添加相应的管理用户后来启动访问权限的控制，Rancher支持多种权限控制方案，分别是：Active Directory、Azure AD、GitHub、Local Auth、OpenLDAP和SHIBBOLETH。我们这里选择简单的本地数据权限local Auth,即设置一个用户名密码，然后启动本地权限控制。   
![访问权限](/images/rancher/rancher-base/3.png)  
![本地控制](/images/rancher/rancher-base/4.png)  
启动本地权限控制后，我们可以点击账号管理来添加更多的管理账号  
![账号添加](/images/rancher/rancher-base/5.png)  
或者在系统管理的二级菜单中找到账号设置：  
![账号设置](/images/rancher/rancher-base/6.png)  
点击添加账号，添加更多的管理账号：  
![添加账号信息](/images/rancher/rancher-base/7.png)  

### RANCHER环境管理
Rancher支持将资源分组到多个环境中。每个环境都从用于创建环境的环境模板定义的一组基础结构服务开始。每个环境都有自己的资源集，由一个或多个用户或组拥有。例如，您可以创建单独的“开发”，“测试”和“生产”环境，以保持彼此的隔离，并为您的整个组织提供“开发”访问权限，但将“生产”环境限制为较小的团队。所有主机和任何Rancher资源（如容器，基础设施服务等）都可以在环境中创建。

1. 添加RANCHER环境  
   ![环境管理](/images/rancher/rancher-base/8.png)  
   ![环境管理界面](/images/rancher/rancher-base/9.png)  
   点击添加环境界面，输入名称与描述，从rancher预制的模板中选择一个模板。然后添加具有访问权限的用户  
   ![添加环境](/images/rancher/rancher-base/10.png)  
   ![给环境授权用户](/images/rancher/rancher-base/11.png)  
   在添加完环境的列表中，也可以编辑对应的环境，编辑环境的内容和添加一样。  
   ![编辑环境](/images/rancher/rancher-base/12.png)

2. 添加主机到RANCHER环境  
   主机是Rancher中最基本的资源单元，并且表示为虚拟或物理的任何Linux服务器，具有以下最低要求：
-  任何具有受支持版本的Docker的现代Linux发行版。RancherOS，Ubuntu，RHEL / CentOS 7进行了更严格的测试。
-  - 对于RHEL / CentOS，Docker不推荐使用默认存储驱动程序，即使用环回的devicemapper 。请参考Docker文档，了解如何更改。
-  - 对于RHEL / CentOS，如果要启用SELinux，则需要安装其他SELinux模块。
-  - 对于RHEL / CentOS，请使用内核版本3.10.0-514.2.2.el7.x86_64或更高版本。使用7.3版或更高版本时包括。
-  1GB RAM
-  推荐CPU / AES-NI
-  能够通过预先配置的端口通过http或https与Rancher服务器进行通信。默认值为8080。
-  能够在同一环境下路由到任何其他主机，以利用Rancher对Docker容器的跨主机网络。
-  Rancher还支持Docker Machine，并允许您通过任何支持的驱动程序添加主机。
-  主机上必须安装rancher所支持的版本，可参考[rancher支持的docker版本](http://docs.rancher.com/rancher/v1.5/en/hosts/#supported-docker-versions)

从基础设施 - > 主机选项卡，单击添加主机。  
![添加主机](/images/rancher/rancher-base/13.png)  
点击保存，进入添加主机界面，选择不同类型的主机，这里添加本地服务器，可以给每个主机添加不同的labels  
![添加主机1](/images/rancher/rancher-base/14.png)  
![添加主机2](/images/rancher/rancher-base/15.png)  
点击复制链接，在指定的本地的服务器上粘贴复制的内容，等待下截rancher agent下载运行后，关闭rancher的复制页面，在主机界面就会出现对应的主机信息：
![主机](/images/rancher/rancher-base/16.png)




