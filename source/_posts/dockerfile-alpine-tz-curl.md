---
title: Dockerfile制作openjdk-8u131-jdk-alpine基础镜像
toc: true   // 在文章侧边增加文章目录
date: 7/28/2017 5:26:15 PM 
updated: 7/28/2017 5:26:20 PM 
categories: [docker]
tags: [docker,linux,Dockerfile]

---

从dockerhub上下载openjdk:8u131-jdk-alpine本身是一个轻量级的操作系统，只带有jre，不带任何jre外的软件，如果想增加其它软件就需要我们自己在此基础上再构建自己想的基础镜像。

## 下载openjdk:8u131-jdk-alpine基础镜像

```bash
[root@registry-jenkins ~]# docker images
REPOSITORY                      TAG                    IMAGE ID            CREATED             SIZE
openjdk-8-time                  latest                 ab859b2c2910        14 hours ago        107.8 MB
nginx                           1.13.3-alpine          ba60b24dbad5        2 weeks ago         15.51 MB
openjdk                         8u131-jdk-alpine       478bf389b75b        4 weeks ago         101 MB

```
本次在openjdk:8u131-jdk-alpine基础上增加时区文件和CURL、TREE等工具。

-------

## 编写Dockerfile文件，并构建alpine基础镜像

由于docker构建的时候要到alpine官网上下载软件，因我的构建服务器不能上外网，所在在构建的时候特殊增加http_proxy代理。

```
[root@registry-jenkins ~]# vim Dockerfile
FROM openjdk:8u131-jdk-alpine
# MAINTAINER指令允许你给将要制作的镜像设置作者信息
MAINTAINER iucheng <liu_cheng@topsec.com.cn>

#ADD cacerts /etc/ssl/certs/java/cacerts

ARG http_proxy

ENV http_proxy=${http_proxy}
ENV https_proxy=${http_proxy}

# 设置时区 中国的时区有多种表述 分别为: UTC+8:00 GMT+8 # 写/etc/TZ, 不要设置TZ环境变量 ENV TZ UTC+8:00
RUN  apk update \
    && apk add --no-cache \
    && apk add curl bash tree tzdata && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

ENV http_proxy=
ENV https_proxy=

[root@registry-jenkins ~]# docker build --build-arg http_proxy=http://192.168.72.188:808 -t openjdk:8u131-jdk-tz-curl-alpine .
Sending build context to Docker daemon 49.15 kB
Step 1 : FROM openjdk:8u131-jdk-alpine
 ---> 478bf389b75b
Step 2 : MAINTAINER iucheng <liu_cheng@topsec.com.cn>
 ---> Using cache
 ---> 0fe3b8ac71b7
Step 3 : ARG http_proxy
 ---> Running in 2069da05a659
 ---> 74c725e41083
Removing intermediate container 2069da05a659
Step 4 : ENV http_proxy ${http_proxy}
 ---> Running in ac0bf48d4b6c
 ---> 8c058688cfa8
Removing intermediate container ac0bf48d4b6c
Step 5 : ENV https_proxy ${http_proxy}
 ---> Running in 55e645b7c4b8
 ---> df34875db960
Removing intermediate container 55e645b7c4b8
Step 6 : RUN apk update     && apk add --no-cache     && apk add curl bash tree tzdata &&     ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&     echo "Asia/Shanghai" > /etc/timezone
 ---> Running in 387ca24a223a
fetch http://dl-cdn.alpinelinux.org/alpine/v3.6/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.6/community/x86_64/APKINDEX.tar.gz
v3.6.2-40-gf1c202674f [http://dl-cdn.alpinelinux.org/alpine/v3.6/main]
v3.6.2-32-g6f53cfcccd [http://dl-cdn.alpinelinux.org/alpine/v3.6/community]
OK: 8436 distinct packages available
fetch http://dl-cdn.alpinelinux.org/alpine/v3.6/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.6/community/x86_64/APKINDEX.tar.gz
OK: 99 MiB in 51 packages
(1/10) Installing ncurses-terminfo-base (6.0-r7)
(2/10) Installing ncurses-terminfo (6.0-r7)
(3/10) Installing ncurses-libs (6.0-r7)
(4/10) Installing readline (6.3.008-r5)
(5/10) Installing bash (4.3.48-r1)
Executing bash-4.3.48-r1.post-install
(6/10) Installing libssh2 (1.8.0-r1)
(7/10) Installing libcurl (7.54.0-r0)
(8/10) Installing curl (7.54.0-r0)
(9/10) Installing tree (1.7.0-r0)
(10/10) Installing tzdata (2017a-r0)
Executing busybox-1.26.2-r5.trigger
OK: 111 MiB in 61 packages
 ---> d2ed313ab5f3
Removing intermediate container 387ca24a223a
Step 7 : ENV http_proxy
 ---> Running in 104af5cd52df
 ---> 7c2f38e0d8fd
Removing intermediate container 104af5cd52df
Step 8 : ENV https_proxy
 ---> Running in d93dd0df2c4a
 ---> 4ad2436778d3
Removing intermediate container d93dd0df2c4a
Successfully built 4ad2436778d3

```
构建命中中增加了`--build-arg http_proxy=http://192.168.72.188:808`，主要是为了构建过程中通过代理上外网，从外网下载所需的软件。

## 验证构建的镜像是否成功

```
root@registry-jenkins ~]# docker run -it --rm --name jdk-test openjdk:8u131-jdk-tz-curl-alpine /bin/bash
bash-4.3# date
Fri Jul 28 09:44:59 CST 2017
bash-4.3# curl
curl: try 'curl --help' or 'curl --manual' for more information
bash-4.3# exit
exit
[root@registry-jenkins ~]#
```

