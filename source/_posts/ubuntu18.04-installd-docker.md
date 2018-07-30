---
title: ubuntu18.04 server lts版本安装docker及相关配置
toc: true   // 在文章侧边增加文章目录
date: 27/05/2018 5:26:15 PM 
updated: 27/05/2018 5:26:20 PM 
categories: [ubuntu]
tags: [linux,ubuntu,docker]

---

最近因项目需要，把服务器的相关操作系统转到ubuntu server上来，所以正好记录一下安装docker的常用配置及注意事项。由于之前项目一直是用centos来做服务器的操作系统，很少关注ubuntu的服务器版本。在使用docker及相关容器平台时发现centos会带来一些不可预知的错误，也很难有相关的修复答案。在查询相关资料后，国外大多在使用ubuntu来做容器平台的操作系统，而且社区相关的问题也比较活跃，所以近期我们也正在考虑转入ubuntu来做容器的操作系统。借此以记录ubuntu相关的基本操作与docker安装。

## ubuntu 18.04 server版本的网络及主机名的配置

#### 网络配置
ubuntu18.04的网络配置已经用netplay接管了，所以修改网络需要修改/etc/netplan/下的50-cloud-init.yaml文件，如：

```shell
admin@node01:~$ sudo vim /etc/netplan/50-cloud-init.yaml
# This file is generated from information provided by# the datasource.  Changes to it will not persist across an instance.# To disable cloud-init's network configuration capabilities, write a file# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:# network: {config: disabled}network:    ethernets:        eno1:            addresses:            - 192.168.18.21/24            gateway4: 192.168.18.1            nameservers:                addresses:                - 114.114.114.114                search: []            optional: true    version: 2~                                                                                                                                                                                  :wq
admin@node01:~$ sudo netplan apply
admin@node01:~$ ifconfig
```
![](/images/linux/ubuntu-01/15274068318377.jpg)

#### 主机名配置
如果您正在使用Ubuntu 18.04 LTS服务器，如何重命名或更改服务器主机名。主机名是服务器的唯一名称。这就是服务器在网络上被识别的方式。 
 
* 更改主机名文件/etc/hostname中的服务器名称   
首先编辑/etc/cloud/cloud.cfg并将参数“ preserve_hostname ”从“ false ”设置为“ true

```
admin@node01:~$ sudo vim /etc/cloud/cloud.cfg
```
![](/images/linux/ubuntu-01/15274072047285.jpg)
其次是编辑/etc/hostname文件，并编辑文件中的名字为你所需要的主机名称  
```
admin@node01:~$ sudo vim /etc/hostname
```
![](/images/linux/ubuntu-01/15274073917659.jpg)

* 更改主机文件/etc/hosts中的服务器名称  
更改主机文件/etc/hosts中的服务器名称,在127.0.0.1后面增加主机名称。
```
admin@node01:~$ sudo vim /etc/hosts
```
![](/images/linux/ubuntu-01/15274075104177.jpg)

* 重新启动服务器
最后，重新启动服务器以应用新名称...如果您未重新启动，则新名称将无法正确应用。运行以下命令重新启动服务器。

```
admin@node01:~$ sudu reboot
```

## ubuntu 18.04 server版本的docker安装及配置

#### docker安装

* 移除旧版本docker 

```
admin@node21:~$ sudo apt-get remove docker docker-engine docker.io 
```

* 安装软件包来允许apt通过HTTPS使用存储库 

```
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common 
```

