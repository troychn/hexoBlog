---
title: docker1.12.3 docker-swarm集群服务部署与维护（二）
toc: true
date: 2017-02-19 19:18:23 Sunday
updated: 2017-02-19 19:49:34 Sunday
categories: [docker]
tags: [docker,swarm]

---
在上一节中，我们已经搭建了docker swarm集群，并且完成对docker swarm集群的基本管理。  
接下来我们讨论一下的原理以及怎么在搭建好的docker集群中部署集群服务。


### swarm服务运行的原理：  
在docker swarm中部署应用程序，是通过创建service来实现，这里的service的概念通过是指在一个大的应用上下文中的一个微服务，比如在电商的购物网站中：用户管理、订单管理、库存管理，都是购物网站这个大应用中的一个一个微服务，这些服务可以是单个或者多个的在集群模式中运行。  
service的类型可以是HTTP的应用程序、数据库、缓存服务等分布式环境中任何可执行的程序。  
**service可定义的选项：**
- 可以在swarm集群的外部提供访问服务的端口
- 可以通过覆盖网络(overlay)模式连接到群中的其他服务
- 可以设置CPU和内存限制和预留值
- 可以实现滚动更新策略
- 可以指定要在群中运行的镜像的副本数  

**服务，任务和容器**  
当您将服务部署到群集时，swarm管理器接受您的服务定义作为服务的所需状态。然后它在swarm中的节点上将服务调度为一个或多个副本任务。这些任务在群中的节点上彼此独立地运行。  

**下图解释服务、任务、容器：** 

![服务、任务、容器](/images/docker/swarm/2-1.png)  

**服务的任务及调试说明：**  

![服务的任务及调试说明](/images/docker/swarm/2-2.png)  

**服务部署的复制模式和全局模式说明：**  

![服务部署的复制模式和全局模式说明](/images/docker/swarm/2-3.png)
**参考：** [https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/](https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/)

### docker-swarm服务部署  

- **环境如下：**  

IP | 主机名 | 角色
---|---|--- 
172.19.6.222 | cloud01 | manager
172.19.6.223 | cloud02 | worker
172.19.6.224 | cloud03 | worker
  
```bash
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Active        
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        

```
  
- **服务部署规划：**   

服务名 | 副本数 | 网络 | 部署模式 | 说明
---|---|---|---|---
nginx | 2 | my-network | replicated | nginx服务依赖于bootservice服务  
bootService | 3 | my-network | replicatedl | bootservice服务依赖于mysql服务 
mysql | 1 | my-network | replicated | mysql服务  

- **服务部署图：**   

![服务流程图](/images/docker/swarm/2-4.png)

### 服务部署实施：
####  BUILD服务所需的容器：

- **程序代码结构：**  

![程序结构](/images/docker/swarm/2-5.png)

- **代码参考：**   

