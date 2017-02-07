---
title: CENTOS7上配置Docker存储驱动devicemapper
toc: true
date: 2017-01-14 22:10:38 Saturday
updated: 2017-01-14 22:10:45 Saturday
categories: [docker]
tags: [linux,docker]

---

devicemapper驱动将每一个Docker镜像和容器存储在它自身的具有精简置备(thin-provisioned)、写时拷贝(copy-on-write)和快照功能(snapshotting)的虚拟设备上。由于Device Mapper技术是在块(block)层面而非文件层面，所以Docker Engine的devicemapper存储驱动使用的是块设备来存储数据而非文件系统。  

# devicemapper的模式  
devicemapper是RHEL下Docker Engine的默认存储驱动，它有两种配置模式:loop-lvm和direct-lvm。

## loop-lvm是默认的模式  
它使用OS层面离散的文件来构建精简池(thin pool)。该模式主要是设计出来让Docker能够简单的被”开箱即用(out-of-the-box)”而无需额外的配置。但如果是在生产环境的部署Docker，官方明文不推荐使用该模式。我们使用docker info命令可以看到以下警告:

```bash
[root@tsccloud01 ~]# docker info
Containers: 22
 Running: 5
 Paused: 0
 Stopped: 17
Images: 6
Server Version: 1.12.3
Storage Driver: devicemapper
 Pool Name: docker-253:0-202359283-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 Data file: /dev/loop0
 Metadata file: /dev/loop1
 Data Space Used: 1.528 GB
 Data Space Total: 107.4 GB
 Data Space Available: 50.38 GB
 Metadata Space Used: 4.149 MB
 Metadata Space Total: 2.147 GB
 Metadata Space Available: 2.143 GB
 Thin Pool Minimum Free Space: 10.74 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: false
 Deferred Deletion Enabled: false
 Deferred Deleted Device Count: 0
 Data loop file: /var/lib/docker/devicemapper/devicemapper/data
 WARNING: Usage of loopback devices is strongly discouraged for production use. Use `--storage-opt dm.thinpooldev` to specify a custom block storage device.
 Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
 Library Version: 1.02.107-RHEL7 (2016-06-09)
Logging Driver: json-file
Cgroup Driver: cgroupfs

```

## 配置direct-lvm模式

direct-lvm是Docker推荐的生产环境的推荐模式，他使用块设备来构建精简池来存放镜像和容器的数据。

在操作之前，如果是之间已经运行了docker，请先备份相关的docker容器与镜像，官方建议是如果 是生产环境，在安装完docker后就直接配置direct-lvm模式，本文使用规划分配的一个逻辑卷sda5来配置，推荐使用外部共享存储的设备但不局限于此种方式，可根据自己的环境决定。


查看宿主机上的物理盘的分区情况
```bash
[root@localhost ~]# lsblk
NAME                           MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                              8:0    0   2.7T  0 disk 
├─sda1                           8:1    0     1M  0 part 
├─sda2                           8:2    0     3G  0 part /boot
├─sda3                           8:3    0   948G  0 part 
│ ├─centos-root                253:0    0   300G  0 lvm  /
│ ├─centos-swap                253:1    0    48G  0 lvm  [SWAP]
│ └─centos-home                253:2    0   600G  0 lvm  /home
├─sda4                           8:4    0 838.1G  0 part /nfs-data
└─sda5                           8:5    0  1004G  0 part 
sr0                             11:0    1  1024M  0 rom  
loop0                            7:0    0   100G  0 loop 
└─docker-253:0-1074356357-pool 253:3    0   100G  0 dm   
loop1                            7:1    0     2G  0 loop 
└─docker-253:0-1074356357-pool 253:3    0   100G  0 dm   
```
### 创建物理卷/dev/sda5
```bash
[root@localhost ~]# pvcreate /dev/sda5
WARNING: ext4 signature detected on /dev/sda5 at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/sda5.
  Physical volume "/dev/sda5" successfully created
```
### 创建docker LVM卷组
```bash
[root@localhost ~]# vgcreate docker /dev/sda5
  Volume group "docker" successfully created
```
### 创建thinpool
数据LV大小为VG的95%,元数据LV大小为VG的1%,剩余的空间用来自动扩展
- 创建pool

```bash
[root@localhost ~]# lvcreate --wipesignatures y -n thinpool docker -l 95%VG
  Logical volume "thinpool" created.
[root@localhost ~]# lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG
  Logical volume "thinpoolmeta" created.
```

- 将pool转换为thinpool

```bash
[root@localhost ~]# lvconvert -y --zero n -c 512K --thinpool docker/thinpool --poolmetadata docker/thinpoolmeta
  WARNING: Converting logical volume docker/thinpool and docker/thinpoolmeta to pool's data and metadata volumes.
  THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
  Converted docker/thinpool to thin pool.
```

### 配置thinpool

- 配置池的自动扩展

```bash
[root@localhost ~]# vim /etc/lvm/profile/docker-thinpool.profile
activation {
    thin_pool_autoextend_threshold=80
    thin_pool_autoextend_percent=20
}
```

- 应用配置变更

```bash
[root@localhost ~]# lvchange --metadataprofile docker-thinpool docker/thinpool
  Logical volume "thinpool" changed.
```

- 状态监控检查

```bash
[root@localhost ~]# lvs -o+seg_monitor
  LV       VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Monitor  
  home     centos -wi-ao---- 600.00g                                                              
  root     centos -wi-ao---- 300.00g                                                              
  swap     centos -wi-ao----  48.00g                                                              
  thinpool docker twi-a-t--- 953.73g             0.00   0.01                             monitored
```

### 配置Docker

- 清除graphdriver