* 添加Docker的官方GPG密钥 

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - 
```

* 添加docker的下载源，
因为官方还没有ubuntu18的下载源，所以先用ubuntu17（zesty）的 
如果操作系统是ubuntu14.04把zesty换成trusty，如果是ubuntu16.04换成xenial 

```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu zesty stable" 
```

* 安装docker-ce 

```
sudo apt-get update 
sudo apt-get install docker-ce
```

* 授于当前用户不需要使用sudo来执行docker命令的权限 
创建docker组 

```
sudo groupadd docker 
```
将当前用户加入docker组 

```
sudo gpasswd -a ${USER} docker 
#或者 
sudo usermod -aG docker $USER
```
重启docker 并设置开机启动docker

```
sudo systemctl restart docker
sudo systemctl enable docker
```

#### docker 简单配置
这里我只对docker的存储驱动和国内镜像源加速器（阿里个人加载器）的配置，修改或者创建/etc/docker/daemon.json

```
sudo /etc/docker/daemon.json
{
  "storage-driver": "devicemapper",
  "registry-mirrors": ["https://1i186hp0.mirror.aliyuncs.com"]
}
```

#### 问题：
正式环境配置存储时报错：
配置文件：

```
{
  "storage-driver": "devicemapper",
  "storage-opts": [
    "dm.directlvm_device=/dev/sdb1",
    "dm.thinp_percent=95",
    "dm.thinp_metapercent=1",
    "dm.thinp_autoextend_threshold=80",
    "dm.thinp_autoextend_percent=20",
    "dm.directlvm_device_force=true"
  ],
  "bip":"172.20.10.1/24",
  "insecure-registries": ["tcr:5000","hub.topsec.com.cn"],
  "registry-mirrors": ["https://1i186hp0.mirror.aliyuncs.com"]
}
```


报错信息：

```
#问题1：
(/dev/mapper/control) leaked on pvdisplay invocation. Parent PID 2940: /usr/bin/dockerd\n  Failed to find physical volume \"/dev/sdb\".\n" error="exit status 5"
Jul 25 01:06:20 node207 dockerd[2940]: Error starting daemon: error initializing graphdriver: error looking up command `thin_check` while setting up direct lvm: exec: "thin_check": executable file not found in $PATH
```
解决办法：

```
Install the following packages: 安装以下包RHEL / CentOS: device-mapper-persistent-data, lvm2, and all dependencies
$sudo yum -y install Ubuntu / Debian: thin-provisioning-tools, lvm2, and all dependencies
$sudo apt-get -y install thin-provisioning-tools lvm2
``` 
-------------------------------------------------------------------------------
```
#问题2：
/usr/bin/dockerd\nFile descriptor 10 (/dev/mapper/control) leaked on pvdisplay invocation. Parent PID 3951: /usr/bin/dockerd\n  Failed to find physical volume \"/dev/sdb\".\n" error="exit status 5"
Jul 25 01:27:33 node202 dockerd[3951]: Error starting daemon: error initializing graphdriver: File descriptor 3 (socket:[78604]) leaked on pvcreate invocation. Parent PID 3951: /usr/bin/dockerd
```
解决办法：

```
$ sudo pvcreate /dev/sdb1Physical volume "/dev/sdb1" successfully created.
``` 
-------------------------------------------------------------------------------


```
#问题3：
rpc
Jul 25 01:58:35 node202 dockerd[2468]: time="2018-07-25T01:58:35.446718795Z" level=info msg="pickfirstBalancer: HandleSubConnStateChange: 0xc42021a230, CONNECTING" module=grpc
Jul 25 01:58:35 node202 dockerd[2468]: time="2018-07-25T01:58:35.447079391Z" level=info msg="pickfirstBalancer: HandleSubConnStateChange: 0xc42021a230, READY" module=grpc
Jul 25 01:58:35 node202 dockerd[2468]: Error starting daemon: error initializing graphdriver: /dev/sdb is not available for use with devicemapper
Jul 25 01:58:35 node202 systemd[1]: docker.service: Main process exited, code=exited, status=1/FAILURE
Jul 25 01:58:35 node202 systemd[1]: docker.service: Failed with result 'exit-code'.
```
解决办法：

```
$ sudo fdisk /dev/sdb   输入m ->n->回车->回车
``` 
-------------------------------------------------------------------------------

