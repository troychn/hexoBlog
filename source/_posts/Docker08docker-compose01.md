---
title: docker-compose部署swarm服务(docker1.13.1)
toc: true
date: 2017-02-25 19:18:23 Sunday
updated: 2017-25-19 19:49:34 Sunday
categories: [docker-compose]
tags: [docker,swarm,docker-compose]

---

在2017年发布的 Docker 1.13版本中的Docker Compose v3 规范，已经全面支持 Swarm mode 概念。而且从 1.13 开始，Docker 命令行工具支持直接使用 v3 版本的 docker-compose.yml 通过docker stack deploy **进行部署管理，这大大简化了容器编排使用的复杂性。

### 部署环境

IP | 主机名 | 角色 | 安装软件  
---|---|---|---  
172.193.6.222 | cloud01 | manager | centos7.2/docker1.13/docker-compose1,11
172.193.6.223 | cloud02 | worker | centos7.2/docker1.13/docker-compose1,11
172.193.6.224 | cloud03 | worker | centos7.2/docker1.13/docker-compose1,11  

安装请参考 [docker1.13.x及docker-compose1.11.x的安装与升级](http://www.toutiao.com/i6385354267686863361/)  

### 服务部署  
服务部署示意图请参考 [swarm集群服务部署与维护](http://www.toutiao.com/i6384915440631546370/) 如下图所示：  
![image](http://www.troylc.cc/images/docker/swarm/2-4.png)

### docker-compose在swarm模式中部署服务  
#### compose文件定义：  

```
#docker-compose v3版本规范
version: "3.1"

services:
  mysql:
    image: "test:5000/myhub/mysql:5.7-dws"
    #container_name: dcmysql
    networks:
      - my-overlay-network
    deploy:
      update_config:
        parallelism: 1
        delay: 2s
      restart_policy:
        condition: on-failure
    environment:
      MYSQL_ROOT_PASSWORD: talent
      MYSQL_DATABASE: citydb
    volumes:
      - /nfsdata/data/mysql:/var/lib/mysql
    expose:
     - "3306"
  bootService:
    image: "test:5000/myhub/bootservice:2.0-dws"
    networks:
      - my-overlay-network
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 2s
      restart_policy:
        condition: on-failure
    expose:
      - "8888" 
  nginx:
    image: "test:5000/myhub/nginx:1.11-dws"
    networks:
      - my-overlay-network
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 2s
      restart_policy:
        condition: on-failure  
    ports:
      - "172.19.6.223:80:80"
networks:
  my-overlay-network:
    driver: overlay
    ipam:
      config:
        - subnet: "10.0.7.0/24"
```
**说明：**  
v3 中引入了 deploy 指令，可对Swarm mode中服务部署的进行细粒度控制，包括  
- resources：定义  cpu_shares, cpu_quota, cpuset, mem_limit, memswap_limit 等容器资源控制。（v1/v2中相应指令不再支持）
- mode：支持 global 和 replicated (缺省) 模式的服务；
- replicas：定义 replicated 模式的服务的复本数量
- placement：定义服务容器的部署放置约束条件
- update_config：定义服务的更新方式
- restart_policy：定义服务的重启条件 （v1/v2中restart指令不再支持）
- service：定义服务的标签

#### 部署服务的操作  
Swarm模式允许创建一个Docker Engines集群。在1.13版本中， docker stack deploy命令可以用来部署一个Compose文件到Swarm模式。Docker Compose带给我们多容器应用，  

命令 | 说明
---|---
docker stack deploy --compose-file=docker-compose.yml dws | 启动服务通过compose文件 stack命名为dws
docker service scale xxx=n | 伸缩服务,需要指定服务名称和伸缩副本数  
docker stack rm | 删除服务


#### 命令行操作

```bash
[root@tsccloud01 docker-compose]# docker stack deploy --compose-file=docker-compose.yml dws
Ignoring deprecated options:

expose: Exposing ports is unnecessary - services on the same network can access each other's containers on any port.
Creating service dws_nginx
Creating service dws_mysql 
Creating service dws_bootService

[root@tsccloud01 docker-compose]# docker stack ps dws
ID            NAME               IMAGE                               NODE        DESIRED STATE  CURRENT STATE               ERROR  PORTS
khfdjfs5wycw  dws_bootService.1  tcr:5000/myhub/bootservice:1.0-dws  tsccloud02  Running        Running about a minute ago         
omo3n8ftk8g3  dws_nginx.1        tcr:5000/myhub/nginx:1.11-dws       tsccloud03  Running        Running about a minute ago         
e8mmigd1tj9e  dws_mysql.1        tcr:5000/myhub/mysql:5.7-dws        tsccloud01  Running        Running about an hour ago          
xzlubd2fou4f  dws_bootService.2  tcr:5000/myhub/bootservice:1.0-dws  tsccloud03  Running        Running about a minute ago         
5ws9o7xwmbpd  dws_nginx.2        tcr:5000/myhub/nginx:1.11-dws       tsccloud02  Running        Running about a minute ago         
5gzncbpq9626  dws_bootService.3  tcr:5000/myhub/bootservice:1.0-dws  tsccloud01  Running        Running about a minute ago

[root@tsccloud01 docker-compose]# docker service ls
ID            NAME             MODE        REPLICAS  IMAGE
b63us2e2eksg  dws_nginx        replicated  2/2       tcr:5000/myhub/nginx:1.11-dws
kqo1g3s1z0mr  dws_bootService  replicated  3/3       tcr:5000/myhub/bootservice:1.0-dws
rvlfnogtgtso  dws_mysql        replicated  1/1       tcr:5000/myhub/mysql:5.7-dws

```

通过以上操作，和之前用命令在swarm集群中部署一样，

