---
title: SpringCloud构建异构平台的微服务之-sidecar(五)
toc: true
date: 2017-04-13 19:18:23 Sunday
updated: 2017-04-13 19:49:34 Sunday
categories: [spring cloud]
tags: [spring-boot,spring cloud,docker,swarm,docker-compose,java]

---

通过前面的几个章节介绍，我们使用Spring Cloud的Eureka实现了服务注册中心；而服务间通过Feign实现服务的消费，为了使得服务更为健壮，使用Hystrix的熔断机制来避免在微服务架构中因个别服务出现异常而引起的故障蔓延，为了使外部系统能够调用微服务注册中心注册的各种微服务，我们通过zuul实现了外部系统访问微服务的路由以及相关的权限控制。  
本节我们主要讨论一下异构平台（比如，nodejs、python、php等提供的Rest接口服务）的服务，怎么通过spring cloud组件对这些服务注册到eureka中心以及与在微服务中怎么和异构平台的服务进行通信。这里主要是通过spring cloud的sidecar来构建异构平台的服务注册与通信。  
sidecar灵感来自Netflix Prana。它可以获取注册中心的所有微服务实例的信息(例如host，端口等)的http api。也可以通过嵌入的Zuul代理来代理服务调用，该代理从Eureka获取其路由条目。 Spring Cloud配置服务器可以通过主机查找或通过Zuul代理直接访问。

## 参考

