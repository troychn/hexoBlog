---
title: SpringCloud和docker之微服务-provider(二)
toc: true
date: 2017-03-08 19:18:23 Sunday
updated: 2017-03-08 19:49:34 Sunday
categories: [spring cloud]
tags: [spring-boot,spring cloud,docker,swarm,docker-compose,java]

---
通过上一节我们已经通过docker-compose在swarm中部署了有三个实例的高可用eureka服务注册中心,本节我们讨论一下怎么把一个已知的服务注册到eureka上，并且可以在线查找服务的API接口文档说明.  
本节加入了spring boot中的监控管理的actuator，API接口文档模块swagger2,sping cloud的DiscoveryClient用户将服务注册到eureka注册中心 

# 服务提供者和服务消费者
“服务提供者”和“服务消费者”的名词是借用的，在Spring Cloud中看到这样的概念。下面这张表格，简单描述了服务提供者/消费者是什么：  
名词  | 概念
---|---
服务提供者 | 服务的被调用方（即：为其他服务提供服务的服务）
服务消费者 | 服务的调用方（即：依赖其他服务的服务）  
## 服务提供者(microservice-provider-userService)
这是一个稍微有点杂的程序。我们使用spring-data-jpa操作h2数据库,用swagger2暴露API接口文档，同时将该服务注册到注册中心Eureka中。  
代码结构：
![代码结构](/images/spring-cloud/userservice/1.png)
1. 在spring-cloud-docker-microservic父项目创建一个Maven子工程，并在pom.xml中添加如下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>spring-cloud-docker-microservice</artifactId>
        <groupId>com.troylc.cloud</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>microservice-provider-userservice</artifactId>
    <version>0.0.2</version>
    <properties>
        <!--程序入口-->
        <start-class>com.troylc.cloud.UserServiceApplication</start-class>
    </properties>
    <dependencies>
        <!--添加spring cloud服务注册的依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
        <!-- 数据库JPA操作-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
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
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
            <groupId>com.spotify</groupId>              <artifactId>docker-maven-plugin</artifactId>
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
2. 配置文件：application.yml  

```
server:
  port: 9514
spring:
  application:
    name: microservice-provider-userservice
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
    com.troylc.cloud: ERROR

eureka:
  instance:
    hostname: userService
    prefer-ip-address: true
    ip-address: ${eureka.instance.hostname} #只有当prefer-ip-address: true 时才生效
    instance-id: ${eureka.instance.hostname}:${server.port}  # 将Instance ID设置成IP:端口的形式
    status-page-url-path: /usersApi       #修改info的地址为API接口页面
#    home-page-url-path: /instance-info
  client:
    serviceUrl:
      defaultZone: http://eadmin:eadmin123@eurekaService1:9511/eureka/,http://eadmin:eadmin123@eurekaService2:9512/eureka/,http://eadmin:eadmin123@eurekaService3:9513/eureka/
    healthcheck:
      enabled: true
```
3. 用户建表语句：schema.sql

```sql
drop table user if exists;
create table user (id bigint generated by default as identity, username varchar(255), age int, address VARCHAR(500), primary key (id));
```
4. 用户测试数据插库语句：data.sql  

```
insert into user (id, username, age, address) values (1,'张三',12,'hunan');
insert into user (id, username, age, address) values (2,'李四',32 ,'beijing');
insert into user (id, username, age, address) values (3,'王五',23,'xiamen');
insert into user (id, username, age, address) values (4,'马六',27,'guangdong');
```
5. 编写实体类

```
package com.troylc.cloud.bean;

import javax.persistence.*;
import java.io.Serializable;

/**
 * Created by troylc on 2017/2/27.
 */
@Entity
@Table(name="user")
public class UserBean implements Serializable{
    /**
     * 用户主键ID，自动增长
     */
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    /**
     * 用户姓名
     */
    @Column
    private String username;

    /**
     * 用户名称
     */
    @Column
    private Integer age;
    /**
     * 用户地址
     */
    @Column
    private String address;

    public Long getId() {
        return this.id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getUsername() {
        return this.username;
    }
    public void setUsername(String username) {
        this.username = username;
    }
    public Integer getAge() {
        return this.age;
    }
    public void setAge(Integer age) {
        this.age = age;
    }
    public String getAddress() {
        return address;
    }
    public void setAddress(String address) {
        this.address = address;
    }
}
```

6. 编写JPA-DAO：

```java
package com.troylc.cloud.dao;

import com.troylc.cloud.bean.UserBean;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

/**
 * Created by troylc on 2017/2/27.
 * 用户持久层JPA接口
 */
@Repository
public interface UserDaoRepository extends JpaRepository<UserBean,Long> {
}
```
7. SERVICE接口与实现  
IUserService:

