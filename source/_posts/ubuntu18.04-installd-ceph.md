---
title: 在ubuntu18.04中安装基于docker容器的ceph文件共享服务
toc: true   // 在文章侧边增加文章目录
date: 29/06/2018 5:26:15 PM 
updated: 29/06/2018 5:26:20 PM 
categories: [ceph]
tags: [linux,ubuntu,docker,ceph]

---

最近一直在学习kubernetes容器编排集群技术，为了实现pod数据的持久化。在调研的过程中，ceph在最近几年发展火热，社区活跃度也是相当的高，也有很多企业对ceph的落地提供相当多的经验。在选型方面，GlusterFS、Ceph都在考虑的范围之内，但是由于GlusterFS只提供对象存储和文件系统存储，而Ceph则提供对象存储、块存储以及文件系统存储。怀着对新事物的向往，果断选择Ceph来实现Ceph块存储对接kubernetes来实现pod的数据持久化。本文通过docker安装ceph的服务。

###环境准备

| 主机IP | 主机名 | 安装相关服务说明 |
| :-: | :-: | :-: |
| 192.168.19.13 | Node13 | Docker、chrony、ceph(mon\osd(disk)\mds) |
| 192.168.19.14 | Node14 | Docker、chrony、ceph(mon\osd(disk)\mds) |
| 192.168.19.15 | Node15 | Docker、chrony、ceph(mon\osd(disk)\mds) |
| 192.168.19.16 | Node16 | Docker、chrony、ceph(mon\osd(disk)\mds) |
| 192.168.19.17 | Node17 | Docker、chrony、ceph(mon\osd(disk)\mds) |

### 前置环境安装与配置
#### 网络配置（所有主机）

```
admin@nodeXX:~$ sudo vim /etc/netplan/50-cloud-init.yaml
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        eno1:
            addresses:
            - 192.168.19.XX/24
            gateway4: 192.168.19.1
            nameservers:
                addresses:
                - 114.114.114.114
                search: []
            optional: true
    version: 2
~                                                                                                                                                                                  
:wq
admin@nodeXX:~$ sudo netplan apply
admin@nodeXX:~$ ifconfig
```
其中以上内容中的`XX`代表主机的IP地址最后两位13-17，参考环境准备表
#### 主机名与hosts表配置（所有主机）

```
admin@nodeXX:~$ sudo vim /etc/cloud/cloud.cfg
#编辑/etc/cloud/cloud.cfg并将参数“ preserve_hostname ”从“ false ”设置为“ true
admin@nodeXX:~$ sudo vim /etc/hostname
nodeXX
:wq #保存
admin@nodeXX:~$ sudo vi /etc/hosts  
192.168.19.13 node13
192.168.19.14 node14
192.168.19.15 node15
192.168.19.16 node16
192.168.19.17 node17
:wq
admin@nodeXX:~$ sudu reboot
```
其中以上内容中的`XX`代表主机的IP地址最后两位13-17，参考环境准备表

#### docker安装（所有主机）
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
* docker 简单配置
对docker的存储驱动和国内镜像源加速器（阿里个人加载器）的配置，修改或者创建/etc/docker/daemon.json

    ```
sudo /etc/docker/daemon.json
{
  "storage-driver": "devicemapper",
  "registry-mirrors": ["https://1i186hp0.mirror.aliyuncs.com"] #此为本人阿里云个人加速器，各位可根据自己的阿里云容器加速器来添加
}
```

#### 修改时区及安装时钟同步服务（所有主机）
 
