---
title: 用docker构建与部署spring-cloud的微服务(七)  
toc: true  
date: 2017-04-24 19:18:23 Sunday  
updated: 2017-04-24 19:49:34 Sunday  
categories: [spring cloud]  
tags: [spring cloud,docker,swarm,docker-compose]  

---

首先总结一下前面一系列的spring-cloud微服务学习，我们用eureka做服务的注册中心，通过向服务注册中心注册了三个简单的服务，如：用户微服务(microservice-provider-userservice)，商品微服务(microservice-consumer-productservice)，异构平台的评论微服务(microservice-sidecar-comment),在商品微服务中，通过spring cloud FeignClient来进行微服务之间的相互调用，通过sprig cloud zuul来暴露外维系统想访问微服务的接口，并用spring cloud config搭建了一个分布式的配置中心，通过改造用户微服务，来实现分布式的服务的配置功能。用spring cloud bus以及kafka的消息机制来实现服务配置的无停机就能自动刷新加载。本系列相关的文章，在本节就结束了，此系统纯属老司机的学习总结，欢迎大家指正交流，达到相互学习的目的。

## 此系统列参考文档：
疑问 | 参考
---|--- 
如果你对spring cloud config不知道怎么配置 | [SpringCloud构微服务之-配置中心](http://www.troylc.cc/spring-cloud/2017/04/16/spirng-cloud-config.html)
如果你对spring cloud的怎么把异构平台的服务纳入微服务 | [SpringCloud构建异构平台的微服务之-sidecar](http://www.troylc.cc/spring-cloud/2017/04/13/spirng-cloud-sidecar.html) 
如果你对spring cloud的服务网关不太了解？| [SpringCloud构建微服务之-apiGateway](http://www.troylc.cc/spring-cloud/2017/03/19/spirng-cloud-apigateway.html)
如果你对eureka注册中心不太了解？ | [SpringCloud和docker之微服务-eureka](http://www.troylc.cc/spring-cloud/2017/03/01/spirng-cloud-eureka.html)
如果你对服务注册不太了解 | [SpringCloud和docker之微服务-provider](http://note.youdao.com/)  
如果你对服务消费不太了解 | [SpringCloud和docker之微服务-consumer](http://www.troylc.cc/spring-cloud/2017/03/11/spirng-cloud-productservice.html)
如果你对docker安装不了解 | [docker、docker-compse最新版本安装](http://www.troylc.cc/docker/2017/01/05/docker04ininstall.html)
如果你对docker-swarm集群创建不太了解 | [docker-swarm创建与管理集群](http://www.troylc.cc/docker/2017/02/07/Docker07docker-swarm01.html)
如果你对swarm集群的服务部署不太了解 | [docker-swarm集群服务部署与维护](http://www.troylc.cc/docker/2017/02/19/Docker07docker-swarm02.html)
如果你不知道docker-compose怎么来部署swarm集群? | [docker-compose部署swarm服务(docker1.13.1)](http://www.troylc.cc/docker-compose/2017/02/25/Docker08docker-compose01.html)


本节通过maven和idea的插件来构建docker镜像，编写docker-compose.yml来编排各服务节点，通过compose命令在docker-swarm中部署前面章节编写的微服务内容。

## 微服务镜像的构建。
在构建镜像之前，我们需要把一台内容的docker的远程API开放，以及内网搭建了有自己的docker私有仓库等功能。参考[jenkins-registry持续集成-jenkins-registry安装与数据迁移](http://www.troylc.cc/docker/2017/01/08/Docker05registry-jenkins.html)、[docker系列(二)使用Docker-Remote-API](http://www.troylc.cc/docker/2016/07/31/docker-02.html)

镜像的构建如果是maven管理的java程序，我们可以通过maven的插件来进行镜像构建，如以下几个项目，都是通过maven插件来构建的。
首先在项目的主pom文件中加载一下docker的maven插件docker-maven-plugin，如：  
![build](/images/spring-cloud/docker-swarm/1.png)

```
<build>
    ......
    <!--添加利用maven插件构建docker镜像的插件依赖-->
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.4.13</version>
            </plugin>
        </plugins>
    </pluginManagement>
</build>
```
- microservice-eureka-service注册中心镜像构建，在pom.xml插件处加入如下配置，  

```
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>build-image</id>
            <phase>install</phase>
            <goals>
                <goal>build</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <!--安装了docker的主机，并且打开了api remote接口设置-->
        <dockerHost>http://10.211.55.4:8372</dockerHost>
        <pushImage>true</pushImage><!--设置上传镜像到私有仓库，需要docker设置指定私有仓库地址-->
        <!--镜像名称-->
        <imageName>${docker.repostory}/${docker.image.prefix}/${project.artifactId}:${project.version}</imageName>
        <!--镜像的基础版本-->
        <baseImage>java:openjdk-8-jdk-alpine</baseImage>
        <!--镜像启动参数-->
        <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
        <!--<imageTags>
            <imageTag>${project.version}</imageTag>
        </imageTags>-->
        <resources>
            <resource>
                <targetPath>/</targetPath>
                <directory>${project.build.directory}</directory>
                <include>${project.build.finalName}.jar</include>
            </resource>
        </resources>
    </configuration>
</plugin>
```

![maven的构建](/images/spring-cloud/docker-swarm/2.png)  
执行maven的构建：  
![maven的构建](/images/spring-cloud/docker-swarm/3.png)  
查看对应docker环境上的镜像

在docker远程主机查看本地镜像，如：

```  

[root@docker-master01 ~]# docker images
REPOSITORY                                            TAG                       IMAGE ID            CREATED             SIZE
tcr:5000/myhub/microservice-eureka-service            0.1.0                     032a842e950c        4 minutes ago       186 MB

```
  
- microservice-provider-userservice用户微服务镜像构建  
在用户微服务的pom文件增加构建镜像配置，并执行install命令构建镜像     
![maven的构建](/images/spring-cloud/docker-swarm/4.png)  

在docker远程主机查看本地镜像，如：  

```
[root@docker-master01 ~]# docker images
REPOSITORY                                            TAG                       IMAGE ID            CREATED             SIZE
tcr:5000/myhub/microservice-provider-userservice      0.1.0                     b8bf9b72c012        6 minutes ago       215 MB
tcr:5000/myhub/microservice-eureka-service            0.1.0                     032a842e950c        36 minutes ago      186 MB

```   

  
- microservice-consumer-productservice商品微服务镜像构建  
在商品微服务的pom文件增加构建镜像的配置，并执行install命令构建镜像   
![maven的构建](/images/spring-cloud/docker-swarm/5.png)  

在docker远程主机查看本地镜像，如：  

```
[root@docker-master01 ~]# docker images
REPOSITORY                                            TAG                       IMAGE ID            CREATED             SIZE
tcr:5000/myhub/microservice-consumer-productservice   0.1.0                     4632254f9d3c        17 minutes ago      187 MB
tcr:5000/myhub/microservice-provider-userservice      0.1.0                     b8bf9b72c012        27 minutes ago      215 MB
tcr:5000/myhub/microservice-eureka-service            0.1.0                     032a842e950c        57 minutes ago      186 MB
  
```  

  
- microservice-sidecar-comment异构平台接入微服务构建代码  
在接入微服务异构平台的接入项目中的pom文件增加构建镜像的配置，并执行install命令构建镜像     
![maven的构建](/images/spring-cloud/docker-swarm/6.png)   

在docker远程主机查看本地镜像，如：  

```  

[root@docker-master01 ~]# docker images
REPOSITORY                                            TAG                       IMAGE ID            CREATED             SIZE
tcr:5000/myhub/microservice-sidecar-comment           0.1.0                     148fb0bf84e9        8 minutes ago       184 MB
tcr:5000/myhub/microservice-consumer-productservice   0.1.0                     4632254f9d3c        17 minutes ago      187 MB
tcr:5000/myhub/microservice-provider-userservice      0.1.0                     b8bf9b72c012        27 minutes ago      215 MB
tcr:5000/myhub/microservice-eureka-service            0.1.0                     032a842e950c        57 minutes ago      186 MB
  
```
  
- microservice-config-service配置中心服务的构建代码  
在微服务配置中心的项目中的pom文件增加构建镜像的配置，并执行install命令构建镜像  
![maven的构建](/images/spring-cloud/docker-swarm/7.png)  

在docker远程主机查看本地镜像，如：

```  

[root@docker-master01 ~]# docker images
REPOSITORY                                            TAG                       IMAGE ID            CREATED             SIZE
tcr:5000/myhub/microservice-config-service            0.1.0                     c895d68cfff0        5 minutes ago       204 MB
tcr:5000/myhub/microservice-sidecar-comment           0.1.0                     148fb0bf84e9        8 minutes ago       184 MB
tcr:5000/myhub/microservice-consumer-productservice   0.1.0                     4632254f9d3c        17 minutes ago      187 MB
tcr:5000/myhub/microservice-provider-userservice      0.1.0                     b8bf9b72c012        27 minutes ago      215 MB
tcr:5000/myhub/microservice-eureka-service            0.1.0                     032a842e950c        57 minutes ago      186 MB

```

- microservice-nodejs-comment异构平台构建方法  
此服务是用nodes编写的，和现有的微服务不是一种语言，如果要接入到微服务中，并部署到docker-swarm集群中，其一就是把自己的服务提供rest接口供微服务接入项目sidecar来配置接入，其二把编写的整个项目docker化。
此项目的docker化，因为没有maven来管理，所以需要编写一个dockerfile文件来进行构建，并且构建的方式是通过idea工具的docker插件来操作的。  
Dockerfile文件内容：  

```

FROM node:7.7.4-alpine

# Create app directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Install app dependencies
COPY package.json /usr/src/app/
#RUN npm install

# Bundle app source
COPY . /usr/src/app

EXPOSE 3000
CMD [ "npm", "start" ]

```  

docker插件构建nodejs项目的镜像：  
1. 首页在idea工具中安装docker插件  
![idea插件](/images/spring-cloud/docker-swarm/8.png)  
2. 其次配置cloud连接远程docker环境  
![插件配置](/images/spring-cloud/docker-swarm/9.png)  
3. 最后通过docker控制台构建与运行镜像  
![构建镜像](/images/spring-cloud/docker-swarm/10.png)  
![构建镜像](/images/spring-cloud/docker-swarm/11.png)  

在docker远程主机查看本地镜像，如：  

```  

[root@docker-master01 ~]# docker images
REPOSITORY                                            TAG                       IMAGE ID            CREATED             SIZE
tcr:5000/myhub/microservice-nodejs-comment            0.1.0                     5f30b81a7df3        4 minutes ago       66.3 MB
tcr:5000/myhub/microservice-config-service            0.1.0                     c895d68cfff0        About an hour ago   204 MB
tcr:5000/myhub/microservice-sidecar-comment           0.1.0                     148fb0bf84e9        About an hour ago   184 MB
tcr:5000/myhub/microservice-consumer-productservice   0.1.0                     4632254f9d3c        About an hour ago   187 MB
tcr:5000/myhub/microservice-provider-userservice      0.1.0                     b8bf9b72c012        About an hour ago   215 MB
tcr:5000/myhub/microservice-eureka-service            0.1.0                     032a842e950c        2 hours ago         186 MB
  
```
7.microservice-api-gateway微服务的API对外网关镜像构建  
![maven构建镜像](/images/spring-cloud/docker-swarm/12.png)  

在docker远程主机查看本地镜像，如：  

```  

[root@docker-master01 ~]# docker images
REPOSITORY                                            TAG                       IMAGE ID            CREATED             SIZE
tcr:5000/myhub/microservice-api-gateway               0.1.0                     208d6656085a        2 minutes ago       184 MB
tcr:5000/myhub/microservice-nodejs-comment            0.1.0                     5f30b81a7df3        34 minutes ago      66.3 MB
tcr:5000/myhub/microservice-config-service            0.1.0                     c895d68cfff0        2 hours ago         204 MB
tcr:5000/myhub/microservice-sidecar-comment           0.1.0                     148fb0bf84e9        2 hours ago         184 MB
tcr:5000/myhub/microservice-consumer-productservice   0.1.0                     4632254f9d3c        2 hours ago         187 MB
tcr:5000/myhub/microservice-provider-userservice      0.1.0                     b8bf9b72c012        2 hours ago         215 MB
tcr:5000/myhub/microservice-eureka-service            0.1.0                     032a842e950c        2 hours ago         186 MB

```  

到此运行本次微服务的所有镜像都已经构建完成。版本都为0.1.0  
  
## docker-compose.yml的编写

```  

version: "3"
services:
  eurekaService1:      # 默认情况下，其他服务可以使用服务名称连接到该服务。因此，对于eurekaService1的节点，它需要连接http://eurekaService2/3:951X/eureka/，因此需要配置该服务的名称是eurekaService1。
    image: tcr:5000/myhub/microservice-eureka-service:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9511:9511"
    environment:
      - spring.profiles.active=eurekaService1
  eurekaService2:    # 高可用eureka注册节点2
    image: tcr:5000/myhub/microservice-eureka-service:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9512:9512"
    environment:
      - spring.profiles.active=eurekaService2
  eurekaService3:    # 高可用eureka注册节点3
    image: tcr:5000/myhub/microservice-eureka-service:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9513:9513"
    environment:
      - spring.profiles.active=eurekaService3
  productService:    # 商品微服务
    image: tcr:5000/myhub/microservice-consumer-productservice:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9515:9515"
    depends_on:
      - eurekaService1
      - eurekaService2
      - eurekaService3
  apiGateway:  #服务网关服务
    image: tcr:5000/myhub/microservice-api-gateway:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9516:9516"
    depends_on:
      - productService
      - userService
      - eurekaService1
      - eurekaService2
      - eurekaService3
  nodeComment:    #异构平台商品评价服务
    image: tcr:5000/myhub/microservice-nodejs-comment:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "3000:3000"
  sidecarComment:    #接入异构平台的微服务
    image: tcr:5000/myhub/microservice-sidecar-comment:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9517:9517"
    depends_on:
      - nodeComment
      - eurekaService1
      - eurekaService2
      - eurekaService3
  zookeeper:       #zookeeper服务，主要是协助kafka消息中心的
    image: tcr:5000/myhub/zookeeper:3.4.9
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "2181:2181"
  kafka:      #kafka消息中心，在此主要是用户配置刷新的消息通知。
    image: tcr:5000/myhub/kafka:0.10.1.1
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9092:9092"
    environment:
      - 'KAFKA_ADVERTISED_HOST_NAME=kafka'
      - 'KAFKA_ADVERTISED_PORT=9092'
      - 'KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181'
    depends_on:
      - zookeeper
  rabbitmq:     #rqbbitmq消息中心，在此主要是用户配置刷新的消息通知。
    image: tcr:5000/myhub/rabbitmq:3.6.9-management-alpine
    hostname: 'my-rabbit'
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "5672:5672"
      - "15672:15672"
  configService:    #微服务的配置中心
    image: tcr:5000/myhub/microservice-config-service:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9518:9518"
    depends_on:
      - kafka
      - zookeeper
      - eurekaService1
      - eurekaService2
      - eurekaService3
  userService:      #用户微服务，且通过config service中获取相关配置信息
    image: tcr:5000/myhub/microservice-provider-userservice:0.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - eureka-net
    ports:
      - "9514:9514"
    depends_on:
      - kafka
      - zookeeper
      - configService
      - eurekaService1
      - eurekaService2
      - eurekaService3
networks:
  eureka-net:
    driver: overlay
      
```
  
## docker compose在docker-swarm中部署微服务

```  

[root@docker-master01 docker-compose]# docker stack deploy -c docker-compose.yml microservice
Creating network microservice_eureka-net
Creating service microservice_apiGateway
Creating service microservice_kafka
Creating service microservice_configService
Creating service microservice_eurekaService2
Creating service microservice_userService
Creating service microservice_zookeeper
Creating service microservice_sidecarComment
Creating service microservice_eurekaService3
Creating service microservice_nodeComment
Creating service microservice_rabbitmq
Creating service microservice_eurekaService1
Creating service microservice_productService
[root@docker-master01 docker-compose]# docker stack services microservice
ID            NAME                         MODE        REPLICAS  IMAGE
3gjonfu6ebh5  microservice_kafka           replicated  1/1       tcr:5000/myhub/kafka:0.10.1.1
5qen0mvwwa5x  microservice_productService  replicated  1/1       tcr:5000/myhub/microservice-consumer-productservice:0.1.0
be7j9n6vdm69  microservice_apiGateway      replicated  1/1       tcr:5000/myhub/microservice-api-gateway:0.1.0
ifjhurz97i80  microservice_eurekaService2  replicated  1/1       tcr:5000/myhub/microservice-eureka-service:0.1.0
j6gsnfet2esa  microservice_configService   replicated  1/1       tcr:5000/myhub/microservice-config-service:0.1.0
js55ijbe4crw  microservice_userService     replicated  1/1       tcr:5000/myhub/microservice-provider-userservice:0.1.0
lqriwyu6npph  microservice_nodeComment     replicated  1/1       tcr:5000/myhub/microservice-nodejs-comment:0.1.0
qc7s6fm4lnqg  microservice_sidecarComment  replicated  1/1       tcr:5000/myhub/microservice-sidecar-comment:0.1.0
t67efu3aw48t  microservice_eurekaService3  replicated  1/1       tcr:5000/myhub/microservice-eureka-service:0.1.0
tn33z3ebce8a  microservice_zookeeper       replicated  1/1       tcr:5000/myhub/zookeeper:3.4.9
wiqe6x1pqpx4  microservice_rabbitmq        replicated  1/1       tcr:5000/myhub/rabbitmq:3.6.9-management-alpine
xd5owqhu7h61  microservice_eurekaService1  replicated  1/1       tcr:5000/myhub/microservice-eureka-service:0.1.0
[root@docker-master01 docker-compose]# docker stack ps microservice
ID            NAME                           IMAGE                                                      NODE             DESIRED STATE  CURRENT STATE           ERROR  PORTS
3b6wsnap14jw  microservice_productService.1  tcr:5000/myhub/microservice-consumer-productservice:0.1.0  docker-node01    Running        Running 12 seconds ago         
pwir78iy5gwr  microservice_eurekaService1.1  tcr:5000/myhub/microservice-eureka-service:0.1.0           docker-node01    Running        Running 31 seconds ago         
sp3hcoibmh6l  microservice_rabbitmq.1        tcr:5000/myhub/rabbitmq:3.6.9-management-alpine            docker-node01    Running        Running 20 seconds ago         
gwdrnfyyoia1  microservice_nodeComment.1     tcr:5000/myhub/microservice-nodejs-comment:0.1.0           docker-node02    Running        Running 26 seconds ago         
rva9h3xc82ub  microservice_eurekaService3.1  tcr:5000/myhub/microservice-eureka-service:0.1.0           docker-master01  Running        Running 25 seconds ago         
a3qy8xlqq65p  microservice_sidecarComment.1  tcr:5000/myhub/microservice-sidecar-comment:0.1.0          docker-node01    Running        Running 15 seconds ago         
d8s8lpwj7pp6  microservice_zookeeper.1       tcr:5000/myhub/zookeeper:3.4.9                             docker-node02    Running        Running 21 seconds ago         
ob1z00fyzhov  microservice_userService.1     tcr:5000/myhub/microservice-provider-userservice:0.1.0     docker-master01  Running        Running 24 seconds ago         
xjrakh9bxh0j  microservice_eurekaService2.1  tcr:5000/myhub/microservice-eureka-service:0.1.0           docker-node02    Running        Running 30 seconds ago         
7y2nczvpi7wl  microservice_configService.1   tcr:5000/myhub/microservice-config-service:0.1.0           docker-node02    Running        Running 18 seconds ago         
85k0gkln3k7p  microservice_kafka.1           tcr:5000/myhub/kafka:0.10.1.1                              docker-master01  Running        Running 25 seconds ago         
csniz9pc8bbs  microservice_apiGateway.1      tcr:5000/myhub/microservice-api-gateway:0.1.0              docker-master01  Running        Running 29 seconds ago 

```
出现以上结束，表示通过compose对docker化的微服务部署就完成了。下面我们来验证一下

## 测试相关微服务的功能
查看一下eureka注册中心，我们可以看到所有的微服务都已经注册上来了，我们把之前每篇的spring cloud的验证环节的操作，都在此操作一下，就会看到相同的结束，这里就一一验证了，大家可根据上面的源码及部署方式自行测试一下。  
![测试](/images/spring-cloud/docker-swarm/13.png)


