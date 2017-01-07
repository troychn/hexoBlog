---
title: docker系列-docker、docker-compse最新版本安装及版本升级
toc: true
date: 2017-01-05 16:10:45 Thursday
updated: 2017-01-05 16:10:53 Thursday
categories: [docker]
tags: [linux,docker]

---

### 前言
一直由于工作的原因，在2016年下半年几乎停止了更新自己的博客，新年依始，勿忘初心，重新更新了自己的博客空间主题，虽有不足，总算是顺利完成了更新。 希望2017更进一步。
以下为docker1.12.x和docker-compose1.9的安装

#### - **环境信息**

| 操作系统  | 软件  | 说明  |
| ------------ | ------------ | ------------ | 
| centos7.2  | docker engine 1.12  | Docker是一个能够把开发的应用程序自动部署到容器的开源引擎  |
| centos7.2  | docker-compose 1.9  | 是 Docker 官方编排（Orchestration）项目之一，负责快速在集群中部署分布式应用  |

#### - **系统环境安装**  

升级这台主机的内核到最新版本。为安装最新的docker版本
升级内核(在连网的环境下)

```bash
[root@dmpdev ~]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
[root@dmpdev ~]# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
[root@dmpdev ~]# yum --enablerepo=elrepo-kernel install kernel-ml
```

也可以通过此http://pkgs.org/download/kernel-devel网站手工下载安装
重要：目前内核还是默认的版本，如果在这一步完成后你就直接reboot了，重启后使用的内核版本还是默认的3.10，不会使用新的4.X以上，想修改启动的顺序，需要进行下一步
查看默认启动顺序

```
[root@dmpdev ~]# awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
```

默认启动的顺序是从0开始，但我们新内核是从头插入（目前位置在0，而3.10的是在1），所以需要选择0，如果想生效最新的内核，需要

```
[root@dmpdev ~]# grub2-set-default 0   我这里是1
```

#### - **docker环境安装**
##### 1. 确认旧的docker相关的组件并删除  
如果你的机器上有用centos简易安装方式yum install docker安装的各种docker组件。安装1.12之前先把它们删掉吧，不然后面有可能还是会提示你删除的。（适合想对docker进行升级的情况，先删除，再安装最新的，放心删除docker engine,他不会删除你本地的镜像和容器，一但是重新安装好后，原来的镜像容器同样可以用，这点已经实验了，大家放心删除）

```
[root@dmpdev ~]# rpm -qa |grep docker
docker-selinux-1.10.3-44.el7.centos.x86_64
docker-common-1.10.3-44.el7.centos.x86_64
docker-forward-journald-1.10.3-44.el7.centos.x86_64
docker-1.10.3-44.el7.centos.x86_64
[root@dmpdev ~]# yum -y remove docker-selinux-1.10.3-44.el7.centos.x86_64
[root@dmpdev ~]# yum -y remove docker-common-1.10.3-44.el7.centos.x86_64
[root@dmpdev ~]# yum -y remove docker-forward-journald-1.10.3-44.el7.centos.x86_64
[root@dmpdev ~]# yum -y remove docker-1.10.3-44.el7.centos.x86_64
```

##### 2. 安装docker 1.12.x版本  

###### - 设定docker的Yum源来安装  

docker缺省的Yum库使用的是main，基本上是稳定的版本。而不是最新版本的docker。在centos上安装只需要设定为experimental。将其baseurl设定为https://yum.dockerproject.org/repo/experimental/centos/7/即可。以后升到1.99估计也可以用同样的花招抢先试用吧。以下为设定方式：

```
[root@dmpdev ~]# cat > /etc/yum.repos.d/docker.repo <<-EOF
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/experimental/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

[root@dmpdev ~]# yum -y install docker-engine
省略安装内容.......
已安装:
  docker-engine.x86_64 0:1.12.1-1.el7.centos                                                                                                    
作为依赖被安装:
  audit-libs-python.x86_64 0:2.4.1-5.el7    checkpolicy.x86_64 0:2.1.12-6.el7    docker-engine-selinux.noarch 0:1.12.1-1.el7.centos libcgroup.x86_64 0:0.41-8.el7  libseccomp.x86_64 0:2.2.1-1.el7   
  libsemanage-python.x86_64 0:2.1.10-18.el7 libtool-ltdl.x86_64 0:2.4.2-21.el7_2 policycoreutils-python.x86_64 0:2.2.5-20.el7       python-IPy.noarch 0:0.75-6.el7 setools-libs.x86_64 0:3.3.7-46.el7
[root@dmpdev ~]# systemctl start docker 启动 docker
[root@dmpdev ~]# systemctl enable docker 设置开机启动
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.

```

###### - 通过CURL或者WGET运行bash命令来安装：

```
[root@dmpdev ~]# curl -fsSL https://get.docker.com/ | sh 或者 wget -qO- https://get.docker.com/ |sh
[root@dmpdev ~]# systemctl restart docker
[root@dmpdev ~]# docker version
[root@dmpdev ~]# docker version
Client:
 Version:      1.12.3
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   6b644ec
 Built:        
 OS/Arch:      linux/amd64

Server:
 Version:      1.12.3
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   6b644ec
 Built:        
 OS/Arch:      linux/amd64
 [root@dmpdev ~]# systemctl start docker 启动 docker
[root@dmpdev ~]# systemctl enable docker 设置开机启动
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

#### - docker-compose环境安装  

Docker Compose是一个一次部署多个容器的简单但是非常必要的工具，

##### - PIP安装

```
[root@dmpdev ~]# yum install python-pip -y
[root@dmpdev ~]# pip install docker-compose
[root@dmpdev ~]# docker-compose -version
docker-compose version 1.9.0, build 2585387
```


##### - curl下载二进制可执行文件的方式(推荐)  

安装Docker-compose   [https://github.com/docker/compose/releases](https://github.com/docker/compose/releases)

```
[root@dmpdev ~]# curl -L "https://github.com/docker/compose/releases/download/1.9.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
[root@dmpdev ~]# chmod +x /usr/local/bin/docker-compose
[root@dmpdev ~]# docker-compose -version
docker-compose version 1.9.0, build 2585387

```

##### - 手工下载二进制方式(离线安装)  

安装docker-compose
自己找一台能上网的机器，进入github网站的docker-compose代码仓库的releases，找到latest版本，手工下载 具体地址：[https://github.com/docker/compose/releases/download/1.9.0/docker-compose-Darwin-x86_64](https://github.com/docker/compose/releases/download/1.9.0/docker-compose-Darwin-x86_64)  

把下载的进制文件(docker-compose-Darwin-x86_64)通过工具拷贝到有docker环境的主机的/usr/local/bin/下，修改名称为docker-compose

```
[root@dmpdev bin] mv docker-compose-Darwin-x86_64 docker-compose
[root@dmpdev bin]# chmod +x /usr/local/bin/docker-compose
[root@dmpdev bin]# docker-compose -version
docker-compose version 1.9.0, build 2585387
```

至此docker与docker-compose环境就已经安装完成。
