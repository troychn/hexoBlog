---
title: docker及swarm-mode之docker-ui实践-portainer  
toc: true   // 在文章侧边增加文章目录
date: 9/16/2017 5:26:15 PM 
updated: 9/16/2017 5:26:20 PM 
categories: [docker]
tags: [docker,swarm,docker-ui,linux]

---

portainer（基于 Go） 是一个轻量级的、开放源码的轻量级管理界面，可让您轻松管理DOCKER主机或DOCKER-SWARM集群,经过对portainer的部署与操作，个人觉得满足基本docker操作的需求，portainer支持容器、镜像、volume、network的管理，支持权限分配，支持应用容器模板操作、支持集群的简单管理，支持TLS的证书认证，支持本地用户与组和LDAP的权限管理；而且该软件轻量，消耗资源少。

参考：
github: [https://github.com/portainer/portainer](https://github.com/portainer/portainer)  

doc: [http://portainer.readthedocs.io/en/latest/](http://portainer.readthedocs.io/en/latest/)   

#部署portainer
本篇主要是在swarm mode的环境上，通过部署一个portainer的service来运行。

```bash
[root@docker-master ~]# docker service create \
>     --name portainer \
>     --publish 9000:9000 \
>     --constraint 'node.role == manager' \
>     --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
>     --mount type=bind,src=/nfs-data/swarmui/data,dst=/data \
>     portainer/portainer:latest \
>     -H unix:///var/run/docker.sock
ykapxftktk0xdsdgrftgfvht
[root@docker-master ~]# docker service ps portainer
ID            NAME         IMAGE                       NODE           DESIRED STATE  CURRENT STATE         ERROR  PORTS
ykapxftktk0x  portainer.1  portainer/portainer:latest  docker-master  Running        Running 19 hours ago 
```

**相关参数：**

* `--host，-H`：Docker守护程序端点
* `--bind，-p`：地址和端口提供Portainer（默认值：:9000）
* `--data，-d`：Portainer数据将被存储的目录（默认：/data在Linux上，C:\data在Windows上）
* `--tlsverify`：支持TLS（默认false）
* `--tlscacert`：CA路径（默认值：/certs/ca.pem在Linux上，C:\certs\ca.pem在Windows上）
* `--tlscert`：的路径TLS证书文件（默认：/certs/cert.pem，C:\certs\cert.pem在Windows上）
* `--tlskey`：TLS键的路径（默认值：/certs/key.pem，C:\certs\key.pem在Windows上）
* `--no-analytics`：禁用分析（默认值：false）
* `--no-auth`：禁止内部认证机制（默认值：false）
* `--external-endpoints`：通过指定文件中JSON端点源的路径来启用外部端点管理
* `--sync-interval`：两个端点的同步请求之间的时间间隔表示为一个字符串，例如30s，5m，1h...利用所支持的time.ParseDuration方法（默认：60s）
* `--admin-password`：表单中的管理员密码 admin:<hashed_password>
* `--ssl`：使用SSL安全Portainer实例（默认值：false）
* `--sslcert`：路径的SSL证书用于保护Portainer实例（默认：/certs/portainer.crt，C:\certs\portainer.crt在Windows上）
* `--sslkey`：路径用来固定Portainer实例的SSL密钥（默认/certs/portainer.key，C:\certs\portainer.key在Windows上）

# 运行portainer
在浏览器中输入http://[remote.ip]:9000,第一次会提示设置密码，设置完后进行登录界面
![portainer登录面](/images/docker/docker-ui/portainer/15055481781929.jpg)

## docker主机运行管理

- 主页Dashboard：

![主页Dashboard](/images/docker/docker-ui/portainer/15055482535001.jpg)

- 模板管理

![模板管理](/images/docker/docker-ui/portainer/15055525167399.jpg)

根据官方文档，我们可以自己创建一个本地模板，参考：[构建自己的模板](https://portainer.readthedocs.io/en/stable/templates.html)

```bash
$ git clone https://github.com/portainer/templates.git portainer-templates
$ cd portainer-templates
# Edit the file templates.json
$ docker build -t portainer-templates .
$ docker run -d -p "8080:80" portainer-templates

```

- swarm service 管理页：

![管理管理](/images/docker/docker-ui/portainer/15055487334364.jpg)
点击上面的add service,进入服务的添加界面：
![添加服务](/images/docker/docker-ui/portainer/15055487862443.jpg)
创建服务的相关参数可以直接参考docker官方文档，portainer也是通过docker API进行操作的。

- manager节点的容器管理：

![pwkk容器管理](/images/docker/docker-ui/portainer/15055490209966.jpg)
可手动选择指定的容器进行：start、stop、kill、restart、pause、resume、remove。
点击Add container,可以单独在manager的管理节点上运行一个容器：
![运行容器](/images/docker/docker-ui/portainer/15055492476988.jpg)
点击具体的一个容器可以看到针对这个容器的一些配置信息和配置修改
![容器配置](/images/docker/docker-ui/portainer/15055494365002.jpg)
查看单一容器的运行情况：
![](/images/docker/docker-ui/portainer/15055495521281.jpg)
![cpu、内存、网络、进程简单w分析](/images/docker/docker-ui/portainer/15055507618067.jpg)
![日志打印](/images/docker/docker-ui/portainer/15055508641131.jpg)
![进入控制台方式](/images/docker/docker-ui/portainer/15055509161527.jpg)
![控制台](/images/docker/docker-ui/portainer/15055509908660.jpg)

- manager节点本地镜像管理：

![manager节点本地镜像管理](/images/docker/docker-ui/portainer/15055510798802.jpg)

- 本地及集群网络管理：

![本地及集群网络管理](/images/docker/docker-ui/portainer/15055512402124.jpg)

- 本地存储管理

![存储管理](/images/docker/docker-ui/portainer/15055513062965.jpg)

- docker swarm中的Secrets管理

![Secrets管理](/images/docker/docker-ui/portainer/15055516378425.jpg)

- docker swarm 管理

![swarm 管理](/images/docker/docker-ui/portainer/15055516936067.jpg)

## portainer设置管理

- 本地用户及用户组管理
![本地用户及用户组管理](/images/docker/docker-ui/portainer/15055519875461.jpg)

- docker节点管理
![](/images/docker/docker-ui/portainer/15055521110780.jpg)

- 本地镜像仓库管理
![本地镜像仓库管理](/images/docker/docker-ui/portainer/15055521435740.jpg)
![添加仓库](/images/docker/docker-ui/portainer/15055521780267.jpg)

把本地创建好的镜像仓库，添加到这里。参考[docker-registry私有仓库及web ui镜像管理](http://www.troylc.cc/docker/2017/06/20/docker-registry-web.html)

- 其它设置

![logo、本地模板、标签设置](/images/docker/docker-ui/portainer/15055524234336.jpg)

- LDAP认证

![LDAP认证](/images/docker/docker-ui/15055557567875.jpg)