```java
package com.troylc.cloud.service;
import com.troylc.cloud.bean.UserBean;
import java.util.List;
/**
 * Created by troylc on 2017/2/27.
 * 用户业务逻辑处理层
 */
public interface IUserService {
    /**
     * 添加用户
     * @param userBean
     * @return
     */
     public UserBean saveUser(UserBean userBean) throws Exception ;
    /**
     * 根据ID删除用户
     * @param Id
     * @return
     */
     public boolean deleteUser(Long Id) throws Exception;

    /**
     * 获取所有用户的信息
     * @return
     */
     public List<UserBean> getAllUser() throws Exception;
    /**
     * 根据ID查的对应的用户
     * @param id
     * @return
     */
     public UserBean getUserById(Long id) throws Exception;
    /**
     * 更新用户信息
     * @param userBean
     * @return
     */
     public boolean updateUser(UserBean userBean) throws Exception;
}
```
UserServiceImpl:  

```
package com.troylc.cloud.service.impl;
import com.troylc.cloud.bean.UserBean;
import com.troylc.cloud.dao.UserDaoRepository;
import com.troylc.cloud.service.IUserService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import javax.annotation.Resource;
import java.util.List;
/**
 * 用户业务处理接口实现类
 * Created by troylc on 2017/2/27.
 */
@Service
public class UserServiceImpl implements IUserService{
    private static Logger log = LoggerFactory.getLogger(UserServiceImpl.class);
    @Resource
    private UserDaoRepository userDaoRepository;
    @Override
    @Transactional
    public UserBean saveUser(UserBean userBean) throws Exception {
        return userDaoRepository.save(userBean);
    }
    @Override
    @Transactional
    public boolean deleteUser(Long Id) throws Exception {
        try{
            userDaoRepository.delete(Id);
            return true;
        } catch (Exception e){
             log.error(e.getMessage());
        }
        return false;
    }
    @Override
    public List<UserBean> getAllUser() throws Exception {
        return userDaoRepository.findAll();
    }
    @Override
    public UserBean getUserById(Long id) throws Exception {
        return userDaoRepository.findOne(id);
    }
    @Override
    @Transactional
    public boolean updateUser(UserBean userBean) throws Exception {
        return userDaoRepository.saveAndFlush(userBean)!=null?true:false;
    }
}
```
8. 编写Controller：
UserController：
```
package com.troylc.cloud.controller;

import com.troylc.cloud.bean.UserBean;
import com.troylc.cloud.service.IUserService;
import com.troylc.cloud.utils.ReturnInfoEnum;
import com.troylc.cloud.vbean.ResultInfo;
import io.swagger.annotations.ApiImplicitParam;
import io.swagger.annotations.ApiImplicitParams;
import io.swagger.annotations.ApiOperation;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.*;
import springfox.documentation.annotations.ApiIgnore;

import java.util.ArrayList;
import java.util.List;

/**
 * Rest服务提供者，供其它服务调用
 * Created by troylc on 2017/2/28.
 */
@RestController
public class UserController {

    private static Logger log = LoggerFactory.getLogger(UserController.class);

    @Autowired
    private DiscoveryClient discoveryClient;

    @Autowired
    private IUserService userServiceImpl;


    /**
     * 获取所有用户
     * @return
     */
    @ApiOperation(value = "获取用户列表", notes = "获取所有用户")
    @GetMapping("/users")
    public ResultInfo getUserList() {
        List<UserBean> userBeans = new ArrayList<>();
        try {
            userBeans = userServiceImpl.getAllUser();
        } catch (Exception e) {
            log.error(e.getMessage());
            return new ResultInfo<UserBean>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        return new ResultInfo<List<UserBean>>(ReturnInfoEnum.SUCCESS.getState(),ReturnInfoEnum.SUCCESS.getStateInfo(),userBeans);
    }

    /**
     * 根据User对象创建用户
     * @param user
     * @return
     */
    @ApiOperation(value = "创建用户", notes = "根据User对象创建用户")
    @ApiImplicitParam(name = "user", value = "用户详细实体user", required = true, dataType = "UserBean")
    @PostMapping("/users")
    public ResultInfo postUser(@RequestBody UserBean user) {
        try {
            userServiceImpl.saveUser(user);
        } catch (Exception e) {
            log.error(e.getMessage());
            return new ResultInfo<UserBean>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        return new ResultInfo<UserBean>(ReturnInfoEnum.SUCCESS.getState(), ReturnInfoEnum.SUCCESS.getStateInfo());
    }


    /**
     * 注：@GetMapping("/{id}")是spring 4.3的新注解等价于：
     * @param id
     * @return user信息
     * @RequestMapping(value = "/id", method = RequestMethod.GET)
     * 类似的注解还有@PostMapping等等
     */
    @ApiOperation(value = "查找用户", notes = "根据用户的ID，查找对应的用户")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true, paramType = "path", dataType = "Long")
    @GetMapping("/users/{id}")
    public ResultInfo findById(@PathVariable Long id) {
        UserBean userBean = null;
        try {
            userBean = userServiceImpl.getUserById(id);
        } catch (Exception e) {
            log.error(e.getMessage());
            return new ResultInfo<UserBean>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        ResultInfo resultInfo = new ResultInfo<UserBean>(ReturnInfoEnum.SUCCESS.getState(), ReturnInfoEnum.SUCCESS.getStateInfo(), userBean);
        return resultInfo;
    }

    /**
     * 根据url的id来指定更新对象，并根据传过来的user信息来更新用户详细信息
     * @param id
     * @param user
     * @return
     */
    @ApiOperation(value = "更新用户详细信息", notes = "根据url的id来指定更新对象，并根据传过来的user信息来更新用户详细信息")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id", value = "用户ID", required = true, paramType = "path", dataType = "Long"),
            @ApiImplicitParam(name = "user", value = "用户详细实体user", required = true, dataType = "User")
    })
    @PutMapping("/users/{id}")
    public ResultInfo updateUser(@PathVariable Long id, @RequestBody UserBean user) {
        try {
            user.setId(id);
            userServiceImpl.updateUser(user);
        } catch (Exception e) {
            log.error(e.getMessage());
            return new ResultInfo<UserBean>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        return new ResultInfo<UserBean>(ReturnInfoEnum.SUCCESS.getState(), ReturnInfoEnum.SUCCESS.getStateInfo());
    }

    /**
     * 删除用户
     * @param id
     * @return
     */
    @ApiOperation(value = "删除用户", notes = "根据url的id来指定删除对象")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true, paramType = "path", dataType = "Long")
    @DeleteMapping("/users/{id}")
    public ResultInfo deleteUser(@PathVariable Long id) {
        try {
            userServiceImpl.deleteUser(id);
        } catch (Exception e) {
            log.error(e.getMessage());
            return new ResultInfo<UserBean>(ReturnInfoEnum.SYSTEM_ERROR.getState(), ReturnInfoEnum.SYSTEM_ERROR.getStateInfo());
        }
        return new ResultInfo<UserBean>(ReturnInfoEnum.SUCCESS.getState(), ReturnInfoEnum.SUCCESS.getStateInfo());
    }


    /**
     * 本地服务实例的信息
     * @return
     */
    @ApiIgnore //swagger忽略此API，在前台暴露
    @GetMapping("/instance-info")
    public ServiceInstance showInfo() {
        ServiceInstance localServiceInstance = this.discoveryClient.getLocalServiceInstance();
        return localServiceInstance;
    }

}
```
ApiControoler：

