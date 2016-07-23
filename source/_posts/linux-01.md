---
title: linux系列(一)-linux下的mysql定时备份-nfs异地存储
date: 2016-07-10 21:27:22
updated: 2016-07-10 21:27:22
categories: [linux]
tags: [linux,mysql,docker]

---

# 前言
最近一直忙着工作方面的事，没时间来总结和更新博客，本来想法是想一周，至少两篇关于技术方面的博文，原来一起想写java方面的文章，一直没有总结好。后面的文章，都是有经过自己新手实践过。希望能和大家一起坚持下来。

# 正文
基于docker容器的mysql定时备份-nfs异地存储，正好是最近工作中实践过，所以拿来分享一下，
## 首先nfs网络文件系统搭建：
**安装：**
```bash
yum -y install nfs-utils rpcbind
```
一般系统安装后，都会安装这个必备的软件的，设置开机启动:
```bash
[root@master-zookeeper ~]#systemctl enable nfs 或者 systemctl enable nfs-server.service
[root@master-zookeeper ~]#systemctl enable rpcbind
```
### 一、环境介绍： 
- 三台台Vmware虚拟机(网络模式为nat)：
服务器：(service)192.168.159.71(centos7)
客户端：(client1)192.168.159.72(centos7)
客户端：(client2)192.168.159.73(centos7)
- 升级这三台（192.168.159.71/72/73）主机的内核到最新版本。为安装最新的docker版本
升级内核(在连网的环境下)
```bash
[root@localhost bin]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
[root@localhost bin]# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
[root@localhost bin]# yum --enablerepo=elrepo-kernel install kernel-ml
```
也可以通过此http://pkgs.org/download/kernel-devel网站手工下载安装
重要：目前内核还是默认的版本，如果在这一步完成后你就直接reboot了，重启后使用的内核版本还是默认的3.10，不会使用新的4.3，想修改启动的顺序，需要进行下一步
- 查看默认启动顺序
```bash
[root@localhost bin]# awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
```
默认启动的顺序是从0开始，但我们新内核是从头插入（目前位置在0，而3.10的是在1），所以需要选择0，如果想生效最新的内核，需要
```bash
[root@localhost bin]# grub2-set-default 0   我这里是1
```

### 二、服务器端配置（192.168.159.71）： 
- 创建共享目录：
```bash
[root@master-zookeeper ~]# mkdir –p /nfs-data/dmp/db-bak/ /nfs-data/dmp/data/ /nfs-data/dmp/frequency/
```
- NFS文件配置：
```bash
[root@master-zookeeper ~]# vim /etc/exports
/nfs-data/dmp/db-bak/ 192.168.159.*(rw,no_root_squash,sync,no_subtree_check)
/nfs-data/dmp/data/ 192.168.159.*(rw,no_root_squash,sync,no_subtree_check)
/nfs-data/dmp/frequency/ 192.168.159.*(rw,no_root_squash,sync,no_subtree_check)
:wq(保存)```

- 使配置生效：
```bash
[root@master-zookeeper ~]# exportfs -r
```
- 启动:
```bash
[root@master-zookeeper ~]# systemctl start rpcbind
[root@master-zookeeper ~]# systemctl start nfs-server.service
如果已经启动则重启：
[root@master-zookeeper ~]# systemctl restart rpcbind
[root@master-zookeeper ~]# systemctl restart nfs-server.service
```
### 三、客户端挂载（192.168.159.72、192.168.159.73）： 

- 创建需要挂载的目录：
```bash 
[root@localhost ~]# mkdir –p /nfs-data/dmp/db-bak/ /nfs-data/dmp/data/ /nfs-data/dmp/frequency/ 
```
- 测试挂载（两台客户端上执行以下命令）： 
```bash
[root@node02 ~]# showmount -e 192.168.159.71
Export list for 192.168.159.71:
/nfs-data/dmp/frequency 192.168.159.*
/nfs-data/dmp/data      192.168.159.*
/nfs-data/dmp/db-bak    192.168.159.*
```
- 挂载服务器上的三个目录： 
客户端在挂载的时候遇到的一个问题如下，可能是网络不太稳定，NFS默认是用UDP协议，换成TCP协议即可：
```bash
[root@node02 ~]#  mount -t nfs 192.168.159.71:/nfs-data/dmp/db-bak /nfs-data/dmp/db-bak -o proto=tcp -o nolock
[root@node02 ~]# mount -t nfs 192.168.159.71:/nfs-data/dmp/data /nfs-data/dmp/data -o proto=tcp -o nolock
[root@node02 ~]# mount -t nfs 192.168.159.71:/nfs-data/dmp/frequency /nfs-data/dmp/frequency -o proto=tcp -o nolock
[root@node02 ~]# mount
```
两台客户端执行完以上命令后，在159.72上看挂载情况： 
如果信息如上显示则应该是挂载成功!

### 四、NFS加到启动项，让开机自动mount

