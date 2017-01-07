---
title: docker系列(二)使用Docker-Remote-API
toc: true   # 在文章侧边增加文章目录
date: 7/31/2016 1:10:36 PM
updated: 7/31/2016 1:10:40 PM
categories: [docker]
tags: [linux,docker]

---
### 前言
Docker Remote API是一个取代远程命令行界面（rcli）的REST API。我们将使用命令行工具curl来处理url相关操作。curl可以发送请求、获取以及发送数据、检索信息。是docker自带的一个rest api 管理docker所有的操作都有对应的http rest API可供操作。下面简单说一下API的操作

### 正文
#### 环境

|  主机 |  安装软件 |
| ------------ | ------------ |
| 192.168.253.129  | 安装docker，打开docker的API访问端口，主机  |
| 192.168.253.131  |  安装docker,远程通过API访问docker主机的客户端 |

#### 配置(192.168.253.129)启动Remote API

```bash
[root@localhost ~]# vim /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target
Wants=docker-storage-setup.service

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
Environment=GOTRACEBACK=crash
ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
MountFlags=slave
TimeoutStartSec=1min
[Install]
WantedBy=multi-user.target
```
说明：
上面那句话，在不同的版本，设置有可能不一样，
重启docker
[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl restart docker

在本机(192.168.253.129)测试：
```bash
[root@localhost ~]# docker -H 192.168.253.129:2375 images
REPOSITORY                              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
myhub/mysql                             latest              d298157e388e        2 days ago          359.9 MB
troylc/maven                            latest              0d244f5623e3        4 days ago          831.8 MB
jenkins                                 latest              988e2e1b7418        9 days ago          707.8 MB
registry                                latest              a8706c2bfd21        10 days ago         422.8 MB
maven                                   latest              e6a340bb7d58        2 weeks ago         651.9 MB
kubetomcat                              latest              d7b68af6b003        4 weeks ago         395.6 MB
kubenginx                               latest              3820887c73d5        4 weeks ago         223.4 MB
dmptomcat                               latest              82775047bc5e        6 weeks ago         395.5 MB
toptomcat                               latest              06666482da5f        6 weeks ago         395.5 MB
daocloud.io/daocloud/daocloud-toolset   latest              4a115833bcba        6 weeks ago         145.8 MB
topsecssh                               latest              ac2ba2efcaf2        6 weeks ago         302.5 MB
topsecnginx                             latest              0694365f1f87        6 weeks ago         223.4 MB
tomcat                                  latest              e4b99e523705        7 weeks ago         347.8 MB
mysql                                   latest              196db1908492        7 weeks ago         359.8 MB
nginx                                   latest              5135500ec6a1        7 weeks ago         132.7 MB
redis                                   latest              05babbd460f7        8 weeks ago         109.1 MB
ubuntu                                  latest              a5a467fddcb8        8 weeks ago         187.9 MB
centos                                  latest              ce20c473cd8a        9 weeks ago         172.3 MB
gcr.io/google_containers/pause          0.8.0               2c40b0526b63        8 months ago        241.7 kB
```
在docker客户端机(192.168.253.130)上访问
```bash
[root@localhost ~]# docker -H 192.168.253.129:2375 images
REPOSITORY                              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
myhub/mysql                             latest              d298157e388e        2 days ago          359.9 MB
troylc/maven                            latest              0d244f5623e3        4 days ago          831.8 MB
jenkins                                 latest              988e2e1b7418        9 days ago          707.8 MB
registry                                latest              a8706c2bfd21        10 days ago         422.8 MB
maven                                   latest              e6a340bb7d58        2 weeks ago         651.9 MB
kubetomcat                              latest              d7b68af6b003        4 weeks ago         395.6 MB
kubenginx                               latest              3820887c73d5        4 weeks ago         223.4 MB
dmptomcat                               latest              82775047bc5e        6 weeks ago         395.5 MB
toptomcat                               latest              06666482da5f        6 weeks ago         395.5 MB
daocloud.io/daocloud/daocloud-toolset   latest              4a115833bcba        6 weeks ago         145.8 MB
topsecssh                               latest              ac2ba2efcaf2        6 weeks ago         302.5 MB
topsecnginx                             latest              0694365f1f87        6 weeks ago         223.4 MB
tomcat                                  latest              e4b99e523705        7 weeks ago         347.8 MB
mysql                                   latest              196db1908492        7 weeks ago         359.8 MB
nginx                                   latest              5135500ec6a1        7 weeks ago         132.7 MB
redis                                   latest              05babbd460f7        8 weeks ago         109.1 MB
ubuntu                                  latest              a5a467fddcb8        8 weeks ago         187.9 MB
centos                                  latest              ce20c473cd8a        9 weeks ago         172.3 MB
gcr.io/google_containers/pause          0.8.0               2c40b0526b63        8 months ago        241.7 kB


[root@localhost ~]# docker -H 192.168.253.129:2375 run -d -p 80:80 -p 443:443 -p 5005:22 -v /home/nfs/data:/home/nfs/data --name dmpnginx nginxa
997f216e969a72146b52f916ec20b2c4b892c4ea86acaa329cfd7028d974389
[root@localhost ~]# docker -H 192.168.253.129:2375 run -d -p 6379:6379 --name dmpredis redis
f74477908b1767a35f8666b164c60399943bc08964423fea967a364fe79f3fea
[root@localhost ~]# docker -H 192.168.253.129:2375 ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED                  STATUS              PORTS           
                                                 NAMESf74477908b17        redis               "/entrypoint.sh redis"   Less than a second ago   Up 8 seconds        0.0.0.0:6379->63
79/tcp                                           dmpredisa997f216e969        nginx               "nginx -g 'daemon off"   Less than a second ago   Up 32 seconds       0.0.0.0:80->80/t
cp, 0.0.0.0:443->443/tcp, 0.0.0.0:5005->22/tcp   dmpnginx
```
docker宿主机上的操作，都可以通过（docker -H 192.168.253.129:2375 命令）的方式在其它docker客户机上操作。



#### 使用远程API构建镜像，运行容器，停止容器，删除容器等。
##### 使用info接入点(类似在宿主机上输入docker info)
curl http://192.168.253.129:2375/info
![dockerinfo](/images/docker/clipboard.png)

##### 通过API获取远程docker主机上的镜像列表（类似于输入docker images)
```bash
[root@localhost ~]# curl http://192.168.253.129:2375/images/json | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  5504    0  5504    0     0  43744      0 --:--:-- --:--:-- --:--:-- 44032
[
    {
        "Created": 1450623232,
        "Id": "03b5e135264764e8b51a2f13f174e541ec3b8952d64762495e369cbb6ca9bdad",
        "Labels": null,
        "ParentId": "711aef0427ed0b2f77486a8c884294319b5aaca96c698d288a56d71148e6c0b4",
        "RepoDigests": [],
        "RepoTags": [
            "myhub/education:latest"
        ],
        "Size": 0,
        "VirtualSize": 506413396
    },
    {
        "Created": 1450376000,
        "Id": "d298157e388ed0e6edd000a1234b3f84abbc1463a8f0ece6f2c539034a1e2573",
        "Labels": null,
        "ParentId": "ac704f6f8363d123001de9dd604e4cb1b476d012f0a809c4bf86e0cd60631753",
        "RepoDigests": [],
        "RepoTags": [
            "myhub/mysql:latest"
        ],
        "Size": 0,
        "VirtualSize": 359871512
    },
    {
        "Created": 1450215230,
        "Id": "0d244f5623e3dc64d7f6428069c5dea3cd77299068b3ed2cb3d4eb56fd6f21b4",
        "Labels": null,
        "ParentId": "7605008fbf7d7470810df0c41f6422db254181f4f2eb2f02827f3fb62c9f79b3",
        "RepoDigests": [],
        "RepoTags": [
            "troylc/maven:latest"
        ],
        "Size": 87225781,
        "VirtualSize": 831831039
    },
    {
        "Created": 1449771164,
        "Id": "988e2e1b74182c986873eb4b5ebce2f9dcc8f9ec156e41ec881aaa1074c7d489",
        "Labels": null,
        "ParentId": "d67b67d804fff392fd88e2ba601b6e3cafc117a89c8783f1bf0780b58bc19713",
        "RepoDigests": [],
        "RepoTags": [
            "jenkins:latest"
        ],
        "Size": 861,
        "VirtualSize": 707786365
    }
]
```

##### 获取指定的镜像：
```bash
[root@localhost ~]# curl http://192.168.253.129:2375/images/2c40b0526b6358710fd09e7b8c022429268cc61703b4777e528ac9d469a07ca1/json | python -mjson.tool  
#2c40b0526b6358710fd09e7b8c022429268cc61703b4777e528ac9d469a07ca1为镜像的ID号，可以通过上面的（curl http://192.168.253.129:2375/images/json | python -mjson.tool）查询获得。
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1661  100  1661    0     0  83337      0 --:--:-- --:--:-- --:--:-- 87421
{
    "Architecture": "amd64",
    "Author": "",
    "Comment": "",
    "Config": {
        "AttachStderr": false,
        "AttachStdin": false,
        "AttachStdout": false,
        "Cmd": null,
        "Domainname": "",
        "Entrypoint": [
            "/pause"
        ],
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "ExposedPorts": null,
        "Hostname": "d0eb442be084",
        "Image": "56ba5533a2dbf18b017ed12d99c6c83485f7146ed0eb3a2e9966c27fc5a5dd7b",
        "Labels": null,
        "MacAddress": "",
        "NetworkDisabled": false,
        "OnBuild": [],
        "OpenStdin": false,
        "PublishService": "",
        "StdinOnce": false,
        "Tty": false,
        "User": "",
        "VolumeDriver": "",
        "Volumes": null,
        "WorkingDir": ""
    },
    "Container": "c63e747118f6582698545c10277ce8412088258b62601908ed5f0f2accfeb2b3",
    "ContainerConfig": {
        "AttachStderr": false,
        "AttachStdin": false,
        "AttachStdout": false,
        "Cmd": [
            "/bin/sh",
            "-c",
            "#(nop) ENTRYPOINT [/pause]"
        ],
        "Domainname": "",
        "Entrypoint": [
            "/pause"
        ],
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "ExposedPorts": null,
        "Hostname": "d0eb442be084",
        "Image": "56ba5533a2dbf18b017ed12d99c6c83485f7146ed0eb3a2e9966c27fc5a5dd7b",
        "Labels": null,
        "MacAddress": "",
        "NetworkDisabled": false,
        "OnBuild": [],
        "OpenStdin": false,
        "PublishService": "",
        "StdinOnce": false,
        "Tty": false,
        "User": "",
        "VolumeDriver": "",
        "Volumes": null,
        "WorkingDir": ""
    },
    "Created": "2015-03-31T22:05:45.140369385Z",
    "DockerVersion": "1.3.3",
    "GraphDriver": {
        "Data": {
            "DeviceId": "763",
            "DeviceName": "docker-253:0-34749673-2c40b0526b6358710fd09e7b8c022429268cc61703b4777e528ac9d469a07ca1",
            "DeviceSize": "107374182400"
        },
        "Name": "devicemapper"
    },
    "Id": "2c40b0526b6358710fd09e7b8c022429268cc61703b4777e528ac9d469a07ca1",
    "Os": "linux",
    "Parent": "56ba5533a2dbf18b017ed12d99c6c83485f7146ed0eb3a2e9966c27fc5a5dd7b",
    "Size": 0,
    "VirtualSize": 241656
}
```

##### 通过API搜索镜像（类似于docker search:查询的是docker hub上的镜像）
```bash
[root@localhost ~]#  curl "http://192.168.253.129:2375/images/search?term=tomcat" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  5130    0  5130    0     0   2513      0 --:--:--  0:00:02 --:--:--  2513
[
    {
        "description": "Apache Tomcat is an open source implementation of the Java Servlet and JavaServer Pages technologies",
        "index_name": "docker.io",
        "is_automated": false,
        "is_official": true,
        "is_trusted": false,
        "name": "tomcat",
        "registry_name": "docker.io",
        "star_count": 390
    },
    {
        "description": "Tomcat 7.0.57, 8080, \"admin/admin\"",
        "index_name": "docker.io",
        "is_automated": true,
        "is_official": false,
        "is_trusted": true,
        "name": "consol/tomcat-7.0",
        "registry_name": "docker.io",
        "star_count": 14
    },
    {
        "description": "Tomcat 8.0.15, 8080, \"admin/admin\"",
        "index_name": "docker.io",
        "is_automated": true,
        "is_official": false,
        "is_trusted": true,
        "name": "consol/tomcat-8.0",
        "registry_name": "docker.io",
        "star_count": 12
    }
]
```

##### 列出正在运行的容器（docker ps 和docker ps -a)
```bash
[root@localhost ~]#
curl -s "http://192.168.253.129:2375/containers/json"| python -mjson.tool(docker ps)
curl http://192.168.253.129:2375/containers/json?all=1 | python -mjson.tool(docker ps -a)
[
    {
        "Command": "nginx",
        "Created": 1436573284,
        "Id": "e5cac80914cbd41f390b0892324d71505e50ac35485b89374030ec3e65ae470c",
        "Image": "pengji/nginx:latest",
        "Labels": {},
        "Names": [
            "/website"
        ],
        "Ports": [
            {
                "IP": "0.0.0.0",
                "PrivatePort": 80,
                "PublicPort": 32768,
                "Type": "tcp"
            }
        ],
        "Status": "Up 57 seconds"
    }
]
```

##### 创建与启动容器
创建容器
```bash
[root@localhost ~]#  curl -X POST -H "Content-Type: application/json""http://192.168.253.129:2375/containers/create"-d '{"Image":"pengji/nginx","Hostname":"remote_nginx"}'
{"Id":"91784daaf79e87652142f9293dd82ef5716968db5b251ec761566759c0c529c6","Warnings":null}
```
启动容器
```bash
[root@localhost ~]#  curl -X POST -H "Content-Type: application/json""http://192.168.253.129:2375/containers/91784daaf79e87652142f9293dd82ef5716968db5b251ec761566759c0c529c6/start"-d '{"PublishAllPorts":true}'
Usage of loopback devices is strongly discouraged forproduction use. Either use `--storage-opt dm.thinpooldev` or use `--storage-opt dm.no_warn_on_loop_devices=true` to suppress this warning.
```

### 参考：