* 修改时区

    ```
admin@nodeXX:~$ sudo tzselect
```
然后选择亚洲Asia，继续选择中国China，最后选择北京Beijing。  
然后创建时区软链  
    
    ```
admin@nodeXX:~$ sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
admin@nodeXX:~$ date #查看时区是否正确  
```
* 安装时间同步服务

    ```
 #chrony服务端的安装
admin@nodeXX:~$ sudo apt -y update
admin@nodeXX:~$ sudo apt -y install chrony
......
admin@nodeXX:~$ sudo vim /etc/chrony/chrony.conf
......
 #pool ntp.ubuntu.com        iburst maxsources 4
 #pool 0.ubuntu.pool.ntp.org iburst maxsources 1
 #pool 1.ubuntu.pool.ntp.org iburst maxsources 1
 #pool 2.ubuntu.pool.ntp.org iburst maxsources 2
 #修改为国内网络时间服务
server 1.cn.pool.ntp.org iburst maxsources 4
server 1.asia.pool.ntp.org iburst maxsources 1
server 0.asia.pool.ntp.org iburst maxsources 2
 #允许同步的网段
allow 192.168.19.0/24
......
admin@nodeXX:~$ systemctl enable chronyd
admin@nodeXX:~$ systemctl start chronyd
-------------------------------------------------
 #chrony客户端安装
admin@nodeXX:~$ sudo apt -y update
admin@nodeXX:~$ sudo apt -y install chrony
......
admin@nodeXX:~$ sudo vim /etc/chrony/chrony.conf

 #修改为服务端时间服务(nodexx为chrony服务端主机名或者IP地址)
server nodexx iburst 
:wq
admin@nodeXX:~$ systemctl enable chronyd
admin@nodeXX:~$ systemctl start chronyd
```

### ceph服务安装
修改 ssh 配置(中途需要用 scp 复制文件) 

```
admin@nodeXX:~$ sudo passwd root
xxxxxxx(root密码)
admin@nodeXX:~$ sudo vi /etc/ssh/sshd_config
#PermitRootLogin prohibit-password
PermitRootLogin yes
admin@nodeXX:~$ sudo service ssh restart
#获取ceph docker镜像
admin@nodeXX:~$ sudo docker pull ceph/daemon:latest-jewel

```
#### 部署 mon(所有主机)

node13主机上
```
admin@node13:~$ sudo mkdir /etc/ceph
admin@node13:~$ sudo mkdir /var/lib/ceph
admin@node13:~$ sudo docker run -d --net=host \
--restart=on-failure \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-e MON_IP=192.168.19.13 \
-e CEPH_PUBLIC_NETWORK=192.168.19.0/24 \
ceph/daemon:latest-jewel mon
 #拷当前主机部署的配置文件到其它4台主机上,其中XX代表14-17这四台主机
admin@node13:~$ sudo scp -r /etc/ceph root@192.168.19.XX:/etc
admin@node13:~$ sudo scp -r /var/lib/ceph/bootstrap-* root@192.168.19.XX:/var/lib/ceph
admin@node13:~$ 
```
node14主机上：

```
sudo docker run -d --net=host \
--restart=on-failure \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-e MON_IP=192.168.19.14 \
-e CEPH_PUBLIC_NETWORK=192.168.19.0/24 \
ceph/daemon:latest-jewel mon
```
node15主机上：

```
sudo docker run -d --net=host \
--restart=on-failure \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-e MON_IP=192.168.19.15 \
-e CEPH_PUBLIC_NETWORK=192.168.19.0/24 \
ceph/daemon:latest-jewel mon
```
node16主机上：

```
sudo docker run -d --net=host \
--restart=on-failure \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-e MON_IP=192.168.19.16 \
-e CEPH_PUBLIC_NETWORK=192.168.19.0/24 \
ceph/daemon:latest-jewel mon
```
node17主机上：

```
sudo docker run -d --net=host \
--restart=on-failure \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-e MON_IP=192.168.19.17 \
-e CEPH_PUBLIC_NETWORK=192.168.19.0/24 \
ceph/daemon:latest-jewel mon
```

#### 部署 osd (disk) (所有主机)
node13-node17主机上执行：

```
sudo docker run -d --net=host \
--restart=on-failure \
--privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
-v /dev/:/dev/ \
-e OSD_DEVICE=/dev/sdb \
-e OSD_TYPE=disk \
ceph/daemon:latest-jewel osd
```
 **备注:-e OSD_FORCE_ZAP=1 \ 初次启动时，需要强制擦除硬盘，第二次启动时不能添加此参数
 初次启动osd的容器运行下面的命令，一量创建后就不需要OSD_FORCE_ZAP=1这个参数，所以重启时运行上面的命令**
 