之前已提醒数据备份，因为在这里清除graphdriver会将image,container和volume所有数据都删除。如果不删除，则会遇到以下的错误导致docker服务起不来

```bash
[root@localhost ~]# mkdir /var/lib/docker.bk
[root@localhost ~]# mv /var/lib/docker/* /var/lib/docker.bk
mv: 无法将"/var/lib/docker/devicemapper" 移动至"/var/lib/docker.bk/devicemapper": 设备或资源忙
[root@localhost ~]# systemctl stop docker
[root@localhost ~]# mv /var/lib/docker/* /var/lib/docker.bk
[root@localhost ~]# rm -rf /var/lib/docker/*
```

- 修改服务配置文件

```bash
[root@localhost ~]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: inactive (dead) since 一 2017-01-09 21:50:21 CST; 32s ago
     Docs: https://docs.docker.com
  Process: 9401 ExecStart=/usr/bin/dockerd (code=exited, status=0/SUCCESS)
 Main PID: 9401 (code=exited, status=0/SUCCESS)

1月 09 21:47:59 localhost dockerd[9401]: time="2017-01-09T21:47:59.307356931+08:00" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be used t...rred IP address"
1月 09 21:47:59 localhost dockerd[9401]: time="2017-01-09T21:47:59.387442387+08:00" level=info msg="Loading containers: done."
1月 09 21:47:59 localhost dockerd[9401]: time="2017-01-09T21:47:59.387855556+08:00" level=info msg="Daemon has completed initialization"
1月 09 21:47:59 localhost dockerd[9401]: time="2017-01-09T21:47:59.387889962+08:00" level=info msg="Docker daemon" commit=7392c3b graphdriver=devicemapper version=1.12.5
1月 09 21:47:59 bogon dockerd[9401]: time="2017-01-09T21:47:59.398837780+08:00" level=info msg="API listen on /var/run/docker.sock"
1月 09 21:47:59 bogon systemd[1]: Started Docker Application Container Engine.
1月 09 21:50:20 bogon systemd[1]: Stopping Docker Application Container Engine...
1月 09 21:50:20 bogon dockerd[9401]: time="2017-01-09T21:50:20.470035651+08:00" level=info msg="Processing signal 'terminated'"
1月 09 21:50:20 bogon dockerd[9401]: time="2017-01-09T21:50:20.50479833+08:00" level=info msg="stopping containerd after receiving terminated"
1月 09 21:50:21 bogon systemd[1]: Stopped Docker Application Container Engine.
Hint: Some lines were ellipsized, use -l to show in full.
[root@localhost ~]# vim /usr/lib/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd --insecure-registry=tcr:5000 --storage-driver=devicemapper --storage-opt=dm.thinpooldev=/dev/mapper/docker-thinpool --storage-opt=dm.use_deferred_removal=true --storage-opt=d
m.use_deferred_deletion=trueExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
```

- 重启docker

```bash
[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl restart docker
[root@localhost ~]# lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                         8:0    0   2.7T  0 disk 
├─sda1                      8:1    0     1M  0 part 
├─sda2                      8:2    0     3G  0 part /boot
├─sda3                      8:3    0   948G  0 part 
│ ├─centos-root           253:0    0   300G  0 lvm  /
│ ├─centos-swap           253:1    0    48G  0 lvm  [SWAP]
│ └─centos-home           253:2    0   600G  0 lvm  /home
├─sda4                      8:4    0 838.1G  0 part /nfs-data
└─sda5                      8:5    0  1004G  0 part 
  ├─docker-thinpool_tmeta 253:4    0    10G  0 lvm  
  │ └─docker-thinpool     253:6    0 953.7G  0 lvm  
  └─docker-thinpool_tdata 253:5    0 953.7G  0 lvm  
    └─docker-thinpool     253:6    0 953.7G  0 lvm  
sr0                        11:0    1  1024M  0 rom  
[root@localhost ~]# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.12.5
Storage Driver: devicemapper
 Pool Name: docker-thinpool
 Pool Blocksize: 524.3 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 Data file: 
 Metadata file: 
 Data Space Used: 19.92 MB
 Data Space Total: 1.024 TB
 Data Space Available: 1.024 TB
 Metadata Space Used: 1.188 MB
 Metadata Space Total: 10.78 GB
 Metadata Space Available: 10.78 GB
 Thin Pool Minimum Free Space: 102.4 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: true
 Deferred Deletion Enabled: true
 Deferred Deleted Device Count: 0
 Library Version: 1.02.107-RHEL7 (2015-10-14)
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: null host bridge overlay
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Security Options: seccomp
Kernel Version: 3.10.0-327.el7.x86_64
Operating System: CentOS Linux 7 (Core)
OSType: linux
Architecture: x86_64
CPUs: 4
Total Memory: 7.591 GiB
Name: bogon
ID: ISYY:44CF:DXHU:PAFM:JULI:TMZ7:326S:A7EH:MZAB:3RIL:UIVK:FXJG
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disabled
Experimental: true
Insecure Registries:
 127.0.0.0/8

```
- 查看direct-lvm是本配置成功

用docker命令pull下载一个镜像，你会发现thinpool的Data%会增加
```bash
[root@docker-master ~]# lvs
  LV       VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home     centos -wi-ao----   1.00t                                                    
  root     centos -wi-ao---- 300.00g                                                    
  swap     centos -wi-ao----  48.00g                                                    
  thinpool docker twi-aot---   1.32t             0.23   0.02    
```

# 参考
linux命令创建PV/VG/LV：[http://man.linuxde.net/pvcreate](http://man.linuxde.net/pvcreate "linux命令创建PV/VG/LV")
docker devicemapper存储驱动官方说明: [(https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/ "docker devicemapper存储驱动官方说明")