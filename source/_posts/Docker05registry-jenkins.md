---
title: jenkins-registry持续集成-jenkins-registry安装与数据迁移(一)
toc: true
date: 2017-01-08 00:49:23 Sunday
updated: 2017-01-08 00:49:34 Sunday
categories: [docker]
tags: [linux,docker,jenkins,registry]

---

最近项目中也是一直在用jenkins做持续集成，正好更新jenkins为最新版本及迁移原来的老环境。顺手把它记录下来，和大家一起分享（本文同样适用新入手jenkins的同学）。 

### 持续集成原理
持续集成, 简称CI（continuous integration）.
- CI作为敏捷开发重要的一步，其目的在于让产品快速迭代的同时，尽可能保持高质量.
- CI一种可以增加项目可见性，降低项目失败风险的开发实践。其每一次代码更新，都要通过自动化测试来检测代码和功能的正确性，只有通过自动测试的代码才能进行后续的交付和部署.
- CI 是团队成员间（产研测）更好地协调工作，更好的适应敏捷迭代开发，自动完成减少人工干预，保证每个时间点上团队成员提交的代码都能成功集成的，可以很好的用于对各种WEB、APP项目的打包.  
  
Jenkins  
[Jenkins](https://jenkins.io/index.html)是一个用Java编写的开源的持续集成工具，提供了软件开发的持续集成服务，可监控并触发持续重复的工作，具有开源，支持多平台和插件扩展，安装简单，界面化管理等特点。


附网上jenkins持续交付流程图  
![jenkins持续集成](/images/docker/jenkins/jenkins1-1.png)  
持续集成，持续交付各个阶段所使用的一些典型工具的使用，以及在各个阶段中的相关团队的相关活动，以下图为典型的DevOps相关的活动  
![jenkins持续集成2](/images/docker/jenkins/jenkins1-2.png)

  
### docker环境安装
参考：[docker系列-docker、docker-compse最新版本安装及版本升级](http://www.troylc.cc/docker/2017/01/05/docker04ininstall.html)

### jenkins和registry环境安装

首页在有联网条件下的docker环境中下载jenkins最新版本的docker镜像

#### 下载jenkins镜像和registry镜像

```
[root@registry-jenkins ~]# docker pull jenkins:2.32.1-alpine
2.32.1-alpine: Pulling from library/jenkins
b7f33cc0b48e: Pull complete 
43a564ae36a3: Pull complete 
b294f0e7874b: Downloading [=========================================>         ] 41.12 MB/49.36 MB
63c7a703a76a: Downloading [===========>                                       ] 5.144 MB/23.26 MB
1948a77ff7cc: Download complete 
ceb220f57d17: Downloading [=========================>                         ] 35.11 MB/69.93 MB
d0fbbc51c7ae: Waiting 
6eee39234906: Waiting 
6eee39234906: Pulling fs layer 
[root@registry-jenkins ~]# docker pull registry
latest: Pulling from library/registry
b7f33cc0b48e: Pull complete 
43a564ae36a3: Pull complete 
b294f0e7874b: Downloading [=========================================>         ] 41.12 MB/49.36 MB
63c7a703a76a: Downloading [===========>                                       ] 5.144 MB/23.26 MB
1948a77ff7cc: Download complete 
6eee39234906: Pulling fs layer 

[root@registry-jenkins ~]# docker images
REPOSITORY                   TAG                    IMAGE ID            CREATED             SIZE
jenkins                      2.32.1-alpine          0c0c0a437b20        10 days ago         263.7 MB
registry                     latest                 182810e6ba8c        10 days ago         37.6 MB
nginx                        latest                 abf312888d13        5 weeks ago         181.5 MB
mysql                        latest                 d9124e6c552f        6 weeks ago         383.4 MB
tomcat                       8.0.39-jre8-alpine     fbb6a04c1245        7 weeks ago         135.4 MB
java                         openjdk-8-jdk-alpine   d991edd81416        7 weeks ago         145 MB
tomcat                       8.5.5-jre8-alpine      af393862df5a        4 months ago        135 MB
redis                        latest                 4f5f397d4b7c        10 months ago       177.5 MB
toptomcat                    latest                 ce8c4307d74c        14 months ago       395.5 MB
topsecnginx                  latest                 bd299f0f0516        14 months ago       223.4 MB

```

#### 镜像加速器安装

==注意：==  

一般由于docker hub是国外的网站，下载镜像非常慢，各位可以用国内的加载器，这里我推荐两种加载器：

- daocloud加载器

```
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://69693d08.m.daocloud.io
```
该脚本可以将 --registry-mirror 加入到你的 Docker 配置文件 /etc/default/docker 中。适用于 Ubuntu14.04、Debian、CentOS6 、CentOS7、Fedora、Arch Linux、openSUSE Leap 42.1，具体地址为 [https://www.daocloud.io/mirror#accelerator-doc](https://www.daocloud.io/mirror#accelerator-doc)其他版本可能有细微不同。更多详情请[访问文档](http://guide.daocloud.io/dcs/daocloud-9153151.html)。

-  阿里云加载器  

阿里云加载器是你用阿里云账号登录后，在产品与服务中有一个加速器，点击加速器出现如下内容：
他会给没一个账号弄一个专属的加载器：我的是 专属加速器地址： https://1i186hp0.mirror.aliyuncs.com

安装或升级Docker

您可以通过阿里云的镜像仓库下载： mirrors.aliyun.com/help/docker-engine

```
curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
```

配置Docker加速器

您可以使用如下的脚本将mirror的配置添加到docker daemon的启动参数中。  
**系统要求 CentOS 7 以上，Docker 1.9 以上。**

```
sudo cp -n /lib/systemd/system/docker.service /etc/systemd/system/docker.service
sudo sed -i "s|ExecStart=/usr/bin/docker daemon|ExecStart=/usr/bin/docker daemon --registry-mirror=https://1i186hp0.mirror.aliyuncs.com|g" /etc/systemd/system/docker.service
sudo systemctl daemon-reload
sudo service docker restart
```


#### 通过docker-compose运行jenkins和regsitry

##### docker-compose文件内容：

```bash
[root@registry-jenkins ~]# vim docker-compose.yml
#docker run -p 8080:8080 -p 50000:50000 -v /your/home:/var/jenkins_home jenkins
jenkins:
  image: jenkins:2.32.1-alpine
  container_name: jenkins
  ports:
    - "8080:8080"
    - "50000:50000"
  volumes:
    - /opt/data/jenkins_home:/var/jenkins_home
  extra_hosts: 
    - "repo.topsec.com:172.19.6.42"
  user: root
  restart: always
#docker run -d --restart=always -p 5000:5000 -v /opt/devdata/registry:/var/lib/registry --name devregistry registry
registry:
  image: registry
  container_name: devregistry
  ports:
    - "5000:5000"
  volumes:
    - /opt/devdata/registry:/var/lib/registry
  restart: always
  
[root@registry-jenkins ~]# docker-compose up -d
[root@registry-jenkins devdata]# docker-compose ps
   Name                  Command               State                        Ports                       
-------------------------------------------------------------------------------------------------------
devregistry   /entrypoint.sh /etc/docker ...   Up      0.0.0.0:5000->5000/tcp                           
jenkins       /bin/tini -- /usr/local/bi ...   Up      0.0.0.0:50000->50000/tcp, 0.0.0.0:8080->8080/tcp 


```
##### 迁移jenkins和registry数据  
如果是之前有运行了老版本的jenkins持续集成的内容，docker环境会有一个volumes的映射目录，只有从老的jenkins映射的目录，打包，拷贝到新的指定目录，然后在docker-compose.yml文件中在volumes中指定你拷贝的新的目录就行。后面的操作和上面一样。

registry和jenkns类似，都有一个挂载的目录存储着镜像内容，只有将该文件下的内容拷贝到新的环境目录，然后在docker-compose文件指定目录位置就行。如上 docker-compose.yml文件中的内容

### 参考：

[docker registry v2 认证服务器](http://www.dockerinfo.net/?s=registry)  
[从零开始搭建Jenkins+Docker自动化集成环境](http://www.dockerinfo.net/2457.html)