疑问 | 参考
---|--- 
如果你对spring cloud的服务网关不太了解？| [SpringCloud构建微服务之-apiGateway](http://www.troylc.cc/spring-cloud/2017/03/19/spirng-cloud-apigateway.html)
如果你对eureka注册中心不太了解？ | [SpringCloud和docker之微服务-eureka](http://www.troylc.cc/spring-cloud/2017/03/01/spirng-cloud-eureka.html)
如果你对服务注册不太了解 | [SpringCloud和docker之微服务-provider](http://note.youdao.com/)  
如果你对服务消费不太了解 | [SpringCloud和docker之微服务-consumer](http://www.troylc.cc/spring-cloud/2017/03/11/spirng-cloud-productservice.html)
如果你对docker安装不了解 | [docker、docker-compse最新版本安装](http://www.troylc.cc/docker/2017/01/05/docker04ininstall.html)
如果你对docker-swarm集群创建不太了解 | [docker-swarm创建与管理集群](http://www.troylc.cc/docker/2017/02/07/Docker07docker-swarm01.html)
如果你对swarm集群的服务部署不太了解 | [docker-swarm集群服务部署与维护](http://www.troylc.cc/docker/2017/02/19/Docker07docker-swarm02.html)
如果你不知道docker-compose怎么来部署swarm集群? | [docker-compose部署swarm服务(docker1.13.1)](http://www.troylc.cc/docker-compose/2017/02/25/Docker08docker-compose01.html)


## 开启sidecar之旅
通过Node.js构建的评论服务通过Sidecar接入Spring Cloud微服务集群的整体架构，如下图：  
![整体架构](/images/spring-cloud/sidecar/11.png) 
  
### nodejs应用-简单的评论服务
- 首页需要构建一个异构平台的rest服务，我们这里采用nodejs创建，为了能够让微服务的注册中心知道这个异构平台的服务，需要在异构平台应用中实现一个健康检查接口，让Sidecar可以把这个服务实例的健康情况告诉Eureka注册中心。如：  
![image](/images/spring-cloud/sidecar/1.png)  
创建一个nodejs的工程，在工程中实现一个健康接口并且返回如下形式的json文档： 

```
{
  "status":"UP"
}
```
- 编写一个首页和获取评论接口，该评论接口主要是供其它在微服务来调用，如：   
![image](/images/spring-cloud/sidecar/2.png)  
评价返加接口的json数据如：  

```
{
    status: '100',
    message: '操作成功',
    data: {
        commentId: '123456',
        userId: '2',
        productId: '1',
        commentContext: '这个品质不错，快递速度很快！'
    }
}
```

### 把nodejs的评论服务接入微服务的sidecar应用  
- 创建一个sidecar的子项目，在pom文件中添加sidecar的依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-netflix-sidecar</artifactId>
</dependency>
```
- 在spring boot的启动类上，加上启动sidecar的注解@EnableSidecar

```java
@SpringBootApplication
@EnableSidecar
public class SidecarApplication {

	public static void main(String[] args) {
		SpringApplication.run(SidecarApplication.class, args);
	}
}

```
我们可以看看@EnableSidecar都做了些什么事，点击这个注解查看源码，我们发现hystrix熔断器、Eureka服务发现、zuul代理，这些组件都启动了，可以看到这个注解是一个组合注解。

```
@EnableCircuitBreaker
@EnableDiscoveryClient
@EnableZuulProxy
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({SidecarConfiguration.class})
public @interface EnableSidecar {
}
```
- 配置sidecar的application.yml属性文件

```
spring:
  application:
    name: microservice-sidecar-comment
server:
  port: 9517
eureka:
  instance:
     hostname: sidecar
     prefer-ip-address: true
     ip-address: ${eureka.instance.hostname} #只有当prefer-ip-address: true 时才生效
     instance-id: ${eureka.instance.hostname}:${server.port}  # 将Instance ID设置成IP:端口的形式
  client:
      serviceUrl:
        defaultZone: http://eadmin:eadmin123@eurekaService1:9511/eureka/,http://eadmin:eadmin123@eurekaService2:9512/eureka/,http://eadmin:eadmin123@eurekaService3:9513/eureka/
      healthcheck:
        enabled: true
        
sidecar:
  port: 3000    # Node.js微服务的端口
  health-uri: http://sidecar:3000/health   # Node.js微服务的健康检查URL

hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: false
```
这里主要说明以下两个属性：  
sidecar.port属性代表Node.js应用监听的端口。  
sidecar.health-uri是一个用来模拟Spring Boot应用健康检查的接口的，接口返加的json必须是"status":"UP"。

### 消费端微服务定义
在之前的商品微服务中，定义一个feign接口，写上对应的sidecar 的服务ID，如：

![image](/images/spring-cloud/sidecar/3.png)

FeignClient可以根据serviceId去Eureka注册中心上找对应的服务信息，如果服务的实例不止一个，就会使用Ribbon进行客户端负载均衡
```
@FeignClient(name = "microservice-sidecar-comment")
public interface CommentServiceFeign {
    @RequestMapping(value = "/comment", method = RequestMethod.GET)
    ResultInfo getComment() throws Exception;
}
```

## compose部署并测试
通过docker-compose在swarm部署并启动所有服务进行测试，docker-compose文件如下

```
version: "3"
services:
  eurekaService1:      # 默认情况下，其他服务可以使用服务名称连接到该服务。因此，对于peer2的节点，它需要连接http://peer1:8761/eureka/，因此需要配置该服务的名称是peer1。
    image: tcr:5000/myhub/microservice-eureka-service:0.0.1
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
  eurekaService2:
    image: tcr:5000/myhub/microservice-eureka-service:0.0.1
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
  eurekaService3:
    image: tcr:5000/myhub/microservice-eureka-service:0.0.1
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
  productService:
    image: tcr:5000/myhub/microservice-consumer-productservice:0.0.1
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
  apiGateway:
    image: tcr:5000/myhub/microservice-api-gateway:0.0.1
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
  nodeComment:
    image: tcr:5000/myhub/microservice-nodejs-comment:0.0.1
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
  sidecarComment:
    image: tcr:5000/myhub/microservice-sidecar-comment:0.0.1
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
  userService:
    image: tcr:5000/myhub/microservice-provider-userservice:0.0.3
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
      - eurekaService1
      - eurekaService2
      - eurekaService3
networks:
  eureka-net:
    driver: overlay

```
登录eureka，可看所有服务  
![image](/images/spring-cloud/sidecar/4.png)

通过sidecar访问一下nodejs的提供的健康检查接口：  
![image](/images/spring-cloud/sidecar/5.png)  
这说明zuul功能已经开启了  
通过http://sidecar:9517/hosts/microservice-sidecar-comment访问sidecar的路由地址：  
![image](/images/spring-cloud/sidecar/51.png)  
通过商品微服务访问nodejs异构平台构建的评论微服务信息，  
![image](/images/spring-cloud/sidecar/6.png)

**附代码仓库地址：**  
码云：  
[https://git.oschina.net/gittroylc/spring-cloud-docker-microservice](https://git.oschina.net/gittroylc/spring-cloud-docker-microservice)  
github:  
[https://github.com/troychn/spring-cloud-docker-microservice](https://github.com/troychn/spring-cloud-docker-microservice)