```java
package com.troylc.cloud.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import springfox.documentation.annotations.ApiIgnore;
/**
 * 用户服务API转向
 * Created by troylc on 2017/3/1.
 */
@Controller
public class ApiControoler {
    /**
     * 在服务注册中心点击该服务重定向到api接口中心
     *
     * @return
     */
    @ApiIgnore
    @GetMapping("/usersApi")
    public String redirectApi() {
        return "redirect:/swagger-ui.html";
    }
}
```
9. 编写Swagger2 API接口配置

```java
package com.troylc.cloud.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 * Created by Administrator on 2016/12/1.
 */

@Configuration
@EnableSwagger2
public class Swagger2 {


    /**
     * 访问地址 http://ip:prot/swagger-ui.html
     * @return
     */
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())       //创建API基本信息
                .groupName("controller API")     //指定分组，对应(/v2/api-docs?group=)
                .pathMapping("")  //base地址，最终会拼接Controller中的地址
                .select()    //控制要暴露的接口
                .apis(RequestHandlerSelectors.basePackage("com.troylc.cloud.controller"))  //通过指定扫描包暴露接口
                .paths(PathSelectors.any())       //设置过滤规则暴露接口
                .build();
    }
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("使用Swagger2构建用户RESTful APIs")
                .description("更多相关文章，请关注：http://www.troylc.cc/")
                .termsOfServiceUrl("http://www.troylc.cc")
                .contact("troylc")
                .version("1.0")
                .build();
    }
}
```
10. 编写Spring Boot启动程序，通过@EnableDiscoveryClient注解，即可将服务注册到Eureka上面去

