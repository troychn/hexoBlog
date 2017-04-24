---
title: SpringCloud构微服务之-配置中心(六)  
toc: true  
date: 2017-04-16 19:18:23 Sunday  
updated: 2017-04-16 19:49:34 Sunday  
categories: [spring cloud]  
tags: [spring-boot,spring cloud,docker,swarm,docker-compose,java]  

---



通过前面的章节介绍，我们使用Eureka实现了服务中心；通过Feign实现服务间的消费，为了使服务更为健壮，使用Hystrix的熔断来避免在微服务架构中因个别服务出现异常而引起的故障蔓延，我们通过zuul实现了外部系统访问微服务的路由以及相关的权限控制，为了整合异构平台提供的微服务，我们通过sidecar组件把异构平台的微服务集成到了整个微服务的环境中。  
当我们的业务系统越来越庞大复杂的时候，各种配置就会层出不群。一旦配置修改了，那么我们就是必须修改后停服务，然后再上线，如果服务少，我们可以手动来操作，如果是成千上百的服务，如果是手动操作，肯定就不合适宜了，这个时候我们就需要考虑分布式配置管理，spring cloud config配置中心就是为了解决这个问题的组件， 

## 参考

疑问 | 参考
---|--- 
如果你对spring cloud的怎么把异构平台的服务纳入微服务 | [SpringCloud构建异构平台的微服务之-sidecar](http://www.troylc.cc/spring-cloud/2017/04/13/spirng-cloud-sidecar.html) 
如果你对spring cloud的服务网关不太了解？| [SpringCloud构建微服务之-apiGateway](http://www.troylc.cc/spring-cloud/2017/03/19/spirng-cloud-apigateway.html)
如果你对eureka注册中心不太了解？ | [SpringCloud和docker之微服务-eureka](http://www.troylc.cc/spring-cloud/2017/03/01/spirng-cloud-eureka.html)
如果你对服务注册不太了解 | [SpringCloud和docker之微服务-provider](http://note.youdao.com/)  
如果你对服务消费不太了解 | [SpringCloud和docker之微服务-consumer](http://www.troylc.cc/spring-cloud/2017/03/11/spirng-cloud-productservice.html)
如果你对docker安装不了解 | [docker、docker-compse最新版本安装](http://www.troylc.cc/docker/2017/01/05/docker04ininstall.html)
如果你对docker-swarm集群创建不太了解 | [docker-swarm创建与管理集群](http://www.troylc.cc/docker/2017/02/07/Docker07docker-swarm01.html)
如果你对swarm集群的服务部署不太了解 | [docker-swarm集群服务部署与维护](http://www.troylc.cc/docker/2017/02/19/Docker07docker-swarm02.html)
如果你不知道docker-compose怎么来部署swarm集群? | [docker-compose部署swarm服务(docker1.13.1)](http://www.troylc.cc/docker-compose/2017/02/25/Docker08docker-compose01.html)

## spring cloud 微服务配置中心
spring cloud config 由server端和client端组成，下面我就来结合git仓库来实现分布式配置中心搭建，在此章节我们搭建一个configServer,一个configClient端，并把服务端注册到eureka中心，以便于configClient通过eureka上注册的信息连接到configServer上,通过在configServer中集成spring cloud bus利用rabbitmq或kafka消息机制来实现配置更新后的自动刷新。

![image](/images/spring-cloud/config/1.png)

## 配置中心 服务端
### 创建configService
- 新建一个microservice-config-service的子工程，添加spring-cloud-config-server、spring-cloud-starter-eureka、spring-boot-starter-security、spring-cloud-starter-bus-kafka，如果消息总线是rabbitmq则替换kafka的依赖为spring-cloud-starter-bus-amqp  

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<!--配置需要认证所需要引用的包-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-bus-kafka</artifactId>
 </dependency>
```
- 在新建的工程中的spring boot启动类上加上@EnableConfigServer、@EnableDiscoveryClient注册，第一个注解是启动此项目为configService端，第二个注册是把这个configService注册到Eureka上。  

```
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer
public class ConfigServceApplication
{
    public static void main(String[] args) {
        SpringApplication.run(ConfigServceApplication.class, args);
    }
}

```
- 配置configservice的配置文件，在bootstrap.yml,添加安全认证的配置，git仓库的配置,eurekaservice的配置，zookeeper-kafka的配置。如：

```
server:
  port: 9518
eureka:
  instance:
    hostname: configService
    prefer-ip-address: true
    ip-address: ${eureka.instance.hostname} #只有当prefer-ip-address: true 时才生效
    instance-id: ${eureka.instance.hostname}:${server.port}  # 将Instance ID设置成IP:端口的形式
  client:
    serviceUrl:
     defaultZone: http://eadmin:eadmin123@eurekaService1:9511/eureka/,http://eadmin:eadmin123@eurekaService2:9512/eureka/,http://eadmin:eadmin123@eurekaService3:9513/eureka/   #把configservice注册到eureka上，以便于客户端通过eureka上注册的信息找到configservice
#实现的基本的 HttpBasic 的认证
security:
  basic:
    enabled: true     # 开启基于HTTP basic的认证
  user:
    name: cadmin      # 配置登录的账号是user
    password: cadmin123   # 配置登录的密码是eadmin123
#
spring:
  application:
    name: microservice-config-service
  cloud:
    config:
      server:
        git:
          uri: https://git.oschina.net/gittroylc/microservice-config-repo  #配置git仓库位置
          clone-on-start: true #在启动的时候克隆仓库
          search-paths: '{application}' #配置仓库路径下的相对搜索位置，可以配置多个
          username: username   #填写git仓库的用户名
          password: password   #填写git仓库的密码
    stream:   #配置通过spring cloud bus利用kafka消息机制实现自动刷新配置文件
      kafka:
        binder:
          zk-nodes: zookeeper:2181
          brokers: kafka:9092
```
### 创建git仓库
创建一个git仓库microservice-config-repo目录作为配置仓库，在仓库下创建一个microservice-provider-userservice对应的微服务文件夹，并根据不同环境新建了下面四个配置文件：   
![image](/images/spring-cloud/config/2.png)  
microservice-provider-userservice-dev.yml：

```
spring:
  jpa:
    generate-ddl: false
    show-sql: true
    hibernate:
      ddl-auto: none
  datasource:                           # 指定数据源
    platform: h2                        # 指定数据源类型
    schema: classpath:schema.sql        # 指定h2数据库的建表脚本
    data: classpath:data.sql            # 指定h2数据库的insert脚本
logging:
  level:
    root: INFO
    org.hibernate: INFO
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
    org.hibernate.type.descriptor.sql.BasicExtractor: TRACE
    com.troylc.cloud: debug

management:
  security:
    enabled: false
# 测试配置属性的自动刷新功能，增加一个自定义的属性文件
from: git-dev-15.0
```
在这四个文件中都有一个from的属性，其属性值分别为：  
from: git-default-1.0  
from: git-dev-15.0  
from: git-prod-1.0  
from: git-test-1.0  

### 服务端验证
启动eurekaservice和configservice两个应用，并把相关的zookeeper,kafka等服务启动，我这里是开发阶段，zookeeper,kafka是用docker镜像启动的服务，两个服务是直接通过idea启动来验证，后面全搭建完成了，直接通过一个docker-compose.yml来统一部署所有服务  

在浏览器中输入：http://configservice:9518/microservice-provider-userservice/dev
![image](/images/spring-cloud/config/3.png)  
![image](/images/spring-cloud/config/4.png)  
从返回的json可以看出，propertySources读取microservice-provider-userservice-dev.yml，还读取了microservice-provider-userservice.yml。从读取的情况来看，如果没有microservice-provider-userservice-dev.yml文件，他会默认的读取microservice-provider-userservice.yml文件

对git仓库中的配置文件microservice-provider-userservice.yml的访问方式有：

```
curl -s http://localhost:端口/test-service/dev |jq .
curl -s http://localhost:端口/test-service-dev.properties
curl -s http://localhost:端口/test-service-dev.json | jq .
curl -s http://localhost:端口/test-service-dev.yml
```
HTTP服务的git资源构成:

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```
- {application}:对应客户端的spring.application.name属性;
- {profile}:对应客户端的 spring.profiles.active属性(逗号分隔的列表);
- {label}:对应服务端属性配置文件的版本。对应git是:提交id,分支名称或tag。  

优先级  
- profiles的优先级高于defaults,有多个profiles,最后一个起作用。
- /{application}/{profile}[/{label}]优先级高于application.properties

其它方式配置仓库位置：  
- Spring Cloud Config提供本地存储配置的方式。只需要设置属性spring.profiles.active=native，Config Server会从应用的src/main/resource目录下搜索配置文件。  
- spring.cloud.config.server.native.searchLocations=file:F:/properties/ 属性来指定配置文件的位置。  

虽然Spring Cloud Config提供了其它配置仓库的功能，但为了能更好的管理内容和版本控制，推荐使用git的方式。

## 微服务端(configclient)配置

在开发完成并测试验证了configservice之后，下面我们看看如何在微服务应用中获取相关的配置信息。
- 改造之前我们开发的用户微服务的项目，在pom文件中增加spring-cloud-starter-config、spring-cloud-starter-bus-kafka两个依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>

 <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```
- 新建一个配置文件bootstrap.yml,指定configservice相关的配置。

```
spring:
  application:
    name: microservice-provider-userservice
  cloud:
    config:
      username: cadmin  #configservice认证的用户名
      password: cadmin123   #认证密码
      label: master   # 仓库的分支节点
      discovery:  
        enabled: true  #开启通过eureka上configservice找到相应的配置
        service-id: microservice-config-service #configservice注册在Eureka上的service-id
      profile: dev   #仓库中对应文件的环境，如dev、prod、test等
      fail-fast: true
    stream:   #配置通过spring cloud bus利用kafka消息机制实现自动刷新配置文件
      kafka:
        binder:
          zk-nodes: zookeeper:2181  
          brokers: kafka:9092
    bus:
      trace:
        enabled: true   #Spring Cloud Bus事件传播的细节
eureka:
  client:
    serviceUrl:
      defaultZone: http://eadmin:eadmin123@eurekaService1:9511/eureka/,http://eadmin:eadmin123@eurekaService2:9512/eureka/,http://eadmin:eadmin123@eurekaService3:9513/eureka/
management:
  security:
    enabled: false
#服务状态UNKNOWN
#如果把微服务的 eureka.client.healthcheck.enabled 属性配置在 bootstrap.yml 里面，可能会引起一些不良反应
#比如，实际测试发现，Eureka 首页显示的服务状态，本应是 UP(1)，却变成大红色的粗体 UNKNOWN(1)
#    healthcheck:
#      enabled: true

```
指定相关的configservice的eureka上的注册ID，指定zookeeper、kafka服务配置信息。

属性名 | 说明 | 默认值
---|---|---
spring.cloud.stream.kafka.binder.brokers | Kafka的服务端列表 | localhost
spring.cloud.stream.kafka.binder.defaultBrokerPort | Kafka服务端的默认端口，当brokers属性中没有配置端口时，就会个默认这端口 | 9092
spring.cloud.stream.kafka.binder.zk-nodes | Kafka服务端连接的ZooKeeper节点列表 | localhost
spring.cloud.stream.kafka.binder.defaultZkPort |  ZooKeeper节点的默认端口，当zk-nodes属性中没有配置端口时，就会默认这个端口 | 2181

通过以上配置说明，可以看出，如果我们在配置文件中不配置相关的zookeeper和kafka信息时，就会使用以上说明的默认值

## 注意：  
上面这些属性必须配置在bootstrap.yml，configservice的内容才能正确加载。因为通过bootstrap.yml的加载优先级比configService的高，configservice加载优先于application.yml，所以如果你把上面的配置写在application.yml中，相当于默认不是从configService中读取的配置信息，而是spring boot的默认加载。启动的时候就会看到加载的配置，不是你指定的configservice的服务器,而是默认的http://localhost:8888服务中加载


- 在userController中添加一个restAPI的方法，用来从配置仓库中读取一个from属性,如：

```
@RefreshScope
@RestController
public class UserController {
    private static Logger log = LoggerFactory.getLogger(UserController.class);

    ......

    @Value("${from}")
    private String fromInfostr;
    /**
     * 获取所有用户
     * @return
     */
    @ApiOperation(value = "获取自动刷新后的配置文件中from中的值", notes = "获取自动刷新后的配置文件")
    @GetMapping("/from")
    public ResultInfo geFromInfo() {
        String fromInfostr = null;
        try {
            fromInfostr = this.fromInfostr;
        } catch (Exception e) {
            log.error(e.getMessage());
            return new ResultInfo<String>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        return new ResultInfo<String>(ReturnInfoEnum.SUCCESS.getState(),ReturnInfoEnum.SUCCESS.getStateInfo(), fromInfostr);
    }
    
    .......
    
```
## 微服务端测试
启动configservice和configclient，为了便于观察消息总线刷新配置的效果，可以启动多个不同端口的configclient。可以看到configservice以及多个configclient都连接上由Kafka实现的消息总线。直接访问每个configclient上的/from请求，查看获取到的from配置的内容，可以看到一开始，都是之前写的默认值。之后，修改Git中对应配置文件中的参数内容，向configservice发送POST请求：/bus/refresh，再去访问各个configclient上的/from请求，可以看到各客户端上的配置都刷新为最新配置内容。  
eureka上看启动的服务：  
![image](/images/spring-cloud/config/5.png)  
请求configclient上的查看from属性值：  
![image](/images/spring-cloud/config/6.png)  
修改git仓库对应的microservice-provider-userservice-dev.yml文件的from的值，并提交，push到git仓库

```
from: git-dev-16.0
```
再从本地ssh中用curl向configServic发送一个/bus/refresh刷新请求，会看到configservice和configclient端应用程序会打出刷新后重新加载配置文件的日志


```
curl -X POST cadmin:cadmin123@configService:9518/bus/refresh

```

```

2017-04-14 17:04:19.078  INFO 4600 --- [afka-listener-1] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_MICROSERVICE-PROVIDER-USERSERVICE/userService:9514 - deregister  status: 200
2017-04-14 17:04:19.092  INFO 4600 --- [afka-listener-1] com.netflix.discovery.DiscoveryClient    : Completed shut down of DiscoveryClient
2017-04-14 17:04:19.093  INFO 4600 --- [afka-listener-1] c.n.e.EurekaDiscoveryClientConfiguration : Unregistering application microservice-provider-userservice with eureka with status DOWN
.......
2017-04-14 17:04:19.224  INFO 4600 --- [afka-listener-1] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1492160659224 with initial instances count: 2
2017-04-14 17:04:19.230  INFO 4600 --- [afka-listener-1] c.n.e.EurekaDiscoveryClientConfiguration : Registering application microservice-provider-userservice with eureka with status UP
2017-04-14 17:04:19.230  WARN 4600 --- [afka-listener-1] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1492160659230, current=UP, previous=DOWN]
2017-04-14 17:04:19.230  INFO 4600 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_MICROSERVICE-PROVIDER-USERSERVICE/userService:9514: registering service...
2017-04-14 17:04:19.234  INFO 4600 --- [afka-listener-1] c.n.e.EurekaDiscoveryClientConfiguration : Unregistering application microservice-provider-userservice with eureka with status DOWN
2017-04-14 17:04:19.234  INFO 4600 --- [afka-listener-1] c.n.e.EurekaDiscoveryClientConfiguration : Registering application microservice-provider-userservice with eureka with status UP
2017-04-14 17:04:19.234  INFO 4600 --- [afka-listener-1] o.s.cloud.bus.event.RefreshListener      : Received remote refresh request. Keys refreshed [config.client.version, from]
2017-04-14 17:04:19.237  INFO 4600 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_MICROSERVICE-PROVIDER-USERSERVICE/userService:9514 - registration status: 204
2017-04-14 17:04:19.248  INFO 4600 --- [afka-listener-1] o.a.k.clients.producer.ProducerConfig    : ProducerConfig values: 

```
再次请求configclient的/from  
![image](/images/spring-cloud/config/7.png)

至此spring cloud 的基本使用总结到这里，后续会出一个在docker swarm中把这些微服务，通过docker-compose进行统一编排部署。

本系列的完整示例：  
码云：  
[https://git.oschina.net/gittroylc/spring-cloud-docker-microservice](https://git.oschina.net/gittroylc/spring-cloud-docker-microservice)    
github:  
[https://github.com/troychn/spring-cloud-docker-microservice](https://github.com/troychn/spring-cloud-docker-microservice)






