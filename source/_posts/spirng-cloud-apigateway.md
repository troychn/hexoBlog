---
title: SpringCloud和docker之微服务-apigateway(四)
toc: true
date: 2017-03-19 19:18:23 Sunday
updated: 2017-03-19 19:49:34 Sunday
categories: [spring cloud]
tags: [spring-boot,spring cloud,docker,swarm,docker-compose,java]

---

通过前面几节，我们已经通过spring cloud的组件构建了一个简单的微服务架构。  
我们使用Spring Cloud中的Eureka实现了服务注册中心以及服务注册与发现；而服务间通过Feign实现服务的消费，为了使得服务集群更为健壮，使用Hystrix的融断机制来避免在微服务架构中个别服务出现异常时引起的故障蔓延。

本文继续spring cloud和docker之微服务的api网关的相关介绍与案例，先来看看如下图：  
![服务网关](/images/spring-cloud/apigateway/1-1.png)

本图说明：内部服务Service A和Service B，他们都会注册与订阅服务至Eureka Server，而Open Service是一个对外的服务，通过负载均衡公开至服务调用方，在open Service中我们需要将权限控制从我们的服务单元中抽离出去，而最适合这些逻辑的地方就是处于对外访问最前端的地方，我们需要一个更强大一些的均衡负载器，它就是本文将要介绍的：服务网关。  
服务网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供REST API的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制等功能。Spring Cloud中的Zuul就担任了这样的一个角色，为微服务架构提供了前门保护的作用，同时将权限控制这些较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。 

为什么微服务中实现服务网关很重要：  
- 服务网关实现了路由功能来屏蔽诸多服务细节，更实现了服务级别、均衡负载的路由。
- 实现了接口权限校验与微服务业务逻辑的解耦。通过服务网关中的过滤器，在各生命周期中去校验请求的内容，将原本在对外服务层做的校验前移，保证了微服务的无状态性，同时降低了微服务的测试难度，让服务本身更集中关注业务逻辑的处理。
- 实现了断路器，不会因为具体微服务的故障而导致服务网关的阻塞，依然可以对外服务。

## 参考