[github源代码](https://github.com/troychn/springboot-docker-swarm)
![github源码](/images/docker/swarm/2-6.png)

把代码从github中下载下来，通过maven编译工具进行编译打包，一般都集成一些开发工具一起使用，我这里通过IDEA编译打包好的jar和dockerfile文件拷贝到docker环境下，如：  

```bash
[root@dmpdev docker-swarm-service]# ls
bootService  mysql  nginx
[root@dmpdev docker-swarm-service]# ll
总用量 0
drwxr-xr-x. 2 root root 67 11月 30 17:42 bootService
drwxr-xr-x. 2 root root 74 11月 30 17:42 mysql
drwxr-xr-x. 2 root root 59 11月 30 17:42 nginx

```
- **构建mysql镜像：**

```bash
[root@dmpdev docker-swarm-service]# cd mysql
[root@dmpdev mysql]# ls
city-db.sql  Dockerfile  my.cnf
[root@dmpdev mysql]# docker build -t tcr:5000/myhub/mysql:5.7-dws .
Sending build context to Docker daemon 28.67 kB
Step 1 : FROM mysql
 ---> d9124e6c552f
Step 2 : MAINTAINER troylc <troylc@163.com>
 ---> Running in 0b95a555c8cf
 ---> 5ab93bf1762d
Removing intermediate container 0b95a555c8cf
Step 3 : COPY city-db.sql /docker-entrypoint-initdb.d/city-db.sql
 ---> 81385fa16444
Removing intermediate container 718e7a1369c3
Step 4 : COPY my.cnf /etc/mysql/my.cnf
 ---> 5669d98b188a
Removing intermediate container 926998a0e768
Step 5 : RUN echo "Asia/shanghai" > /etc/timezone
 ---> Running in 9b23322b62db
 ---> ff6e7d0e75af
Removing intermediate container 9b23322b62db
Step 6 : EXPOSE 3306
 ---> Running in 8bb45bc9c3e6
 ---> 9d71c84043af
Removing intermediate container 8bb45bc9c3e6
Successfully built 9d71c84043af
```
push到docker私有仓库
```bash
[root@dmpdev mysql]# docker push tcr:5000/myhub/mysql:5.7-dws
The push refers to a repository [tcr:5000/myhub/mysql]
0b63e7f5c61b: Image successfully pushed 
a9103bf08dd6: Image successfully pushed 
4a8cfaac9133: Image successfully pushed 
ee30b869dd90: Image successfully pushed 
b5d824491b78: Already exists 
b26238180bc8: Already exists 
01e91410235e: Already exists 
b610b16e919f: Already exists 
1574ff8789b1: Already exists 
e7048a1643a4: Already exists 
1bc74a039df4: Already exists 
6ebad06b3e49: Already exists 
f1621398948b: Already exists 
fe4c16cbf7a4: Already exists 
Pushing tag for rev [9d71c84043af] on {http://tcr:5000/v1/repositories/myhub/mysql/tags/5.7-dws}
```
- **构建bootService镜像：**  

```bash
[root@cloud03 mysql] cd ../bootService/
[root@cloud03 bootService]# docker build -t tcr:5000/myhub/bootservice:1.0-dws .
Sending build context to Docker daemon 28.77 MB
Step 1 : FROM java:openjdk-8-jre-alpine
 ---> ed933d9cbb9b
Step 2 : MAINTAINER troylc <troylc@163.com>
 ---> Running in e020e5dc61a4
 ---> 1dca6a2dc31c
Removing intermediate container e020e5dc61a4
Step 3 : ADD springboot_controller-1.0-SNAPSHOT.jar /app.jar
 ---> 94d9cb5ce2a3
Removing intermediate container f151c9ea203e
Step 4 : EXPOSE 8888
 ---> Running in 1257615513df
 ---> a192b60e42f7
Removing intermediate container 1257615513df
Step 5 : ENTRYPOINT java -jar /app.jar
 ---> Running in 623ec299ddaa
 ---> 87e646e370af
Removing intermediate container 623ec299ddaa
Successfully built 87e646e370af

[root@dmpdev mysql]# docker push tcr:5000/myhub/bootservice:1.0-dws
......
```

- **构建nginx镜像：**   

```bash
[root@cloud03 bootService] cd ../nginx/
[root@cloud03 nginx]# docker build -t tcr:5000/myhub/nginx:1.11-dws .
Sending build context to Docker daemon 7.168 kB
Step 1 : FROM nginx
 ---> abf312888d13
Step 2 : MAINTAINER troylc <troylc@163.com>
 ---> Running in 883582f786a5
 ---> f234706e9ee5
Removing intermediate container 883582f786a5
Step 3 : COPY nginx.conf /etc/nginx/nginx.conf
 ---> 82053ab65cae
Removing intermediate container b2eb37dd4ecc
Step 4 : COPY default.conf /etc/nginx/conf.d/default.conf
 ---> f43184cfb2fa
Removing intermediate container 0f7add1f3f67
Step 5 : RUN mkdir -p /etc/nginx/logs
 ---> Running in 74c0fbabe4ee
 ---> f20f5d30e5db
Removing intermediate container 74c0fbabe4ee
Step 6 : RUN echo "Asia/shanghai" > /etc/timezonedo
 ---> Running in 35003953a04b
 ---> b3bb199c7506
Removing intermediate container 35003953a04b
Step 7 : EXPOSE 80
 ---> Running in 6d7ded8bb94b
 ---> 7e0d4b715b92
Removing intermediate container 6d7ded8bb94b
Successfully built 7e0d4b715b92
[root@dmpdev mysql]# docker push tcr:5000/myhub/nginx:1.11-dws
......
```
#### 创建overlay网络：  
my-network,使用overlay网络连接集群中的一个或多个服务。
在manager节点上创建overlay网络，使用docker network create命令：  
**--driver overlay** 网络类型  
**--subnet 10.0.9.0/24** 子网地址段  
**--opt encrypted** 给此网络加密（[具体说明请参考：https://docs.docker.com/engine/userguide/networking/overlay-security-model/](https://docs.docker.com/engine/userguide/networking/overlay-security-model/)）

```bash
[root@cloud01 ~]#  docker network create \
>   --driver overlay \
>   --subnet 10.0.9.0/24 \
>   --opt encrypted \
>   my-network
bhna620fxpuryf2e1te6hui1t
[root@cloud01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
8fd3e2a10dff        bridge              bridge              local               
987934476ba0        docker_gwbridge     bridge              local               
77ac8b224344        host                host                local               
aeyt32avj7kq        ingress             overlay             swarm               
bhna620fxpur        my-network          overlay             swarm               
436685dca07d        none                null                local               
[root@cloud01 ~]# docker network inspect my-network
[
    {
        "Name": "my-network",
        "Id": "bhna620fxpuryf2e1te6hui1t",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "10.0.9.0/24",
                    "Gateway": "10.0.9.1"
                }
            ]
        },
        "Internal": false,
        "Containers": null,
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "257",
            "encrypted": ""
        },
        "Labels": null
    }
]

```  
#### 创建mysql服务  
在manager节点创建mysql服务，参数说明：  
**--replicas :** 服务运行的副本个数  
**--name :** 服务名称  
**--network :** 服务运行时所处在的网络  
**--endpoint-mode dnsrr :** 服务发现的方式dns  
**--mount type=bind :** 服务以绑定方式挂载数据目录(挂载前宿主机目录必须先创建)  
**--e :** 针对mysql服务创建时需要的启动参数  
**--tcr:5000/myhub/mysql:5.7-dws** 所运行的镜像

```bash
[root@cloud01 ~]# docker service create \
   --replicas 1 \
   --name mysql \
   --network my-network \
   --endpoint-mode dnsrr \
   --mount type=bind,src=/var/lib/mysql,dst=/var/lib/mysql \
   -e MYSQL_ROOT_PASSWORD=talent \
   -e MYSQL_DATABASE=citydb \
   tcr:5000/myhub/mysql:5.7-dws
33wyod9syd1fcq2ip58872ce6
[root@cloud01 ~]# docker service ls
ID            NAME   REPLICAS  IMAGE                         COMMAND
33wyod9syd1f  mysql  0/1       tcr:5000/myhub/mysql:5.7-dws  
[root@cloud01 ~]# docker service ps mysql 
ID                         NAME     IMAGE                         NODE        DESIRED STATE  CURRENT STATE             ERROR
2nrar9c71olxrc7wo3ab6hxfh  mysql.1  tcr:5000/myhub/mysql:5.7-dws  cloud02  Running        Preparing 28 seconds ago  
[root@cloud01 ~]# docker service ls
ID            NAME   REPLICAS  IMAGE                         COMMAND
33wyod9syd1f  mysql  1/1       tcr:5000/myhub/mysql:5.7-dws 
```
查看cloud02主机上是否运行了mysql数据库服务：

```bash
[root@cloud02 ~]# docker ps -a
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS               NAMES
d168b5948e55        tcr:5000/myhub/mysql:5.7-dws   "docker-entrypoint.sh"   24 seconds ago      Up 22 seconds       3306/tcp            mysql.1.2nrar9c71olxrc7wo3ab6hxfh

```
#### 创建bootService服务  
在manager节点上操作创建3个容器副本的bootService服务：

```bash
[root@cloud01 ~]# docker service create \
   --replicas 3 \
   --name bootService \
   --network my-network \
   --endpoint-mode dnsrr \
   tcr:5000/myhub/bootservice:1.0-dws
5cy0l5cuidlgqjqczwotikvke
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
33wyod9syd1f  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
5cy0l5cuidlg  bootService  2/3       tcr:5000/myhub/bootservice:1.0-dws  
[root@cloud01 ~]# docker service ps bootService
ID                         NAME           IMAGE                               NODE        DESIRED STATE  CURRENT STATE           ERROR
b7va0y4dg01chnhu3hlvzi7o8  bootService.1  tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 7 seconds ago   
8zbgh0rl9w6amhmjxj2rmu627  bootService.2  tcr:5000/myhub/bootservice:1.0-dws  cloud03  Running        Running 22 seconds ago  
3lgnfy11sgs4hj3375o30j5us  bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud03  Running        Running 22 seconds ago  
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
33wyod9syd1f  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
5cy0l5cuidlg  bootService  3/3       tcr:5000/myhub/bootservice:1.0-dws  
[root@cloud01 ~]# docker ps -a
CONTAINER ID        IMAGE                                COMMAND                CREATED             STATUS              PORTS               NAMES
37218abdd4ec        tcr:5000/myhub/bootservice:1.0-dws   "java -jar /app.jar"   41 seconds ago      Up 40 seconds       8888/tcp            bootService.1.b7va0y4dg01chnhu3hlvzi7o8

```
在cloud03上看另外两个容器：  

```bash
[root@cloud03 nginx]# docker ps -a
CONTAINER ID        IMAGE                                COMMAND                CREATED             STATUS              PORTS               NAMES
4eef20ce8f95        tcr:5000/myhub/bootservice:1.0-dws   "java -jar /app.jar"   2 minutes ago       Up 2 minutes        8888/tcp            bootService.2.8zbgh0rl9w6amhmjxj2rmu627
960e905306e9        tcr:5000/myhub/bootservice:1.0-dws   "java -jar /app.jar"   2 minutes ago       Up 2 minutes        8888/tcp            bootService.3.3lgnfy11sgs4hj3375o30j5us

```
#### 创建nginx服务  
在manager节点上操作创建运行2个容器副本的nginx服务:

```bash
[root@cloud01 ~]# docker service create \
  --name nginx \
  --replicas 2 \
  --publish 80:80 \
  --network my-network \
  tcr:5000/myhub/nginx:1.11-dws
5cd0eaow2zp0ytul3muy71gcl
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        0/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  3/3       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
[root@cloud01 ~]# docker service ps nginx
ID                         NAME     IMAGE                          NODE        DESIRED STATE  CURRENT STATE           ERROR
5aiaqlbvn36hn4q1n6vuzbfre  nginx.1  tcr:5000/myhub/nginx:1.11-dws  cloud03  Running        Running 1 seconds ago   
c5y5ppps3i36wpsx5jovo7f8h  nginx.2  tcr:5000/myhub/nginx:1.11-dws  cloud01  Running        Running 11 seconds ago  
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  3/3       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws  
```

### 服务的管理与测试  

- **访问应用测试**  

通过浏览器输入http://cloud01(对就服务的IP),如：
![应用测试](/images/docker/swarm/2-7.png)  

可以多次刷新一下，监控bootService服务打印出来的日志，他会帮你负载到不同的服务容器上，这里就不做说明了。

- **应用异常停止测试**  

找一台运行了bootService应用的机器手动停止容器，docker-swarm会根据你创建时副本的数量为标准，自动新建副本，以达到服务运行的要求，如下：  
cloud02上停止一个bootService应用  

```bash
[root@cloud02 ~]# docker ps -a
CONTAINER ID        IMAGE                                COMMAND                CREATED             STATUS              PORTS               NAMES
c5ade28e1900        tcr:5000/myhub/bootservice:1.0-dws   "java -jar /app.jar"   3 hours ago         Up 3 hours          8888/tcp            bootService.2.9w5fhethqmyob9l27257ijl4k
01aedb0fbdcd        tcr:5000/myhub/bootservice:1.0-dws   "java -jar /app.jar"   3 hours ago         Up 3 hours          8888/tcp            bootService.3.0ixrtpss8jk62iqdq0z8q4z0g
[root@cloud02 ~]# docker stop bootService.3.0ixrtpss8jk62iqdq0z8q4z0g
bootService.3.0ixrtpss8jk62iqdq0z8q4z0g
[root@cloud02 ~]# docker ps -a
CONTAINER ID        IMAGE                                COMMAND                CREATED             STATUS                        PORTS               NAMES
050cf1feb5b1        tcr:5000/myhub/bootservice:1.0-dws   "java -jar /app.jar"   29 seconds ago      Up 28 seconds                 8888/tcp            bootService.3.dwbmcqnjwjlntujy6xzvss4qn
c5ade28e1900        tcr:5000/myhub/bootservice:1.0-dws   "java -jar /app.jar"   3 hours ago         Up 3 hours                    8888/tcp            bootService.2.9w5fhethqmyob9l27257ijl4k
01aedb0fbdcd        tcr:5000/myhub/bootservice:1.0-dws   "java -jar /app.jar"   3 hours ago         Exited (143) 34 seconds ago                       bootService.3.0ixrtpss8jk62iqdq0z8q4z0g
[root@cloud02 ~]# 

```
在管理节点查看服务的状态：  

```bash
[root@cloud01 ~]# docker service ps bootService
ID                         NAME               IMAGE                               NODE        DESIRED STATE  CURRENT STATE            ERROR
0es1d8ia3e1u1budw422wgpyo  bootService.1      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 3 hours ago      
9w5fhethqmyob9l27257ijl4k  bootService.2      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running 3 hours ago      
dwbmcqnjwjlntujy6xzvss4qn  bootService.3      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Ready          Preparing 3 seconds ago  
0ixrtpss8jk62iqdq0z8q4z0g   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Failed 3 seconds ago     "task: non-zero exit (143)"
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  3/3       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
[root@cloud01 ~]# docker service ps bootService
ID                         NAME               IMAGE                               NODE        DESIRED STATE  CURRENT STATE           ERROR
0es1d8ia3e1u1budw422wgpyo  bootService.1      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 3 hours ago     
9w5fhethqmyob9l27257ijl4k  bootService.2      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running 3 hours ago     
dwbmcqnjwjlntujy6xzvss4qn  bootService.3      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running 16 seconds ago  
0ixrtpss8jk62iqdq0z8q4z0g   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Failed 22 seconds ago   "task: non-zero exit (143)"

```

- **宿主机停机扩容，更新测试**：  

现在如果有一台宿主机的容量，或者内存等以达到了峰值，想对这台主机进行停机扩容，或者相关的系统更新等操作，可以修改主机的集群可见性状态  

```bash
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        
f1ocfd3u8na92f2to8txdogop    cloud02  Ready   Active        
[root@cloud01 ~]# docker node update --availability Drain cloud02
cloud02
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        
f1ocfd3u8na92f2to8txdogop    cloud02  Ready   Drain         
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  3/3       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
[root@cloud01 ~]# docker service ps nginx
ID                         NAME     IMAGE                          NODE        DESIRED STATE  CURRENT STATE        ERROR
5aiaqlbvn36hn4q1n6vuzbfre  nginx.1  tcr:5000/myhub/nginx:1.11-dws  cloud03  Running        Running 4 hours ago  
c5y5ppps3i36wpsx5jovo7f8h  nginx.2  tcr:5000/myhub/nginx:1.11-dws  cloud01  Running        Running 4 hours ago  
[root@cloud01 ~]# docker service ps mysql
ID                         NAME     IMAGE                         NODE        DESIRED STATE  CURRENT STATE        ERROR
488dnyp5czuwta2velkeidxxh  mysql.1  tcr:5000/myhub/mysql:5.7-dws  cloud03  Running        Running 4 hours ago  
[root@cloud01 ~]# docker service ps bootService
ID                         NAME               IMAGE                               NODE        DESIRED STATE  CURRENT STATE             ERROR
0es1d8ia3e1u1budw422wgpyo  bootService.1      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 4 hours ago       
9tcl88byfznsez65av2x0hq94  bootService.2      tcr:5000/myhub/bootservice:1.0-dws  cloud03  Running        Running 54 seconds ago    
9w5fhethqmyob9l27257ijl4k   \_ bootService.2  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Shutdown 58 seconds ago   
5spkzjw760w4zbd79qiy1xc0u  bootService.3      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 54 seconds ago    
dwbmcqnjwjlntujy6xzvss4qn   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Shutdown 59 seconds ago   
0ixrtpss8jk62iqdq0z8q4z0g   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Failed about an hour ago  "task: non-zero exit (143)"
```
通过以上测试可以发现运行在cloud02上的容器全部停止，并且swarm集群会自动迁移cloud02上的容器到其它宿主机上。这样我们就可以在cloud02上进行硬件升级操作，等升级完后，可以进行恢复操作，如：  

```
[root@cloud01 ~]# docker node update --availability Active cloud02
cloud02
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        
f1ocfd3u8na92f2to8txdogop    cloud02  Ready   Active        
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  3/3       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
[root@cloud01 ~]# docker service ps nginx
ID                         NAME     IMAGE                          NODE        DESIRED STATE  CURRENT STATE        ERROR
5aiaqlbvn36hn4q1n6vuzbfre  nginx.1  tcr:5000/myhub/nginx:1.11-dws  cloud03  Running        Running 5 hours ago  
c5y5ppps3i36wpsx5jovo7f8h  nginx.2  tcr:5000/myhub/nginx:1.11-dws  cloud01  Running        Running 5 hours ago  
[root@cloud01 ~]# docker service ps mysql
ID                         NAME     IMAGE                         NODE        DESIRED STATE  CURRENT STATE        ERROR
488dnyp5czuwta2velkeidxxh  mysql.1  tcr:5000/myhub/mysql:5.7-dws  cloud03  Running        Running 5 hours ago  
[root@cloud01 ~]# docker service ps bootService
ID                         NAME               IMAGE                               NODE        DESIRED STATE  CURRENT STATE             ERROR
0es1d8ia3e1u1budw422wgpyo  bootService.1      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 5 hours ago       
9tcl88byfznsez65av2x0hq94  bootService.2      tcr:5000/myhub/bootservice:1.0-dws  cloud03  Running        Running 11 minutes ago    
9w5fhethqmyob9l27257ijl4k   \_ bootService.2  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Shutdown 11 minutes ago   
5spkzjw760w4zbd79qiy1xc0u  bootService.3      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 11 minutes ago    
dwbmcqnjwjlntujy6xzvss4qn   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Shutdown 11 minutes ago   
0ixrtpss8jk62iqdq0z8q4z0g   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Failed about an hour ago  "task: non-zero exit (143)"

```
恢复回来的主机cloud02，集群并没有把之前在这台主机上运行的容器恢复，他只是会在之前创建service时，会加大在这台主机运行容器的权重，比如接下来我加大对bootService的副本的个为6个，集群应该会优先在cloud02上运行.

- **调整service的副本数：**  

```bash
[root@cloud01 ~]# docker service scale bootService=6
bootService scaled to 6
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  3/6       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
[root@cloud01 ~]# docker service ps bootService
ID                         NAME               IMAGE                               NODE        DESIRED STATE  CURRENT STATE               ERROR
0es1d8ia3e1u1budw422wgpyo  bootService.1      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 6 hours ago         
9tcl88byfznsez65av2x0hq94  bootService.2      tcr:5000/myhub/bootservice:1.0-dws  cloud03  Running        Running about an hour ago   
9w5fhethqmyob9l27257ijl4k   \_ bootService.2  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Shutdown about an hour ago  
5spkzjw760w4zbd79qiy1xc0u  bootService.3      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running about an hour ago   
dwbmcqnjwjlntujy6xzvss4qn   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Shutdown about an hour ago  
0ixrtpss8jk62iqdq0z8q4z0g   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Failed 2 hours ago          "task: non-zero exit (143)"
df2gl62u120hix9ixmdvxmr7s  bootService.4      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running 12 seconds ago      
8jauwg0g07qgcjwydfil71uei  bootService.5      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running 13 seconds ago      
f4xwd11aof6rxxf1alovq8ulp  bootService.6      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running 12 seconds ago      
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  6/6       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws  
```
可以看到在增加3个bootService的副本时，他把三个副本都在cloud02上运行


- **service在线滚动升级：**  
- [x] 升级之前需要修改代码程序，比如：  

![更新代码](/images/docker/swarm/2-8.png)

- [x] 重新build镜像
```bash
[root@cloud03 bootService]# docker build -t tcr:5000/myhub/bootservice:2.0-dws .
Sending build context to Docker daemon 57.54 MB
Step 1 : FROM java:openjdk-8-jre-alpine
 ---> ed933d9cbb9b
Step 2 : MAINTAINER troylc <troylc@163.com>
 ---> Running in 1541be2de9a0
 ---> 045d03ddb174
Removing intermediate container 1541be2de9a0
Step 3 : ADD springboot_controller-1.0-SNAPSHOT.jar /app.jar
 ---> 7cdbe5b89ad9
Removing intermediate container 812ee39672ec
Step 4 : EXPOSE 8888
 ---> Running in f606dddc345b
 ---> 5c3d29d21ffc
Removing intermediate container f606dddc345b
Step 5 : ENTRYPOINT java -jar /app.jar
 ---> Running in 21950f69009e
 ---> 4c051321084e
Removing intermediate container 21950f69009e
Successfully built 4c051321084e
[root@cloud03 bootService]# docker images
REPOSITORY                   TAG                    IMAGE ID            CREATED             SIZE
tcr:5000/myhub/bootservice   2.0-dws                4c051321084e        8 seconds ago       136.6 MB
tcr:5000/myhub/nginx         1.11-dws               ac60c7d1de84        8 hours ago         181.5 MB
tcr:5000/myhub/bootservice   1.0-dws                efde01242708        8 hours ago         136.6 MB
tcr:5000/myhub/mysql         5.7-dws                ab71fac9c8e0        23 hours ago        383.4 MB
nginx                        latest                 abf312888d13        2 days ago          181.5 MB
tcr:5000/myhub/dmpmysql      latest                 a0af62341076        7 days ago          374.1 MB
tcr:5000/myhub/mysql         5.7                    261ed3d3451f        7 days ago          383.4 MB
java                         openjdk-8-jre-alpine   ed933d9cbb9b        13 days ago         107.8 MB
[root@cloud03 bootService]# docker push tcr:5000/myhub/bootservice:2.0-dws
The push refers to a repository [tcr:5000/myhub/bootservice]
0ab32fba5be5: Image successfully pushed 
d1052adb22b9: Already exists 
30125717c842: Already exists 
011b303988d2: Already exists 
Pushing tag for rev [4c051321084e] on {http://tcr:5000/v1/repositories/myhub/bootservice/tags/2.0-dws}
```
为了跟踪查询我们bootService服务在滚动更新中的状态, 我们新开一个ssh窗口连接cloud01, 执行watch命令, 监控worker服务状态:
如下命令可以打开一个监控窗口, 每秒监控我们worker服务的状态变化:

```bash
[root@cloud01 ~]# watch -n1 "docker service ps bootService | grep -v Shutdown.*Shutdown"
Every 1.0s: docker service ps bootService | grep -v Shutdown.*Shutdown                                                                                                           Thu Dec  1 17:52:55 2016

ID                         NAME               IMAGE                               NODE        DESIRED STATE  CURRENT STATE              ERROR
0es1d8ia3e1u1budw422wgpyo  bootService.1      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 7 hours ago
9tcl88byfznsez65av2x0hq94  bootService.2      tcr:5000/myhub/bootservice:1.0-dws  cloud03  Running        Running 2 hours ago
5spkzjw760w4zbd79qiy1xc0u  bootService.3      tcr:5000/myhub/bootservice:1.0-dws  cloud01  Running        Running 2 hours ago
0ixrtpss8jk62iqdq0z8q4z0g   \_ bootService.3  tcr:5000/myhub/bootservice:1.0-dws  cloud02  Shutdown       Failed 4 hours ago         "task: non-zero exit (143)"
df2gl62u120hix9ixmdvxmr7s  bootService.4      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running about an hour ago
8jauwg0g07qgcjwydfil71uei  bootService.5      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running about an hour ago
f4xwd11aof6rxxf1alovq8ulp  bootService.6      tcr:5000/myhub/bootservice:1.0-dws  cloud02  Running        Running about an hour ago

```

下面我们有了新bootService服务的镜像2.0-dws, 我们来滚动更新我们的6个bootService:  

这里只用docker service update 命令:  
**--update-parallelism**指定每次update的容器数量  
**--update-delay** 每次更新之后的等待时间.  
**--image**后面跟服务镜像名称  

```
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  6/6       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
[root@cloud01 ~]# docker service update bootService --update-parallelism 2 --update-delay 5s --image tcr:5000/myhub/bootservice:2.0-dws
bootService
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  6/6       tcr:5000/myhub/bootservice:2.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws        
[root@cloud01 ~]# 

```
执行完后，查看上面打开的监控情况，发现他会按照你指定的更新参数，升级bootService服务。  
打开页面验证  
![验证2.0版本](/images/docker/swarm/2-9.png)

- **容器服务回滚**  

如果我们发现, 新版本bootService有问题希望回滚怎么办呢.
很简单, 跟上面更新的命令一样啊, 直接只用v0.1版本的镜像就可以了.
上面我们演示了滚动更新, 如果你希望一次性都更新或者回滚呢, 更简单了, 不加参数就行了.

下面我们一次性回滚所有bootService服务到1.0版本

```
[root@cloud01 ~]# docker service update bootService --image tcr:5000/myhub/bootservice:1.0-dws
bootService
[root@cloud01 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE                               COMMAND
5cd0eaow2zp0  nginx        2/2       tcr:5000/myhub/nginx:1.11-dws       
5v4kajpztp35  bootService  6/6       tcr:5000/myhub/bootservice:1.0-dws  
6zpecsjj6ttm  mysql        1/1       tcr:5000/myhub/mysql:5.7-dws       
```

可以看到bootService服务都回滚到1.0-dws版本了.


