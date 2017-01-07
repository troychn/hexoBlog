---
title: docker系列(三)Docker的图形化管理工具Shipyard
toc: true
date: 8/7/2016 8:33:03 PM  
updated: 8/7/2016 8:33:06 PM 
categories: [docker]
tags: [linux,docker]

---

### 前言
启动两个虚机，都部署Docker Engine，然后再其中一台上安装shipyard ，管理两个Docker Engine，其中一个Engine 贴标签为dev ,一个为online，表明开发环境或线上环境
采用shipyard发布两个MySQL 实例，分别名字为MySQL-dev与MySQL-Online，在不同的Docker上，截图说明操作过程

### 正文
#### 环境准备
1.两台Vmware虚拟机(网络模式为nat)：
(dev)192.168.253.129(centos7)
(online)192.168.253.134(centos7)
2.升级这两台（192.168.253.134、192.168.253.129）主机的内核到最新版本。为安装最新的docker版本
升级内核(在连网的环境下)
[root@localhost bin]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
[root@localhost bin]# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
[root@localhost bin]# yum --enablerepo=elrepo-kernel install kernel-ml
也可以通过此http://pkgs.org/download/kernel-devel网站手工下载安装
重要：目前内核还是默认的版本，如果在这一步完成后你就直接reboot了，重启后使用的内核版本还是默认的3.10，不会使用新的4.3，想修改启动的顺序，需要进行下一步
查看默认启动顺序
[root@localhost bin]# awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
默认启动的顺序是从0开始，但我们新内核是从头插入（目前位置在0，而3.10的是在1），所以需要选择0，如果想生效最新的内核，需要
[root@localhost bin]# grub2-set-default 0   我这里是1
![nh](/images/docker/3-1.png)
3.修改两台主机的主机名（这一步重要，如果不修改，后续在创建容器互联时两主机间的容器不能通信）：
在192.168.253.129修改主机名并重启
[root@localhost bin]# hostnamectl set-hostname dev
[root@localhost bin]# reboot 
在192.168.253.134修改主机名并重启
[root@localhost bin]# hostnamectl set-hostname online
[root@localhost bin]# reboot 

#### 安装shipyard，通过官方的脚本安装
1.首次部署脚本

[root@dev ~]# curl -sSL https://dockerclub.net/deploy | bash -s
ACTION: 可以使用的指令 (deploy, upgrade, node, remove)
DISCOVERY: 集群系统采用Swarm进行采集和管理(在节点管理中可以使用‘node’)
IMAGE: 镜像，默认使用shipyard的镜像
PREFIX: 容器名字的前缀
SHIPYARD_ARGS: 容器的常用参数
TLS_CERT_PATH: TLS证书路径
PORT: 主程序监听端口 (默认端口: 8080)
PROXY_PORT: 代理端口 (默认: 2375) 

运行这条命令会下载依赖的镜像，国内环境估计无法下载，我用的是阿里云的加载器。至于怎么添加，登录阿里云上去看就知道了。提供下地址：http://console.d.aliyun.com/index2.html/?spm=0.0.0.0.Xx1dX0#/docker/booster
上面命令效果如下：
![nh](/images/docker/3-2.png)
我这肯定是已经下载过了依赖的镜像，所以没有提示下载镜像操作。

2.增加一个部署节点
shipyard节点部署脚本将自动的安装key/value存储系统（etcd系统）。增加一个节点到swarm集群，你可以使用以下的节点部署脚本
[root@online ~]  curl -sSL https://dockerclub.net/deploy | ACTION=node  DISCOVERY=etcd://192.168.253.129:4001 bash -s
注意：192.168.253.129这个ip地址你需要修改为你的首次初始化shipyard系统的主机地址
![nh](/images/docker/3-3.png)
3.删除shipyard系统（运行上面两步，就可以对shipyard使用）
[root@online ~]$  curl -sSL https://dockerclub.net/deploy | ACTION=remove bash -s

#### shipyard使用
在浏览器中验证：
输入：http://localhost:8080/#/login（我这是做了vmware的虚拟IP映射，所以是localhost，admin/shipyard）
节点信息：
![nh](/images/docker/3-4.png)
创建容器：
![nh](/images/docker/3-5.png)
![nh](/images/docker/3-6.png)
两个容器的运行结果，用官方的安装是一个swarm集群环境，好像一不能指定具体在那一个节点上运行。
![nh](/images/docker/3-7.png)


### 参考：