在两台客户端（192.168.159.72、192.168.159.73）上执行以下操作：
```bash
[root@node02 ~]# vim /etc/fstab
#
# /etc/fstab
# Created by anaconda on Fri Mar  4 23:01:47 2016
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=c1781921-85a8-4237-875c-8e4ebecf0e14 /boot                   xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
/dev/sda4 /nfs-data xfs nosuid,nodev,nofail,x-gvfs-show 0 0
192.168.159.71:/nfs-data/dmp/db-bak/ /nfs-data/dmp/db-bak/ nfs auto,noatime,nolock,bg,nfsvers=4,intr,tcp,actimeo=1800 0 0
192.168.159.71:/nfs-data/dmp/data/ /nfs-data/dmp/data/ nfs auto,noatime,nolock,bg,nfsvers=4,intr,tcp,actimeo=1800 0 0
192.168.159.71:/nfs-data/dmp/frequency/ /nfs-data/dmp/frequency/ nfs auto,noatime,nolock,bg,nfsvers=4,intr,tcp,actimeo=1800 0 0
:wq(保存)
```
上述的fstab中的配置：
**192.168.159.71:/nfs-data/dmp/db-bak/ /nfs-data/dmp/db-bak/ nfs auto,noatime,nolock,bg,nfsvers=4,intr,tcp,actimeo=1800 0 0**
的含义是：

命令 | 含意  
---|--- 
192.168.159.71:/nfs-data/dmp/db-bak/ | 是目标NFS服务器的IP（或域名）和NFS共享的路径
/nfs-data/dmp/db-bak/ | 是NFS客户端要mount挂载的路径（一般挂载到/mnt下面某个路径，此处只是测试，就随便挂了）
nfs | 表示挂载的文件系统类型时NFS 
auto | 自动挂载；
noatime | 不要添加access time==上次访问文件时间
nolock | 禁止文件加锁。有时候访问旧的NFS服务器需要此参数。
bg | 挂载作为后台服务去运行，如果第一次挂载失败了。默认是off的。
nfsvers=4 | 指定NFS协议的版本。
intr | 允许NFS请求被中断，如果服务器挂了或连不上
tcp | 指定NFS（不适用默认的UDP而改用）TCP
actimeo=1800 | acregmin==acregmax==acdirmin====acdirmax，都设置为1800s=30分钟，即文件缓存时间为30分钟
0 | 不需要NFS的CacheFS
0 | 不需要NFS的CacheFS

本次测试通过以上测试，重启后，可以自动挂载

## 对Mysql进行定时备份
### 利用linux的定时器crontab对mysql进行定时备份
- crontab -e 是新建或者修改 -l是查看定时器
   通过NFS创建一个db-bak文件，做为异地备份传输（可存储在71、72、73这三台机器的/nfs-data/dmp/db-bak）。具体怎么操作，参考上面nfs网络文件系统搭建章节
- 在db-bak文件下创建一下备份的shell脚本，内容如下
```bash
[root@node2 frequency]# cat backupDmp.sh 
#!/bin/bash
#按时间生成变量strName作为文件名
strName=`date +%Y-%m-%d-%H-%M-%S`
#在运行在docker环境的mysql中执行备份命令 
docker exec dmpmysql mysqldump -u root -p123456 device>/nfs-data/dmp/db-bak/$strName-device.sql
[root@node2 frequency]# chmod +x backupDmp.sh  (修改执行权限)
```
- 在执行以上脚本前，需要mysql容器把/nfs-data/dmp/db-bak/挂到到容器里。所以如果是已经启动的容器，请重新挂载该目录。(参考docker容器挂载目录)
- 加入到linux的crontab任务调度中
```bash
[root@node2 frequency]# crontab -e
#每隔两天，在2.30分钟执行一次
30 2 */2 * * /nfs-data/dmp/db-bak/backupDmp.sh
#每三分钟执行一次
#*/3 * * * * /nfs-data/dmp/db-bak/backupDmp.sh
```
执行以上步骤后，数据备份会每隔两自动备份，并通过NFS异地备份到别的机器上


## Mysql定时删除上月备份

- 在db-bak文件下创建一下备份的shell脚本，内容如下:
```bash
[root@node2 frequency]# vim backupDeleteDmp-Input.sh 
#!/bin/bash
#获取当前时间的上一个月的月份数据为daleteTime作为文件名
deleteTime=`date -d "last month" +%Y-%m`
#执行删除上月备份的数据
rm -rf $deleteTime*.sql
[root@node2 frequency]# chmod +x backupDeleteDmp-Input.sh  (修改执行权限)
```

- 加入到linux的crontab任务调度中
```bash
[root@node2 frequency]# crontab -e
#每隔月16号，在2.30分钟执行删除上月备份数据
30 2 16 * * /nfs-data/dmp/db-bak/backupDeleteDmp-Input.sh
```
执行以上步骤后，数据备份会每个月16号会自动删除上月的备份，并通过NFS异地删除别的机器上的备份


---

