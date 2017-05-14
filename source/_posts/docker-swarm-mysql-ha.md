---
title: docker-swarm下部署mysql高可用(主从复制)  
toc: true  
date: 2017-05-14 19:18:23 Sunday  
updated: 2017-05-14 19:49:34 Sunday  
categories: [mysql]  
tags: [mysql,docker,swarm,docker-compose]  

---

在考虑MySQL数据库的高可用架构时，主要考虑以下几方面：
 
- 如果数据库发生了宕机或者意外中断等故障，能尽快恢复数据库的可用性，尽可能的减少停机时间，保证业务不会因为数据库的故障而中断。
- 用作备份、只读副本等功能的非主节点的数据应该和主节点的数据实时或者最终保持一致。
- 当业务发生数据库切换时，切换前后的数据库内容应当一致，不会因为数据缺失或者数据不一致而影响业务。
 
以下是在docker swarm环境中部署一个mysql的高可用环境，通过MaxScale的一个MySQL数据中间件，实现读写分离，并根据主从状态实现写库的自动切换。

## 环境准备

docker swarm环境

IP | 主机名 | 角色
---|---|--- 
172.19.6.xxx | cloud01 | manager
172.19.6.xxx | cloud02 | worker
172.19.6.xxx | cloud03 | worker

swarm集群的安装与管理，参考：  
[docker-swarm创建与管理集群](http://www.troylc.cc/docker/2017/02/07/Docker07docker-swarm01.html)  
[docker-swarm集群服务部署与维护](http://www.troylc.cc/docker/2017/02/19/Docker07docker-swarm02.html)

查看swarm节点情况：  

```bash
[root@docker-master01 ~]# docker node ls
ID                           HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
60w2g6sd30iep1865qbi8tt87    docker-node02    Ready   Active        
uy5jeecverzi7go34p5pkenz8 *  docker-master01  Ready   Active        Leader
xgwetr888pf6ygkb3rh6zfjqx    docker-node01    Ready   Active 
```


## 在swarm上部署myql高可用  
- 创建一个swarm的全局网络：

```bash
[root@docker-master01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
04b7b4731612        bridge              bridge              local
626ec1305a55        docker_gwbridge     bridge              local
2a5147ecaf4c        host                host                local
p4wh41oefk4d        ingress             overlay             swarm
6fc1d335c2e8        none                null                local
[root@docker-master01 ~]# docker network create -d overlay dbmysqlnet
m26c1w6c1tilycfv8h6u1pogc
[root@docker-master01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
04b7b4731612        bridge              bridge              local
m26c1w6c1til        dbmysqlnet          overlay             swarm
626ec1305a55        docker_gwbridge     bridge              local
2a5147ecaf4c        host                host                local
p4wh41oefk4d        ingress             overlay             swarm
6fc1d335c2e8        none                null                local
[root@docker-master01 ~]# 

```

- 创建一个mysql cluster集群，设置副本为1，--replicas=1，当副本为1时 mariadb-cluster镜像为这个实例自动变成引导节点。

```bash
docker service create --name mysqldbcluster \
--network dbmysqlnet \
--replicas=1 \
--env DB_SERVICE_NAME=mysqldbcluster \
--env MYSQL_ROOT_PASSWORD=rootpass \
--env MYSQL_DATABASE=mytestdb \
--env MYSQL_USER=mysqldbuser \
--env MYSQL_PASSWORD=mysqldbpass \
toughiq/mariadb-cluster
```
注：所提供的服务名称--name必须匹配环境变量DB_SERVICE_NAME镶有--env DB_SERVICE_NAME，并为数据库设置数据库名、root用户名的密码，以及为数据库单独创建的用户名和密码。


```bash
[root@docker-master01 ~]# docker service create --name mysqldbcluster \
> --network dbmysqlnet \
> --replicas=1 \
> --env DB_SERVICE_NAME=mysqldbcluster \
> --env MYSQL_ROOT_PASSWORD=rootpass \
> --env MYSQL_DATABASE=mytestdb \
> --env MYSQL_USER=mysqldbuser \
> --env MYSQL_PASSWORD=mysqldbpass \
> toughiq/mariadb-cluster
v7azlb9yvbv8uumoisddtd4pu
[root@docker-master01 ~]# docker service ls
ID            NAME            MODE        REPLICAS  IMAGE
v7azlb9yvbv8  mysqldbcluster  replicated  1/1       toughiq/mariadb-cluster:latest
[root@docker-master01 ~]# docker service ps mysqldbcluster
ID            NAME              IMAGE                           NODE           DESIRED STATE  CURRENT STATE           ERROR  PORTS
7vwdy4qz8k5b  mysqldbcluster.1  toughiq/mariadb-cluster:latest  docker-node02  Running        Running 27 seconds ago         
[root@docker-master01 ~]# 

```
再通过更新mysqldbcluster服务扩展mysql数据库，这个通过更新mysqldbcluster服务的2个副本的启动将出现在“cluster join”-mode中

```bash
[root@docker-master01 ~]# docker service scale mysqldbcluster=3
mysqldbcluster scaled to 3
[root@docker-master01 ~]# docker service ps mysqldbcluster
ID            NAME              IMAGE                           NODE             DESIRED STATE  CURRENT STATE          ERROR  PORTS
7vwdy4qz8k5b  mysqldbcluster.1  toughiq/mariadb-cluster:latest  docker-node02    Running        Running 5 minutes ago         
i0qw4p83ke6o  mysqldbcluster.2  toughiq/mariadb-cluster:latest  docker-master01  Running        Running 7 seconds ago         
wz5u3sdocykj  mysqldbcluster.3  toughiq/mariadb-cluster:latest  docker-node01    Running        Running 7 seconds ago         
[root@docker-master01 ~]# docker service ls
ID            NAME            MODE        REPLICAS  IMAGE
v7azlb9yvbv8  mysqldbcluster  replicated  3/3       toughiq/mariadb-cluster:latest
[root@docker-master01 ~]# 

```
- 创建MaxScale代理服务并连接到mysqldbcluster   

由于Swarm提供了一个负载平衡器，因此使用该Docker Swarm启用的数据库集群不需要MaxScale Proxy服务。因此，可以使用负载均衡器DNS名称连接到集群,上面运行的例子就是mysqldbcluster。它在同一个名字，由启动时提供--name。
但是MaxScale提供了一些关于负载平衡数据库流量的附加功能。它是获取有关群集状态的信息的简单方法。

```
docker service create --name maxscale \
--network dbmysqlnet \
--env DB_SERVICE_NAME=mysqldbcluster \
--env ENABLE_ROOT_USER=1 \
--publish 3306:3306 \
toughiq/maxscale
```  
要通过MaxScale禁用root对数据库的访问，只需设置--env ENABLE_ROOT_USER=0或删除该行即可。
默认情况下禁用根访问。

```bash
[root@docker-master01 ~]# docker service create --name maxscale \
> --network dbmysqlnet \
> --env DB_SERVICE_NAME=mysqldbcluster \
> --env ENABLE_ROOT_USER=1 \
> --publish 3306:3306 \
> toughiq/maxscale
v50kns05a9f5ufb028g8u3zsr
[root@docker-master01 ~]# docker service ls
ID            NAME            MODE        REPLICAS  IMAGE
v50kns05a9f5  maxscale        replicated  1/1       toughiq/maxscale:latest
v7azlb9yvbv8  mysqldbcluster  replicated  3/3       toughiq/mariadb-cluster:latest
[root@docker-master01 ~]# docker service ps maxscale
ID            NAME        IMAGE                    NODE           DESIRED STATE  CURRENT STATE           ERROR  PORTS
unny1x1y10hs  maxscale.1  toughiq/maxscale:latest  docker-node02  Running        Running 18 seconds ago      


```
可以查看到maxscale是在集群的docker-node02上，到这台主机上查看一下mysql集群的情况如：

```
[root@docker-node02 ~]# docker exec -it maxscale.1.unny1x1y10hsaxkm1kotmg8oq maxadmin -pmariadb list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
10.0.0.4           | 10.0.0.4        |  3306 |           0 | Slave, Synced, Running
10.0.0.3           | 10.0.0.3        |  3306 |           0 | Slave, Synced, Running
10.0.0.5           | 10.0.0.5        |  3306 |           0 | Master, Synced, Running
-------------------+-----------------+-------+-------------+--------------------
[root@docker-node02 ~]# 

```
从上面的部署，mysql高可用集群基本上完成，我们可以做一下容灾的测试，比如停掉一台宿主机，或者停止一台集群节点上的容器，在通过以上命令，看看有什么结果，可以自行测试看看。  

## 测试mysql高可用情况
- 首页查看master节点是那台机器，通过上面，可以看出ip为10.0.0.5为主节点，通过docker inspect,查看主节点在那台机器上，然后重启这台宿主机上的mysql容器，然后swarm集群会马上启动一个mysql容器，以达到mysql集群的副本数，

```
[root@docker-node01 ~]# docker ps
CONTAINER ID        IMAGE                                                                                             COMMAND                  CREATED             STATUS              PORTS                               NAMES
8a4d7deff9db        toughiq/mariadb-cluster@sha256:09213e60b57734206a376d42f87c1aa83163b53745736fc566fd460578fd3461   "docker-entrypoint..."   26 minutes ago      Up 26 minutes       3306/tcp, 4444/tcp, 4567-4568/tcp   mysqldbcluster.3.wz5u3sdocykj1y0e47bbze1ti
[root@docker-node01 ~]# docker stop 26d2e2a34089
26d2e2a34089
[root@docker-node01 ~]# docker ps
CONTAINER ID        IMAGE                                                                                             COMMAND                  CREATED             STATUS              PORTS                               NAMES
2872c109eee8        toughiq/mariadb-cluster@sha256:09213e60b57734206a376d42f87c1aa83163b53745736fc566fd460578fd3461   "docker-entrypoint..."   24 seconds ago      Up 18 seconds       3306/tcp, 4444/tcp, 4567-4568/tcp   mysqldbcluster.3.idh7t6r9h4nxivz1bm0hn7bi7
```

- 在通过swarm 管理节点，查看mysql集群的部署情况，通过maxscale中间件，查看mysql高可用情况  
swarm manager节点查看：  

```
[root@docker-master01 ~]# docker service ls
ID            NAME            MODE        REPLICAS  IMAGE
v50kns05a9f5  maxscale        replicated  1/1       toughiq/maxscale:latest
v7azlb9yvbv8  mysqldbcluster  replicated  3/3       toughiq/mariadb-cluster:latest
[root@docker-master01 ~]# docker service ps mysqldbcluster
ID            NAME                  IMAGE                           NODE             DESIRED STATE  CURRENT STATE           ERROR                             PORTS
7vwdy4qz8k5b  mysqldbcluster.1      toughiq/mariadb-cluster:latest  docker-node02    Running        Running 39 minutes ago                                    
i0qw4p83ke6o  mysqldbcluster.2      toughiq/mariadb-cluster:latest  docker-master01  Running        Running 33 minutes ago                                    
boolbkf16bz2  mysqldbcluster.3      toughiq/mariadb-cluster:latest  docker-master01  Running        Running 6 minutes ago                                     
wz5u3sdocykj   \_ mysqldbcluster.3  toughiq/mariadb-cluster:latest  docker-node01    Shutdown       Failed 6 minutes ago    "No such container: mysqldbclu…"  
```  
maxscale中查看

```
[root@docker-node02 ~]# docker exec -it maxscale.1.unny1x1y10hsaxkm1kotmg8oq maxadmin -pmariadb list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
10.0.0.4           | 10.0.0.4        |  3306 |           0 | Master, Synced, Running
10.0.0.3           | 10.0.0.3        |  3306 |           0 | Slave, Synced, Running
10.0.0.5           | 10.0.0.5        |  3306 |           0 | Down
-------------------+-----------------+-------+-------------+--------------------
```
等待swarm集群再恢复所有节点后查看：  

```
[root@docker-node02 ~]# docker exec -it maxscale.1.unny1x1y10hsaxkm1kotmg8oq maxadmin -pmariadb list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
10.0.0.4           | 10.0.0.4        |  3306 |           0 | Master, Synced, Running
10.0.0.3           | 10.0.0.3        |  3306 |           0 | Slave, Synced, Running
10.0.0.5           | 10.0.0.5        |  3306 |           0 | Slave, Synced, Running
-------------------+-----------------+-------+-------------+--------------------
```