疑问 | 参考
---|---
如果你对eureka注册中心不太了解？ | [SpringCloud和docker之微服务-eureka](http://www.troylc.cc/spring-cloud/2017/03/01/spirng-cloud-eureka.html)
如果你对服务注册不太了解 | [SpringCloud和docker之微服务-provider](http://note.youdao.com/)  
如果你对服务消费不太了解 | [SpringCloud和docker之微服务-consumer](http://www.troylc.cc/spring-cloud/2017/03/11/spirng-cloud-productservice.html)
如果你对docker安装不了解 | [docker、docker-compse最新版本安装](http://www.troylc.cc/docker/2017/01/05/docker04ininstall.html)
如果你对docker-swarm集群创建不太了解 | [docker-swarm创建与管理集群](http://www.troylc.cc/docker/2017/02/07/Docker07docker-swarm01.html)
如果你对swarm集群的服务部署不太了解 | [docker-swarm集群服务部署与维护](http://www.troylc.cc/docker/2017/02/19/Docker07docker-swarm02.html)
如果你不知道docker-compose怎么来部署swarm集群? | [docker-compose部署swarm服务(docker1.13.1)](http://www.troylc.cc/docker-compose/2017/02/25/Docker08docker-compose01.html)

## 准备工作
在使用Zuul之前，我们先构建一个服务注册中心、以及两个简单的服务，比如：我构建了一个microservice-provider-userservice，一个microservice-consumer-productservice。然后启动eureka-server和这两个服务。通过访问eureka-server，我们可以看到microservice-provider-userservice和microservice-consumer-productservice已经注册到了服务中心。


如果不熟悉请在参考中找到对应的文章进行操作。也可以通过文章最后附的源码，自己构建。

## 开始使用Zuul
- 引入依赖spring-cloud-starter-zuul、spring-cloud-starter-eureka，如果不是通过指定serviceId的方式，eureka依赖不需要，但是为了对服务集群细节的透明性，还是用serviceId来避免直接引用url的方式吧

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
- 应用主类使用@EnableZuulProxy注解开启Zuul

```java
package com.troylc.cloud;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@EnableZuulProxy
@SpringBootApplication
public class ApiGatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(ApiGatewayApplication.class, args);
	}
}
```
- application.yml中配置Zuul应用的基础信息，如：应用名、服务端口等。

```
spring:
  application:
    name: microservice-api-gateway
server:
  port: 9516
eureka:
  instance:
     hostname: apiGateway
     prefer-ip-address: true
     ip-address: ${eureka.instance.hostname} #只有当prefer-ip-address: true 时才生效
     instance-id: ${eureka.instance.hostname}:${server.port}  # 将Instance ID设置成IP:端口的形式
  client:
      serviceUrl:
        defaultZone: http://eadmin:eadmin123@eurekaService1:9511/eureka/,http://eadmin:eadmin123@eurekaService2:9512/eureka/,http://eadmin:eadmin123@eurekaService3:9513/eureka/
      healthcheck:
        enabled: true
# 设置默认超时时间60s（default为全局；若想设置某项服务的超时时间，只需要将default替换为对应的服务名）
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 60000
zuul:
   routes:
     api-productservice:
       path: /api/product/**
       serviceId: microservice-consumer-productservice
       stripPrefix: true
# stripPrefix：是否去除前缀，默认为true
# stripPrefix=true, http://apiGateway:9516/api/swagger/api/hello ==> http://apiGateway:9516/api/hello
# stripPrefix=false, http://apiGateway:9516/api/swagger/api/hello ==> http://apiGateway:9516/api/swagger/api/hello
     api-usertservice:
       path: /api/users/**
       serviceId: microservice-provider-userservice
```
zuul配置：  
我们在实现微服务架构时，服务名与服务实例地址的关系在eureka server中已经存在了，所以只需要将Zuul注册到eureka server上去发现其他服务，我们就可以实现对serviceId的映射。

```
zuul:
   routes:
     api-productservice:
       path: /api/product/**
       serviceId: microservice-consumer-productservice
     api-usertservice:
       path: /api/users/**
       serviceId: microservice-provider-userservice   
```
针对我们在准备工作中实现的两个微服务microservice-provider-userservice和microservice-consumer-productservice，定义了两个路由api-a和api-b来分别映射。另外为了让Zuul能发现microservice-provider-userservice和microservice-consumer-productservice，也加入了eureka的配置  


## 服务过滤
在完成了服务路由之后，我们对外开放服务还需要一些安全措施来保护客户端只能访问它应该访问到的资源。所以我们需要利用Zuul的过滤器来实现我们对外服务的安全控制。  
在服务网关中定义过滤器只需要继承ZuulFilter，如：

```java
package com.troylc.cloud.filter;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.troylc.cloud.utils.ReturnInfoEnum;
import com.troylc.cloud.vbean.ResultInfo;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;

/**
 * Created by troylc on 2017/3/14.
 * 自定义过滤器的实现，需要继承ZuulFilter，需要重写实现下面四个方法：
 * filterType：返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型，具体如下：
 * pre：可以在请求被路由之前调用
 * routing：在路由请求时候被调用
 * post：在routing和error过滤器之后被调用
 * error：处理请求时发生错误时被调用
 * filterOrder：通过int值来定义过滤器的执行顺序
 * shouldFilter：返回一个boolean类型来判断该过滤器是否要执行，所以通过此函数可实现过滤器的开关。
 * 在上例中，我们直接返回true，所以该过滤器总是生效。
 * run：过滤器的具体逻辑。需要注意，这里我们通过ctx.setSendZuulResponse(false)令zuul过滤该请求，
 * 不对其进行路由，然后通过ctx.setResponseStatusCode(401)设置了其返回的错误码，
 * 当然我们也可以进一步优化我们的返回，比如，通过ctx.setResponseBody(body)对返回body内容进行编辑，
 * 如果有中文乱码。则可以：ctx.getResponse().setContentType("text/html;charset=UTF-8")
 */
@Component
public class AccessControlFilter extends ZuulFilter {

    private static Logger log = LoggerFactory.getLogger(AccessControlFilter.class);

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        log.info(String.format("%s request to %s", request.getMethod(), request.getRequestURL().toString()));
        Object accessToken = request.getParameter("accessToken");
        if (accessToken == null) {
            log.warn("access token is empty,please enter accessToken!");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            //未认证
            ResultInfo resultInfo = new ResultInfo<>(ReturnInfoEnum.NOT_AUTHENTICATE.getState(),ReturnInfoEnum.NOT_AUTHENTICATE.getStateInfo());
            ctx.getResponse().setContentType("text/html;charset=UTF-8");
            ctx.getResponse().setContentType(String.valueOf(MediaType.APPLICATION_JSON));
            ctx.setResponseBody(resultInfo.toString());
            return null;
        }
        log.info("access token ok");
        return null;
    }

}

```
根据对filterType生命周期介绍，可以参考下图去理解，并根据自己的需要在不同的生命周期中去实现不同类型的过滤器。    
![filterType生命周期](/images/spring-cloud/apigateway/1-2.png)

## 定义服务fallback
完成了服务网站的filter,我们可以针对具体的内部服务，在zuul中定义服务的回退方法
如：
UserFallbackProvider：

```java
package com.troylc.cloud.fallback;

import com.troylc.cloud.utils.ReturnInfoEnum;
import com.troylc.cloud.vbean.ResultInfo;
import org.springframework.cloud.netflix.zuul.filters.route.ZuulFallbackProvider;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.stereotype.Component;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;

/**
 * 路由网关的用户短路器返回调用
 * Created by troylc on 2017/3/14.
 */
@Component
public class UserFallbackProvider implements ZuulFallbackProvider{
    @Override
    public String getRoute() {
        return "microservice-provider-userservice";
    }
    @Override
    public ClientHttpResponse fallbackResponse() {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.OK;
            }
            @Override
            public int getRawStatusCode() throws IOException {
                return 200;
            }
            @Override
            public String getStatusText() throws IOException {
                return "OK";
            }
            @Override
            public void close() {

            }
            @Override
            public InputStream getBody() throws IOException {
                ResultInfo resultInfo = new ResultInfo<>(ReturnInfoEnum.ERROR_SERVICE.getState(),
                        ReturnInfoEnum.ERROR_SERVICE.getStateInfo() + ";服务名为：" + UserFallbackProvider.this.getRoute());
                return new ByteArrayInputStream(resultInfo.toString().getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}

```
ProductFallbackProvider：

```
package com.troylc.cloud.fallback;

import com.troylc.cloud.utils.ReturnInfoEnum;
import com.troylc.cloud.vbean.ResultInfo;
import org.springframework.cloud.netflix.zuul.filters.route.ZuulFallbackProvider;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.stereotype.Component;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;

/**
 * 路由网关的商品短路器返回调用
 * Created by troylc on 2017/3/14.
 */
@Component
public class ProductFallbackProvider implements ZuulFallbackProvider {
    @Override
    public String getRoute() {
        return "microservice-consumer-productservice";
    }

    @Override
    public ClientHttpResponse fallbackResponse() {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.OK;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return 200;
            }

            @Override
            public String getStatusText() throws IOException {
                return "OK";
            }

            @Override
            public void close() {

            }

            @Override
            public InputStream getBody() throws IOException {
                ResultInfo resultInfo = new ResultInfo<>(ReturnInfoEnum.ERROR_SERVICE.getState(),
                        ReturnInfoEnum.ERROR_SERVICE.getStateInfo()+";服务名为："+ProductFallbackProvider.this.getRoute());
                return new ByteArrayInputStream(resultInfo.toString().getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}

```

## docker-compose运行服务

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
    image: tcr:5000/myhub/microservice-provider-userservice:0.0.1
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
networks:
  eureka-net:
    driver: overlay

```

接下来，我们将microservice-eureka-services、microservice-provider-userservice、microservice-consumer-productservice以及这里用Zuul实现的服务网关启动起来，在eureka-server的控制页面中，我们可以看到分别注册了icroservice-provider-userservice、microservice-consumer-productservice以及microservice-api-gateway  
在swarm集群的manager节点中执行以下操作：    
![image](/images/spring-cloud/apigateway/1.png)  
查看eureka注册中心    
![image](/images/spring-cloud/apigateway/2.png)

通过zuul去访问商品服务中的获取用户节点，此时不带accessToken参数，测试访问接受需要授权  
![image](/images/spring-cloud/apigateway/3.png)  
加上accessToken参数访问：  
![image](/images/spring-cloud/apigateway/4.png)  
下面测试停止商品服务后，zuul的fallback方法回调，首先操作swarm集群中，把商品服务停止  
![image](/images/spring-cloud/apigateway/5.png)  
查看eureka注册中心，确认product服务是否已经停止  
![image](/images/spring-cloud/apigateway/6.png)  
再次访问商品服务，提示fallback方法返回的内容：  
![image](/images/spring-cloud/apigateway/7.png)