```java
package com.troylc.cloud;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(UserServiceApplication.class, args);
	}
}
```
11.定义JSON返回信息枚举类和返回类型封装类
ReturnInfoEnum：

```java
package com.troylc.cloud.utils;

/**
 * 定义返回信息枚举
 * Created by troylc on 2017/2/28.
 */
public enum ReturnInfoEnum {

    SUCCESS(1, "请求操作成功"),
    NULL(0, "没有你请求的数据"),
    PARAMETER_ERROR(-1, "请求数据失败，参数错误"),
    SYSTEM_ERROR(-2, "请求数据失败，系统异常"),
    NOT_AUTHENTICATE(-3, "请示数据失败，未认证");

    private int state;

    private String stateInfo;

    ReturnInfoEnum(int state, String stateInfo) {
        this.state = state;
        this.stateInfo = stateInfo;
    }

    public int getState() {
        return state;
    }

    public String getStateInfo() {
        return stateInfo;
    }

    public static ReturnInfoEnum stateOf(int index) {
        for (ReturnInfoEnum state : values()) {
            if (state.getState() == index) {
                return state;
            }
        }
        return null;
    }
}

```
ResultInfo：

```
package com.troylc.cloud.vbean;

import java.io.Serializable;

/**
 * 所有的Rest请求的返回类型封装JSON结果
 * Created by troylc on 2017/2/28.
 */
public class ResultInfo<T> implements Serializable {

    private int success;

    private T data;

    private String mesagess;

    public ResultInfo(int success, String mesagess) {
        this.success = success;
        this.mesagess = mesagess;
    }

    public ResultInfo(int success, String mesagess, T data) {
        this.success = success;
        this.mesagess = mesagess;
        this.data = data;
    }

    public int isSuccess() {
        return success;
    }

    public void setSuccess(int success) {
        this.success = success;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }


    public String getMesagess() {
        return mesagess;
    }

    public void setMesagess(String mesagess) {
        this.mesagess = mesagess;
    }

    @Override
    public String toString() {
        return "ResultInfo{" +
                "success=" + success +
                ", data=" + data +
                ", error='" + mesagess + '\'' +
                '}';
    }
}
```
12. 修改编排部署文件docker-compose文件

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
networks:
  eureka-net:
    driver: overlay

```

至此代码部署完成，其中需要注意的地方是，在加入swagger2的依赖包时，版本需为2.5.0,如果是2.6.0,注册在eureka上的用户服务的端口不会根据application.yml中配置的一样。而是会出现tomcat的默认端口8080


## 打包及部署操作
![打包](/images/spring-cloud/userservice/2.png)  
将docker-compose.yml拷贝到swarm集群的master节点上，执行如下操作：
```
[root@docker-master01 docker-compose]# docker stack deploy -c docker-compose.yml eureka
Creating network eureka_eureka-net
Creating service eureka_userService
Creating service eureka_eurekaService1
Creating service eureka_eurekaService2
Creating service eureka_eurekaService3
[root@docker-master01 docker-compose]# docker stack ps eureka
ID            NAME                     IMAGE                                                   NODE             DESIRED STATE  CURRENT STATE           ERROR  PORTS
izc9zknt5vay  eureka_eurekaService3.1  tcr:5000/myhub/microservice-eureka-service:0.0.1        docker-master01  Running        Running 17 seconds ago         
losked39x42d  eureka_eurekaService2.1  tcr:5000/myhub/microservice-eureka-service:0.0.1        docker-node02    Running        Running 15 seconds ago         
jaygfei03isr  eureka_eurekaService1.1  tcr:5000/myhub/microservice-eureka-service:0.0.1        docker-node02    Running        Running 18 seconds ago         
mvv6lzvlq072  eureka_userService.1     tcr:5000/myhub/microservice-provider-userservice:0.0.2  docker-node01    Running        Running 19 seconds ago  


```
## 测试用户服务注册情况
![eureka注册中心](/images/spring-cloud/userservice/3.png)  
![API接口](/images/spring-cloud/userservice/4.png)

服务提供者用户服务已经注册到eureka上，后续服务消费者对用户服务进行消费，敬请期待.......

**附代码仓库地址：**  
码云：[https://git.oschina.net/gittroylc/spring-cloud-docker-microservice](https://git.oschina.net/gittroylc/spring-cloud-docker-microservice)  
github: [https://github.com/troychn/spring-cloud-docker-microservice](https://github.com/troychn/spring-cloud-docker-microservice)



