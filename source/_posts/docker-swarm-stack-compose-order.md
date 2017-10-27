---
title: docker容器及swarm集群之compose健康检查与容器依赖
toc: true   // 在文章侧边增加文章目录
date: 10/27/2017 5:26:15 PM 
updated: 10/27/2017 5:26:20 PM 
categories: [docker swarm,docker,docker-compose]
tags: [docker,swarm,docker-compose,linux]

---

在很多情况下，docker容器在compose中的启动顺序，决定了docker容器的健康状态，如果启动时所依赖的容器没有启动，所依赖的容器也会出现问题，这样整体应用都会引起连锁反应。虽然docker可以使用[depends_on](https://docs.docker.com/compose/compose-file/#depends_on)选项控制服务启动顺序 。撰写总是依赖顺序；但是，Compose不会等到容器“准备好”也就是说他不会等待依赖的容器完全运行完，再启动当前容器，而仅只是在所依赖的容器之前运行。  
最好的解决方案是在启动时或者是由于任何原因丢失连接时，在应用程序代码中执行检查与尝试连接所依赖的服务的。但是，如果您不需要如此级别的弹性检查，根据环境，可以有以下几种解决方案：

# compose 部署单机容器启动顺序解决方案
在compose2.1之后的文档规范中，提供了通过一个健康检查来解决两个服务之前的依赖。如：

```yaml
version: '2.1'
services:
  web:
    images: nodejs
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
  redis:
    image: redis
    healthcheck:
      test: ["CMD-SHELL", "nc -v -w 5 localhost -z 6379 || exit 1"]
  db:
    image: mysql
    healthcheck:
      test: ["CMD-SHELL", "nc -v -w 5 localhost -z 3306 || exit 1"]
      
```
* 说明：  
这里的redis和db,需要在基础镜像中添加nc命令，让这两个服务容器启动时一直等到具体服务端口启动，再来改变容器的健康状态。所依赖的服务会根据被依赖服务的健康状态来决定启动自己的服务。这就是单机的情况下来操作容器的启动顺序。

# compose 部署swarm集群的容器启动顺序解决方案
很多时候我们的服务是一堆服务器组成的集群模式，这里主要解决一下swarm mode模式下的方案，诚然[官方对于容器的启动顺序的解决方案](https://docs.docker.com/compose/startup-order/)是：

* Use a tool such as [wait-for-it](https://github.com/vishnubob/wait-for-it), [dockerize](https://github.com/jwilder/dockerize), or sh-compatible [wait-for](https://github.com/Eficode/wait-for). These are small wrapper scripts which you can include in your application’s image and will poll a given host and port until it’s accepting TCP connections.  

官方主要说明的是第一种解决方案wait-for-it，下面主要是解决第二种方案dockerize，这种方案也是需要从github上下载dockerize工具来进行操作，镜像中需要把dockerize工具build到具体的镜像中，让容器运行时，可以通过dockerize的工具来进行检查。以下是具体的两个应用服务，需要依赖mysql、kafka、redis。web服务nginx依赖具体的两个应用服务。具体compose文件如下：

```yaml
version: "3"
services:
  zookeeper:       #zookeeper服务，主要是协助kafka消息中心的
    image: zookeeper:3.4.9
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    environment:
      - "TZ=Asia/Shanghai"
    networks:
      - dmp-net
    healthcheck:
      test: ["CMD-SHELL", "nc -v -w 5 localhost -z 2181 || exit 1"]
      interval: 2m30s #健康检查的时间间隔，默认为 30s。
      timeout: 15s    #健康检查的超时时间，默认为 30s。
      retries: 3    #连续几次健康检查失败即认为容器不健康，默认为 3。

  kafka:      #kafka消息中心，
    image: kafka:0.10.1.0
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - dmp-net
    environment:
      - 'KAFKA_ADVERTISED_HOST_NAME=kafka'
      - 'KAFKA_ADVERTISED_PORT=9092'
      - 'KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181'
      - "TZ=Asia/Shanghai"
    ports:
      - "9092:9092"
    healthcheck:
      test: ["CMD-SHELL", "nc -v -w 5 localhost -z 9092 || exit 1"]
      interval: 2m30s #健康检查的时间间隔，默认为 30s。
      timeout: 15s    #健康检查的超时时间，默认为 30s。
      retries: 3    #连续几次健康检查失败即认为容器不健康，默认为 3。
    command: ["dockerize","-wait","tcp://zookeeper:2181","-timeout","300s","start-kafka.sh"]

  mysql:    # 数据库服务
    image: mysql:latest
    deploy:
      placement:
        constraints:
          - "node.role==manager"
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - dmp-net
    volumes:
      - /nfs-data/troylc/mysql:/var/lib/mysql
    environment:
      - "MYSQL_ROOT_PASSWORD=troylc"
      - "TZ=Asia/Shanghai"
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD-SHELL", "nc -v -w 5 localhost -z 3306 || exit 1"]
      interval: 1m30s #健康检查的时间间隔，默认为 30s。
      timeout: 15s    #健康检查的超时时间，默认为 30s。
      retries: 3    #连续几次健康检查失败即认为容器不健康，默认为 3。

  redis:    # redis服务
    image: redis:latest
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - dmp-net
    environment:
      - "TZ=Asia/Shanghai"
    healthcheck:
      test: ["CMD-SHELL", "nc -v -w 5 localhost -z 6379 || exit 1"]
      interval: 1m30s #健康检查的时间间隔，默认为 30s。
      timeout: 15s    #健康检查的超时时间，默认为 30s。
      retries: 3    #连续几次健康检查失败即认为容器不健康，默认为 3。

xxxservice:     
    image: tcr:5000/myhub/dmpservice:2.3.4.1
    deploy:
      replicas: 3   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    environment:
      - "TZ=Asia/Shanghai"
      - "TOMCAT_PASS=xxxservice"
    volumes:
      - /etc/localtime:/etc/localtime:ro
    networks:
      - dmp-net
    command: ["dockerize","-wait","tcp://mysql:3306","-wait","tcp://redis:6379","-wait","tcp://kafka:9092","-timeout","300s","/run.sh"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/xxxservice-web/"]
      interval: 1m30s #健康检查的时间间隔，默认为 30s。
      timeout: 15s   #健康检查的超时时间，默认为 30s。
      retries: 3    #连续几次健康检查失败即认为容器不健康，默认为 3。
      
  upfileservice:
    image: tcr:5000/myhub/ufile:2.2.4
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - dmp-net
    environment:
      - "TZ=Asia/Shanghai"
      - "TOMCAT_PASS=upfileservice"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /nfs-data/troylc/file:/nfs-data/troylc/file
    command: ["dockerize","-wait","tcp://mysql:3306","-timeout","300s","catalina.sh", "run"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/upfileservice-web/"]
      interval: 1m30s #健康检查的时间间隔，默认为 30s。
      timeout: 15s   #健康检查的超时时间，默认为 30s。
      retries: 3    #连续几次健康检查失败即认为容器不健康，默认为 3。

  nginx:
    image: nginx
    deploy:
      replicas: 1   #定义 replicated 模式的服务的复本数量
      update_config:
        parallelism: 1    #每次更新复本数量
        delay: 2s       #每次更新间隔
      restart_policy:
        condition: on-failure     #定义服务的重启条件
    networks:
      - dmpbase_dmp-net
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /nfs-data/troylc/data:/troylc/data
    environment:
      - "TZ=Asia/Shanghai"
    ports:
      - "192.168.100.202:50000:50000"
      - "192.168.100.202:443:443"
    command: ["dockerize","-wait","http://xxxservice:8080","-wait","http://upfileservice:8080","-timeout","300s","nginx","-g","daemon off;"]
    healthcheck:
      test: ["CMD-SHELL", "dockerize -wait tcp://localhost:443 -timeout 3s || exit 1"]
      interval: 1m30s #健康检查的时间间隔，默认为 30s。
      timeout: 15s   #健康检查的超时时间，默认为 30s。
      retries: 3    #连续几次健康检查失败即认为容器不健康，默认为 3。

networks:
  dmp-net:
    driver: overlay
    ipam:
      config:
        - subnet: "10.0.1.0/24"

```
* 从文档中我们可以看出，kafka依赖了zookeeper,xxxservice服务依赖了mysql、redis、kafka，upfileservice依赖了mysql。nginx服务依赖了xxxservice和upfileservice这两个应用服务。

# 容器启动顺序问题
有时候容器的服务启动时，服务的端口启动了，但是具体的服务，还没有完全启动，这同样会操作所依赖的服务启动时，有可能出错。所以最好的解决方案是在具体的所需要依赖的服务中去自己构建所需要依赖服务的健康检查以及连通性检查。