```
sudo docker run -d --net=host \ --restart=on-failure \ --privileged=true \
--pid=host \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \ -v /dev/:/dev/ \
-e OSD_DEVICE=/dev/sdb \
-e OSD_TYPE=disk \
-e OSD_FORCE_ZAP=1 \ ceph/daemon:latest-jewel osd
```

#### 部署 mds (所有主机)
node13-node17主机上执行：

```
sudo docker run -d --net=host \
--restart=on-failure \
-v /var/lib/ceph/:/var/lib/ceph/ \
-v /etc/ceph:/etc/ceph \
-e CEPHFS_CREATE=1 \
ceph/daemon:latest-jewel mds
```

#### ceph配置
* 修改存储池 Pool 的归置组 PG 数和增加元数据存储池 cephfs_metadata 的副本数 视情况而定

    ```
sudo docker exec f097a825abe9 ceph osd pool set cephfs_metadata size 5
sudo docker exec f097a825abe9 ceph osd pool set cephfs_data pg_num 128
sudo docker exec f097a825abe9 ceph osd pool set cephfs_data pgp_num 128
sudo docker exec f097a825abe9 ceph osd pool set cephfs_metadata pg_num 128
sudo docker exec f097a825abe9 ceph osd pool set cephfs_metadata pgp_num 128
```
其中`f097a825abe9`为任意一台上的mon.  
查看ceph服务的状态：

    ```
admin@node13:~$ docker ps
CONTAINER ID        IMAGE                      COMMAND                CREATED             STATUS              PORTS               NAMES
f99101bbf805        ceph/daemon:latest-jewel   "/entrypoint.sh mds"   2 weeks ago         Up 2 weeks                              eloquent_noether
66f307278449        ceph/daemon:latest-jewel   "/entrypoint.sh osd"   2 weeks ago         Up 2 weeks                              naughty_leakey
a36b027a3e64        ceph/daemon:latest-jewel   "/entrypoint.sh mon"   2 weeks ago         Up 2 weeks                              upbeat_goldberg
admin@node13:~$ docker exec -it a36b027a3e64 ceph status
    cluster 56f8a922-3a91-4409-96a7-de74dfa9554d
     health HEALTH_OK
     monmap e5: 5 mons at {node13=192.168.19.13:6789/0,node14=192.168.19.14:6789/0,node15=192.168.19.15:6789/0,node16=192.168.19.16:6789/0,node17=192.168.19.17:6789/0}
            election epoch 46, quorum 0,1,2,3,4 node13,node14,node15,node16,node17
      fsmap e33: 1/1/1 up {0=node16=up:active}, 4 up:standby
     osdmap e88: 5 osds: 5 up, 5 in
            flags sortbitwise,require_jewel_osds
      pgmap v19122: 320 pgs, 3 pools, 164 kB data, 21 objects
            206 MB used, 37244 GB / 37244 GB avail
                 320 active+clean
```
* 测试ceph服务使用
在node13挂载ceph目录

    ```
admin@node13:~$ sudo mkdir /mnt/cephfs
admin@node13:~$ sudo mount -t ceph 192.168.19.13:6789:/ /mnt/cephfs -o
name=admin,secret=AQAgUSNb5YRtHxAAnnCU4X16iyTpkvwzG4Vaiw==
admin@node13:~$ cd /etc/mnt/cephfs
admin@node13:~$ touch 1.txt
```
在node14上同样挂载ceph目录

    ```
admin@node14:~$ sudo mkdir /mnt/cephfs
admin@node14:~$ sudo mount -t ceph 192.168.19.13:6789:/ /mnt/cephfs -o
name=admin,secret=AQAgUSNb5YRtHxAAnnCU4X16iyTpkvwzG4Vaiw==
admin@node14:~$ cd /etc/mnt/cephfs
admin@node14:~$ touch 2.txt
```
然后在node13和node14上的/mnt/cephfs下是否1.txt和2.txt两个文件，如果有恭喜你ceph服务安装成功了。

