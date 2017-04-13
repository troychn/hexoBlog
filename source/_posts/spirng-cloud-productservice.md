---
title: SpringCloud和docker之微服务-consumer(三)
toc: true
date: 2017-03-11 19:18:23 Sunday
updated: 2017-03-11 19:49:34 Sunday
categories: [spring cloud]
tags: [spring-boot,spring cloud,docker,swarm,docker-compose,java]

---
本节主要说明一下通过springcloud的一种声明式、模板化的http客户端feign来实现获取注册到eureka注册中心的用户服务的信息。并集成了hystrix(熔断器)，控制服务和服务之间的节点调用，提高服务的延迟和故障的容错能力，通过hystrix dashboard(熔断器 仪表盘)来直观的展示。

## 参考

疑问 | 参考
---|---
如果你对eureka注册中心不太了解？ | [SpringCloud和docker之微服务-eureka](http://www.troylc.cc/spring-cloud/2017/03/01/spirng-cloud-eureka.html)
如果你对服务注册不太了解 | [SpringCloud和docker之微服务-provider](http://www.troylc.cc/spring-cloud/2017/03/09/spirng-cloud-userservice.html)  
如果你对docker安装不了解 | [docker、docker-compse最新版本安装](http://www.troylc.cc/docker/2017/01/05/docker04ininstall.html)
如果你对docker-swarm服务部署不太了解 | [docker-swarm创建与管理集群](http://www.troylc.cc/docker/2017/02/07/Docker07docker-swarm01.html)、[docker-swarm集群服务部署与维护](http://www.troylc.cc/docker/2017/02/19/Docker07docker-swarm02.html)



## 效果访问说明：  
访问URL | 访问方式 | 说明  
---|---|---
http://productservice:9515/getUsers/1 | FeginClient方式调用接口 | 根据用户ID获取用户信息
http://productservice:9515/users-rest/1 | RestTemplate方式调用接口 | 根据ID获取用户信息  
## Hystrix Dashboard监控说明

访问URL | 说明
---|---
http://productservice:9515/hystrix | hystrix仪表盘访问地址
http://productservice:9515/hystrix.stream | 实时监控接口调用的访问地址，配合仪表盘使用，更能直观展示服务的访问情况。  

## FeignClient
- 首先创建子工程、引用依赖包
创建一个consumer maven子工程microservice-consumer-productservice,并在pom.xml中引用所需依赖，如：

```
       <!--添加spring cloud服务注册的依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
        <!--添加spring cloud的feign依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-feign</artifactId>
        </dependency>
        <!--添加spring cloud的hystrix断路器：主要是服务间调用提供更加强大的容错能力 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix</artifactId>
        </dependency>
        <!-- 用于暴露自身信息的模块，它的主要作用是用于监控与管理 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!-- Spring Boot中使用Swagger2构建RESTful APIs  -->
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
        </dependency>
        <!-- hystrix-dashboard监控 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
        </dependency>
```
- 定义FeignClient、Hystrix
在spring boot启动类上增加@EnableFeignClients、@EnableCircuitBreaker、@EnableHystrixDashboard

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
@EnableCircuitBreaker //使用@EnableCircuitBreaker注解开启断路器功能
@EnableHystrixDashboard //启用HystrixDashboard功能
public class ProductServiceApplication {

    /**
     * 实例化RestTemplate，通过@LoadBalanced注解开启均衡负载能力.
     *
     * @return restTemplate
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

	public static void main(String[] args) {
		SpringApplication.run(ProductServiceApplication.class, args);
	}
}
```
定义需要通过FeignClient访问的接口列表，如下

UserServiceFeign
```
// name为服务名，对应spring.application.name。注意：此服务名必须已注册进Eureka服务中心
@FeignClient(name = "microservice-provider-userservice",fallback = UserServiceFeignFallback.class, configuration = FeignConfig.class)
public interface UserServiceFeign extends IUserService{
    
}
```
IUserService

```
/**
 * 用户业务接口
 * Created by troylc on 2017/3/6.
 */
public interface IUserService {

    /**
     * 根据ID查的对应的用户
     *
     * @param id
     * @return
     */
    @RequestMapping(value = "/users/{id}", method = RequestMethod.GET)
    ResultInfo<UserBean> getUserById(@PathVariable(value = "id") Long id) throws Exception;

}
```
定义各接口对应的fallback方式

```java
/**
 * 定义各接口对应的fallback方法
 * Created by troylc on 2017/3/6.
 */
@Component
public class UserServiceFeignFallback implements UserServiceFeign {

    @Override
    public ResultInfo getUserById(Long id) throws Exception {
        return new ResultInfo<>(ReturnInfoEnum.NULL.getState(), ReturnInfoEnum.NULL.getStateInfo());
    }

}
```
- 定义以RestTemplae方式调用接口
UserServiceFeignRest

```java
/**
 * 定义通过rest方式访问接口
 * Created by troylc on 2017/3/6.
 */
@Component
public class UserServiceFeignRest {

    @Autowired
    private RestTemplate restTemplate;

    public ResultInfo<UserBean> getUser(Long id) {
        ResultInfo<UserBean> resultInfo  = restTemplate.exchange("http://microservice-provider-userservice/users/{id}", HttpMethod.GET, null, new ParameterizedTypeReference<ResultInfo<UserBean>>() {
        }, id).getBody();
        return resultInfo;
    }
}
```

- 定义controller
controoler中增加API文档注解、设置启用Hystrix超时及时间

ProductControoler
```java
/**
 * 测试商品controol
 * Created by troylc on 2017/3/5.
 */
@RestController
public class ProductControoler {

    private static Logger log = LoggerFactory.getLogger(ProductControoler.class);

    @Autowired
    private IProductService productService;

    @Autowired
    private UserServiceFeign userServiceFeign;

    @Autowired
    private UserServiceFeignRest userServiceFeignRest;

    /**
     * 注：@GetMapping("/{id}")是spring 4.3的新注解等价于：
     *
     * @param id
     * @return user信息
     * @RequestMapping(value = "/id", method = RequestMethod.GET)
     * 类似的注解还有@PostMapping等等
     */
    @ApiOperation(value = "查找用户,通过spring cloud feign方式", notes = "根据用户的ID，查找对应的用户")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true, paramType = "path", dataType = "Long")
    @HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1000"),
            @HystrixProperty(name = "execution.timeout.enabled", value = "false")})
    @GetMapping("/getUsers/{id}")
    public ResultInfo getUserById(@PathVariable Long id) {
        UserBean userBean = null;
        try {
            ResultInfo<UserBean> resultInfo = userServiceFeign.getUserById(id);
            userBean = resultInfo.getData();
        } catch (Exception e) {
            e.printStackTrace();
            log.error(e.getMessage());
            return new ResultInfo<UserBean>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        return new ResultInfo<>(ReturnInfoEnum.SUCCESS.getState(), ReturnInfoEnum.SUCCESS.getStateInfo(), userBean);
    }

    /**
     * 注：@GetMapping("/{id}")是spring 4.3的新注解等价于：
     *
     * @param id
     * @return user信息
     * @RequestMapping(value = "/id", method = RequestMethod.GET)
     * 类似的注解还有@PostMapping等等
     */
    @ApiOperation(value = "查找库存端口", notes = "根据端口的ID，查找对应的商品信息")
    @ApiImplicitParam(name = "id", value = "商品ID", required = true, paramType = "path", dataType = "Long")
    @GetMapping("/getCommodityBean/{id}")
    public ResultInfo getCommodityBeanById(@PathVariable Long id) {
        ProductBean productBean = null;
        try {
            productBean = productService.getCommodityById(id);
        } catch (Exception e) {
            log.error(e.getMessage());
            return new ResultInfo<ProductBean>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        return new ResultInfo<>(ReturnInfoEnum.SUCCESS.getState(), ReturnInfoEnum.SUCCESS.getStateInfo(), productBean);
    }


    @ApiOperation(value = "查找用户,通过spring RestTemplate方式", notes = "根据用户的ID，查找对应的用户")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true, paramType = "path", dataType = "Long")
    @HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1000"),
            @HystrixProperty(name = "execution.timeout.enabled", value = "false")})
    @RequestMapping(value = "/users-rest/{id}", method = RequestMethod.GET)
    public ResultInfo getUserByRest(@PathVariable Long id) {
        UserBean userBean = null;
        try {
            ResultInfo<UserBean> resultInfo = userServiceFeignRest.getUser(id);
            userBean =   resultInfo.getData();
        } catch (Exception e) {
            e.printStackTrace();
            log.error(e.getMessage());
            return new ResultInfo<UserBean>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        return new ResultInfo<>(ReturnInfoEnum.SUCCESS.getState(), ReturnInfoEnum.SUCCESS.getStateInfo(), userBean);
    }

}
```
## 部署至swarm集群中-运行测试
docker-compose文件内容：

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
  userService:
        image: tcr:5000/myhub/microservice-provider-userservice:0.0.2
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
  productService:
          image: tcr:5000/myhub/microservice-consumer-productservice:0.0.2
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
networks:
  eureka-net:
    driver: overlay

```
执行：  
![image](/images/spring-cloud/productservice/1.png)


在AIP接口文档界面点击获取用户接口以feign的方式，输入用户ID，
在AIP接口文档界面点击获取用户接口以rest的方式，输入用户ID  
![image](/images/spring-cloud/productservice/2.png)  
![image](/images/spring-cloud/productservice/3.png)  
![image](/images/spring-cloud/productservice/4.png)

打开仪表盘(http://productservice:9515/hystrix)，输入http://productservice:9515/hystrix.stream,再多点击几次就出现了监控的数据。  
![image](/images/spring-cloud/productservice/5.png)  
![image](/images/spring-cloud/productservice/6.png)


**附代码仓库地址：**  
码云：  
[https://git.oschina.net/gittroylc/spring-cloud-docker-microservice](https://git.oschina.net/gittroylc/spring-cloud-docker-microservice)  
github:  
[https://github.com/troychn/spring-cloud-docker-microservice](https://github.com/troychn/spring-cloud-docker-microservice)











