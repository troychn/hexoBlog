---
title: SpringCloud和docker之微服务-eureka(一)
toc: true
date: 2017-03-01 19:18:23 Sunday
updated: 2017-03-01 19:49:34 Sunday
categories: [spring cloud]
tags: [spring-boot,spring cloud,docker,swarm,docker-compose,java]

---
在软件开发中关于服务的讨论呈现出火爆的局面，有人倾向于在系统设计与开发中采用微服务方式实现软件系统的松耦合、跨部门开发；一些公司已经在生产系统中采用了微服务架构，并且取得了良好的效果；下面我们通过spring cloud和docker来构建一个简单的例子，

# spring cloud简介

Spring Cloud是在Spring Boot的基础上构建的，为开发人员提供快速建立分布式系统的有关微服务搭建的一系列的工具，例如：

配置管理（configuration management），服务发现（service discovery），断路器（circuit breakers），智能路由（ intelligent routing），微代理（micro-proxy），控制总线（control bus），一次性令牌（ one-time tokens），全局锁（global locks），领导选举（leadership election），分布式会话（distributed sessions），集群状态（cluster state）  
![spring cloud组件架构图](/images/spring-cloud/eureka/spring-cloud.jpg)  
具体参考：[Spring Cloud 项目主页：http://projects.spring.io/spring-cloud/](http://projects.spring.io/spring-cloud/)

# eureka服务发现

服务发现（Service Discovery）是关键原则之一。手动配置每个客户端或某种形式的约定是很难做的，并且很脆弱。Spring Cloud提供了多种服务发现的实现方式，例如：Eureka、Consul、Zookeeper。这里只讲述基于Eureka的服务发现。

# 案例准备工作

## 开发工具及相关软件准备

环境 | 版本及说明 | 参考地址  
---|---|---
docker | v1.13.1,Docker是一个能够把开发的应用程序自动部署到容器的开源引擎 | [docker、docker-compse最新版本安装](http://www.troylc.cc/docker/2017/01/05/docker04ininstall.html)
doker-compose | v1.11,Docker 官方编排（Orchestration）项目之一，负责快速在集群中部署分布式应 | [docker、docker-compse最新版本安装](http://www.troylc.cc/docker/2017/01/05/docker04ininstall.html)
docker swarm | v1.13.1,Docker Engine 1.12或更高版本中内置了集群管理和编排功能。Swarm模式侧重于微服务架构。| [docker-swarm创建与管理集群](http://www.troylc.cc/docker/2017/02/07/Docker07docker-swarm01.html)
docker registry | registry:latest,用于存储docker镜像的私有仓库 | [registry集成打包上传镜像](http://www.troylc.cc/docker/2017/02/01/Docker06registry-jenkins.html)
spring boot | 1.5.1.RELEASE,是开箱即用，提供一系列大型项目常用的非功能性特征的快速度开发工具 | [spring boot官网](https://projects.spring.io/spring-boot/)
spring cloud | Camden SR5,Spring Cloud 为开发者提供了在分布式系统（如配置管理、服务发现、断路器、智能路由、微代理、控制总线、一次性 Token、全局锁、决策竞选、分布式会话和集群状态）操作的开发工具集 | [spring cloud官网](http://projects.spring.io/spring-cloud/)  
开发工具 | jdk1.8/IntelliJ idea/maven3.3.9 | 


## 创建父项目
首先创建一个父项目（spring-cloud-docker-microservice），这样可以对项目中的Maven依赖进行统一的管理。  

项目结构如下：  
![项目结](/images/spring-cloud/eureka/1.png)

pom.xml如下  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.troylc.cloud</groupId>
	<artifactId>spring-cloud-docker-microservice</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>pom</packaging>

	<name>spring-cloud-docker-microservice</name>
	<description>Demo project for Spring Boot</description>
    <!-- 添加spring boot父项目的依赖，spring cloud是在spring boot基础之上进行开发的，所以需要依赖它-->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

    <modules>
        <module>microservice-eureka-service</module>
    </modules>

    <properties>
       <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
       <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
       <java.version>1.8</java.version>  
       <docker.image.prefix>myhub</docker.image.prefix><!--配置镜像仓库的属性-->
       <docker.repostory>tcr:5000</docker.repostory><!--配置镜像仓库的对应的地址与端口-->
    </properties>
    <!--添加spring cloud依赖-->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Camden.SR5</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

	<build>
		<plugins>
            <plugin>
                <!-- 配置maven install 跳过test,相当于命令：$mvn install -Dmaven.test.skip = true-->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
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
</project>

```

## 创建microservice-eureka-service子项目

- 创建一个Maven工程（microservice-eureka-service），并在pom.xml中加入如下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<!--引用父项目的pom-->
    <parent>
        <groupId>com.troylc.cloud</groupId>
        <artifactId>spring-cloud-docker-microservice</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.troylc.cloud</groupId>
    <artifactId>microservice-eureka-service</artifactId>
    <version>0.0.1</version>
    <packaging>jar</packaging>

    <name>microservice-eureka-service</name>
    <description>project for Spring cloud</description>
    <properties>
        <!--程序入口-->
        <start-class>com.troylc.cloud.EurekaServiceApplication</start-class>
    </properties>
	<dependencies>
        <!--eurekaService必须引用的包-->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
		</dependency>
        <!--配置需要认证的eurekaservice所需要引用的包-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <executions>
                    <!--设置在执行maven 的install时构建镜像-->
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
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```
- 编写Spring Boot启动程序EurekaServiceApplication：通过@EnableEurekaServer申明一个注册中心  
![eurekaServiceApplication](/images/spring-cloud/eureka/2.png)

```java
package com.troylc.cloud;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

/**
 * 使用Eureka做服务发现.
 * @author troylc
 */
@SpringBootApplication
@EnableEurekaServer
public class EurekaServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServiceApplication.class, args);
	}
}
```
- 在spring boot的配置文件 application.yml中配置Authenticating、HA高可用及其它相关配置：

```
spring:
  application:
    name: microservice-eureka-services
#  profiles:
#    active: eurekaService1
security:
  basic:
    enabled: true     # 开启基于HTTP basic的认证
  user:
    name: eadmin      # 配置登录的账号是user
    password: eadmin123   # 配置登录的密码是eadmin123

# 配置三个实例的eureka高可用配置，如果是在swarm集群中服务，请把swarm中我service名称部署为三个，分别为：eurekaService1,eurekaService2,eurekaService3
---
spring:
  profiles: eurekaService1
server:
  port: 9511                    # 指定该Eureka实例的端口
eureka:
  instance:
    hostname: eurekaService1         # 指定该Eureka实例的主机名
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ipAddress}:${server.port} # 将Instance ID设置成IP:端口的形式
  client:
    serviceUrl:    #设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。
      defaultZone: http://eadmin:eadmin123@eurekaService2:9512/eureka/,http://eadmin:eadmin123@eurekaService3:9513/eureka/
---
spring:
  profiles: eurekaService2
server:
  port: 9512                    # 指定该Eureka实例的端口
eureka:
  instance:
    hostname: eurekaService2         # 指定该Eureka实例的主机名
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ipAddress}:${server.port}
  client:
    serviceUrl:    #设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。
      defaultZone: http://eadmin:eadmin123@eurekaService1:9511/eureka/,http://eadmin:eadmin123@eurekaService3:9513/eureka/
---
spring:
  profiles: eurekaService3
server:
  port: 9513                    # 指定该Eureka实例的端口
eureka:
  instance:
    hostname: eurekaService3         # 指定该Eureka实例的主机名
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ipAddress}:${server.port}
  client:
    serviceUrl:    #设置与Eureka Server交互的地址，查询服务和注册服务都需要依赖这个地址。默认是http://localhost:8761/eureka ；多个地址可使用 , 分隔。
      defaultZone: http://eadmin:eadmin123@eurekaService2:9512/eureka/,http://eadmin:eadmin123@eurekaService1:9511/eureka/
```
- 在父项目的根目录下创建一个docker-compose.yml文件，用于把eurekaService部署在docker swarm集群中,compose内容如下：

```
version: "3"
services:
  eurekaService1:      # 默认情况下，其他服务可以使用服务名称连接到该服务。因此，对于peer2的节点，它需要连接http://peer1:8761/eureka/，因此需要配置该服务的名称是peer1。
    image: tcr:5000/myhub/microservice-eureka-service:0.0.1
    networks:
      - eureka-net
    ports:
      - "9511:9511"
    environment:
      - spring.profiles.active=eurekaService1
  eurekaService2:
    image: tcr:5000/myhub/microservice-eureka-service:0.0.1
    networks:
      - eureka-net
    ports:
      - "9512:9512"
    environment:
      - spring.profiles.active=eurekaService2
  eurekaService3:
      image: tcr:5000/myhub/microservice-eureka-service:0.0.1
      networks:
        - eureka-net
      ports:
        - "9513:9513"
      environment:
        - spring.profiles.active=eurekaService3
networks:
  eureka-net:
    driver: overlay
```

## 通过maven插件构建docker镜像  
通过idea中的maven插件构建eurekService的docker镜像到私有仓库
![构建镜像](/images/spring-cloud/eureka/3.png)  
在docker环境从私有仓库中下载上面构建的镜像  
![下载镜像](/images/spring-cloud/eureka/4.png)

参考：  
[jenkins-registry持续集成-jenkins-registry安装与数据迁移(一)](http://www.troylc.cc/docker/2017/01/08/Docker05registry-jenkins.html)  
[jenkins-registry持续集成-jenkins管理与registry集成打包上传镜像(二)](http://www.troylc.cc/docker/2017/02/01/Docker06registry-jenkins.html)

## 在swarm环境中部署高可用的eureka服务
把之前写的docker-compose.yml文件上传到拥有swarm环境下，执行以下操作

```
[root@docker-master01 docker-compose]# cat docker-compose.yml 
version: "3"
services:
  eurekaService1:      # 默认情况下，其他服务可以使用服务名称连接到该服务。因此，对于peer2的节点，它需要连接http://peer1:8761/eureka/，因此需要配置该服务的名称是peer1。
    image: tcr:5000/myhub/microservice-eureka-service:0.0.1
    networks:
      - eureka-net
    ports:
      - "9511:9511"
    environment:
      - spring.profiles.active=eurekaService1
  eurekaService2:
    image: tcr:5000/myhub/microservice-eureka-service:0.0.1
    networks:
      - eureka-net
    ports:
      - "9512:9512"
    environment:
      - spring.profiles.active=eurekaService2
  eurekaService3:
    image: tcr:5000/myhub/microservice-eureka-service:0.0.1
    networks:
      - eureka-net
    ports:
      - "9513:9513"
    environment:
      - spring.profiles.active=eurekaService3

networks:
  eureka-net:
    driver: overlay
[root@docker-master01 docker-compose]# docker stack deploy -c docker-compose.yml eureka
Creating network eureka_eureka-net
Creating service eureka_eurekaService2
Creating service eureka_eurekaService3
Creating service eureka_eurekaService1
[root@docker-master01 docker-compose]# docker stack ps eureka
ID            NAME                     IMAGE                                             NODE             DESIRED STATE  CURRENT STATE           ERROR  PORTS
nv8xh5b4tayn  eureka_eurekaService1.1  tcr:5000/myhub/microservice-eureka-service:0.0.1  docker-node02    Running        Running 10 seconds ago         
thuhip7pgoa2  eureka_eurekaService3.1  tcr:5000/myhub/microservice-eureka-service:0.0.1  docker-node01    Running        Running 12 seconds ago         
l8lfb9jtdd9m  eureka_eurekaService2.1  tcr:5000/myhub/microservice-eureka-service:0.0.1  docker-master01  Running        Running 12 seconds ago         
[root@docker-master01 docker-compose]# 
```
参考：  
[docker1.12.3 docker-swarm创建与管理集群（一）](http://www.troylc.cc/docker/2017/02/07/Docker07docker-swarm01.html)  
[docker1.12.3 docker-swarm集群服务部署与维护（二）](http://www.troylc.cc/docker/2017/02/19/Docker07docker-swarm02.html)  

## 验证eureka服务
在浏览器上输入部署服务1的地址,输入用户名/密码(eadmin/eadmin123)  
![登录eureka](/images/spring-cloud/eureka/5.png)  
![eureka服务界面](/images/spring-cloud/eureka/6.png)  
在浏览器上输入部署服务2的地址输入用户名/密码(eadmin/eadmin123)  
![eureka服务界面](/images/spring-cloud/eureka/7.png)  
在浏览器上输入部署服务3的地址,输入用户名/密码(eadmin/eadmin123)  
![eureka服务界面](/images/spring-cloud/eureka/8.png) 

至此eureka service注册中心基本上已经搭建起来了，后面会通过spring cloud 进行服务提供者的注册，消费者对服务进行消费,以下相关的一些组件，都在swarm集群中部署运行，敬请期待.......



# 参考：
[使用Spring Cloud和Docker构建微服务 （一）：序言](http://blog.itmuch.com/article/13)  
[Spring Cloud构建微服务架构（一）服务注册与发现](http://blog.didispace.com/springcloud1/)  
[Spring Cloud第一篇 Eureka简介及原理](http://www.itmuch.com/spring-cloud-1/)