---
title: docker-registry私有仓库及web ui镜像管理
toc: true   // 在文章侧边增加文章目录
date: 6/20/2017 5:26:15 PM 
updated: 6/20/2017 5:26:20 PM 
categories: [docker]
tags: [docker,docker-compose,linux,registry]

---

-------
![registry](/images/docker/registry/registry.jpeg)

docker提供了开放式的dockerhub公共中央仓库，我们可以从这上面下载到自己工作和学习中想要的各种镜像，但由于dockerhub是国外的服务器，很多时候国内是很难下载，这里我们可以通过国内的中转服务器来下载。
国内常用的中转服务器有：  
1. daocloud： 在[daocloud](https://hub.daocloud.io/)通过[安装daocloud加载器](https://www.daocloud.io/mirror#accelerator-doc)可以下载到需要的镜像
2. 阿里云容器hub：在[开发者平台的->容器hub](https://dev.aliyun.com/search.html)中搜索镜像，如果想通过阿里云服务来下载镜像，需要登录阿里云->产品与服务->云计算基础服务->弹性计算->容器服务->镜像与方案->镜像->镜像仓库控制台->Docker Hub 镜像站点中有安装阿里云的dockerhub加载器
3. 网易蜂巢镜像中心：在[网易蜂巢镜像中心](https://c.163.com/hub#/m/home/)搜索镜像，然后复制docker pull 链接，到服务器上执行就行

-------

以上是公共的docker仓库获取镜像的方法，但在国内大部分公司的情况是不能把项目上用到的镜像放到公网上，一般从公网上只会下载基础版本的镜像，然后基于这个基础版本的镜像自己在本地build镜像，由于在本地build的镜像，不能在内部网络中直接下载，上传，所以需要一个内部环境的私有仓库，做为内部镜像中心来让容器，可以在内容任何地方下载上传。  
搭建私有仓库有如下的优点：
* 节省网络带宽，提升Docker部署速度，不用每个镜像从DockerHub上去下载，只需从私有仓库下载就可；
* 私有镜像，包含公司敏感信息，不方便公开对外，只在公司内部使用。  
本文就是基于以上目的，并参考[docker官方关于docker registry的构建](https://docs.docker.com/registry/)来进行搭建，并通过web ui来进行管理。

-------

### 从公共仓库中下载搭建私有仓库的镜像

```bash
[root@docker-node01 ~]# docker pull registry
[root@docker-node01 ~]# docker pull hyper/docker-registry-web
[root@docker-node01 ~]# docker images
REPOSITORY                      TAG                       IMAGE ID            CREATED             SIZE
registry                        latest                    9d0c4eabab4d        5 weeks ago         33.2MB
registry:5000/myhub/rabbitmp    latest                    758cc906ba57        2 months ago        37.5MB
tcr:5000/myhub/rabbitmq         3.6.9-management-alpine   758cc906ba57        2 months ago        37.5MB
hyper/docker-registry-web       latest                    0db5683824d8        8 months ago        599MB
```

### 启用身份验证搭建私有仓库
令牌认证需要具有PEM格式的RSA私钥和与该密钥相匹配的证书
#### 生成私钥和证书
编写生成证书的shell脚本：

```bash
[root@docker-node01 devops]# vim generate-key.s
#!/bin/bash

openssl req \
    -new \
    -newkey rsa:4096 \
    -days 3650 \
    -subj "/CN=localhost" \
    -nodes \
    -x509 \
    -keyout /devops/registry-web/conf/auth.key \
    -out /devops/registry/conf/auth.cert

:wq 保存
[root@docker-node01 devops]# chmod +x generate-key.sh
[root@docker-node01 devops]# ./generate-key.sh
Generating a 4096 bit RSA private key
.................................................................++
..................++
writing new private key to '/devops/registry-web/conf/auth.key'
-----
[root@docker-node01 devops]# ls /devops/registry
registry/     registry-web/
[root@docker-node01 devops]# ls /devops/registry/conf/
auth.cert
[root@docker-node01 devops]# ls /devops/registry-web/conf/
auth.key
```
#### 创建registry配置文件

```bash
[root@docker-node01 devops]# vim /devops/registry/config.yml
version: 0.1

storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true

http:
  addr: 0.0.0.0:5000

auth:
  token:
    realm: http://localhost:8090/api/auth
    service: registry:5000
    issuer: test
    rootcertbundle: /etc/docker/registry/auth.cert

log:
  level: info

notifications:
  endpoints:
    - name: listener
      url: http://localhost:8090/api/notification
      timeout: 500ms
      threshold: 5
      backoff: 1s

:wq 保存
      
```
说明：
`storage`选项是必需的
|---`filesystem` 使用本地磁盘来存储注册表文件。它是开发的理想选择，适用于一些小规模的生产应用
|---`delete` 用delete结构启用通过摘要删除图像斑点和清单。它默认为false,可以设置true来启用
`log`小节配置了日志系统的行为。日志记录系统将所有内容输出到stdout
|---`level`设置测井输出的级别。允许值是error，warn，info，和debug。默认是info
`http`选项详细说明托管registry的HTTP服务器的配置。
|---`addr`服务器应该接受连接的地址。表单取决于网络类型（使用HOST:PORTTCP和FILE一个UNIX套接字。
`auth`是验证方式，该选项是可选的。
|---`token`基于令牌的身份验证允许您将身份验证系统与注册表分离。它是一种具有高度安全性的已建立的认证范例。

| 参数 | 是否必须 | 说明 |
| --- | --- | --- |
| realm | 是 | registry服务器认证的API接口，这是通过registry-web中的权限来认证 |
| service | 是 | 指定需要被认证的服务。 |
| issuer | 是 | 令牌颁发者的名称。发行人将其插入到令牌中，因此它必须与为发行者配置的值相匹配 |
| rootcertbundle | 是 | 根证书包的绝对路径。此捆绑包包含用于签署身份验证令牌的证书的公共部分 |  

`notifications`，registry通知，此项不是必须的，可选选项
|---`endpoints`结构包含可以接受事件通知的命名服务（URL）的列表。具体参考[官网说明](https://docs.docker.com/registry/configuration/#notifications)

#### 创建regsitry的web ui管理镜像的配置文件

```bash
[root@docker-node01 devops]# vim /devops/registry-web/config.yml
registry:
   url: http://registry:5000/v2
   name: registry:5000
   readonly: false
   auth:
     enabled: true
     key: /conf/auth.key
     issuer: test
:wq 保存
     
```
说明：
`url`-指定registry的地址
`name`-regsitry私服的名称
`readonly`-是否为只读模式，设置为true时不允许删除
`auth`-registry-web的验证方式
|---`enabled` 是否启动验证
|---`key` 验证的证书key
|---`issuer` 证书颁发者的名称

#### 创建docker-compose文件来编排registry两个服务

```bash
[root@docker-node01 devops]# vim docker-compose.yml
version: '2'
services:
  registry:
    image: registry
    container_name: registry
    ports:
      - "5000:5000"
    volumes:
      - /devops/registry/conf:/etc/docker/registry:ro
      - /devops/registry:/var/lib/registry
    networks:
      - registry-net
    environment:
      - TZ=Asia/Shanghai
    restart: always
  registry-web:
    image: hyper/docker-registry-web
    container_name: registry-web
    ports:
      - "8090:8080"
    volumes:
      - /devops/registry-web/conf:/conf:ro
      - /devops/registry-web/db:/data
    networks:
      - registry-net
    depends_on:
      - registry
    environment:
      - TZ=Asia/Shanghai
    restart: always
networks:
  registry-net:
:wq 保存

[root@docker-node01 devops]# docker-compose up -d
[root@docker-node01 devops]# docker-compose log -f 查看日志
  
```
#### 配置本地hosts表

```bash
[root@docker-node01 devops]# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.211.55.7 registry  #增加registry的hosts表
```

### 验证registry及web ui管理
1. 登录到http://localhost:8090/ 默认用户名/密码  admin/admin  
![registry-web管理界面](/images/docker/registry/1.jpg)
2. 创建测试用户并向该用户授予“全部写入”角色。  
![](/images/docker/registry/2.jpg)
3. 在本地shell上：

```bash
[root@docker-node01 devops]# docker login registry:5000
Username: troylc
Password:
Login Succeeded
[root@docker-node01 devops]# docker pull hello-world
[root@docker-node01 devops]# docker tag hello-world registry:5000/hello-world:latest
[root@docker-node01 devops]# docker push registry:5000/hello-world:latest
[root@docker-node01 devops]# docker rmi registry:5000/hello-world:latest
[root@docker-node01 devops]# docker run registry:5000/hello-world:latest
```
4. 在界面上查看，并操作（列表中可删除）
用troylc用户名登录，可以看到有一个hello-worlad镜像，展开列表  
![](/images/docker/registry/3.jpg)



