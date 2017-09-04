---
title: ranchervm上运行kvm的win7-docker容器
toc: true   // 在文章侧边增加文章目录
date: 9/3/2017 5:26:15 PM 
updated: 9/3/2017 5:26:20 PM 
categories: [rancher]
tags: [rancher,kvm,win7,docker,linux,Dockerfile]

---

最近一直忙于老项目的spring boot和spring cloud改造，8月份没有任何自己的学习笔记，实在是感到惭愧。说好的每个月写几篇自己的工作与学习中的技术笔记与心得，又没完成。9月继续努力！
本文主要是在docker的环境下运行一个kvm版的win7虚拟机，至于为什么要在docker环境中运行win7虚拟机，这就得看各自的业务需求了，我这是因为工作中需要，所以整理成这篇文章。感谢大家的支持与关注。

# centos7下制作kvm的win7虚拟机  
## centos7上安装KVM虚拟化

- 检测cpu是否支持硬件虚拟化

```
[root@bogon ~]# grep -o -E '(vmx|svm)' /proc/cpuinfo
vmx
vmx
```

输出vmx或svm代表支持虚拟化  否则如果什么都没输出代表cpu不支持虚拟化  

- 安装KVM以及相关组件  

安装 kvm 基础包    

```bash
[root@bogon ~]# sudo yum install -y kvm 
```  
安装kvm 管理工具

```bash
[root@bogon ~]# sudo yum install -y qemu-kvm qemu-img virt-manager libvirt libvirt-python libvirt-client virt-install virt-viewer bridge-utils
```

- 开启并运行libvirtd 服务，以及检查kvm是否加载成功

```bash
[root@bogon ~]# systemctl start libvirtd
[root@bogon ~]# systemctl enable libvirtd
#查看KVM模块是否被正确加载
[root@bogon ~]# lsmod | grep kvm
kvm_intel             170181  0 
kvm                   554609  1 kvm_intel
irqbypass              13503  1 kvm
```

## 配置宿主机网络
默认情况下，KVM 虚拟机是基于 NAT 的网络配置，只有同一宿主机的虚拟键之间可以互相访问，跨宿主机是不能访问的。所以需要和宿主机配置成桥接模式，以便虚拟机可以在局域网内可见。
  
- 配置宿主机的桥接模式  

创建桥接网卡

```bash
[root@bogon ~]# cd /etc/sysconfig/network-scripts
[root@bogon ~]# cp ifcfg-ens33 ifcfg-br0
```

修改 ifcfg-br0 文件  

```bash
[root@bogon ~]# vim ifcfg-br0
TYPE="Bridge" #将br0指定为桥接类型
BOOTPROTO="static"
DEFROUTE="yes"
PEERDNS="yes"
PEERROUTES="yes"
NAME="br0"
DEVICE="br0" #将em1改为br0
ONBOOT="yes"
DELAY="0"
STP="yes"
IPADDR=192.168.188.109
PREFIX=24
NETMASK=255.255.255.0
GATEWAY=192.168.188.1
DNS1=192.168.188.1
```

修改 ifcfg-ens33， ifcfg-ens33 为宿主机的物理网卡配置文件  

```bash
[root@bogon ~]# vim ifcfg-ens33
TYPE=Ethernet
BOOTPROTO=static
DEVICE=ens33
ONBOOT=yes
BRIDGE=br0
```

重启网络并检查网络情况：

```bash
[root@bogon network-scripts]# systemctl restart network
[root@bogon network-scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP qlen 1000
    link/ether 00:0c:29:a1:78:75 brd ff:ff:ff:ff:ff:ff
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 00:0c:29:a1:78:75 brd ff:ff:ff:ff:ff:ff
    inet 192.168.188.109/24 brd 192.168.188.255 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fea1:7875/64 scope link 
       valid_lft forever preferred_lft forever
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN qlen 1000
    link/ether 52:54:00:97:73:51 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
5: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN qlen 1000
    link/ether 52:54:00:97:73:51 brd ff:ff:ff:ff:ff:ff
[root@bogon network-scripts]# brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.000c29a17875	yes		ens33
virbr0		8000.525400977351	yes		virbr0-nic
```

## 通过界面创建KVM-win7虚拟机

- centos桌面版系统下运行 virt-manager，创建KVM虚拟机的管理端界面程序  

![创建KVM虚拟机的管理端界面程序](/images/rancher/win7-kvm/15042794157148.jpg)  

如果centos7安装的是mini版本的系统，这个界面是弹不出来。需要安装桌面版本

![新建虚拟机](/images/rancher/win7-kvm/15042837170527.jpg)  

![](/images/rancher/win7-kvm/15042838698836.jpg)  

运行到此，我们需要把win7的安装iso映像下载并传入到当前centos7系统中，由于我们需要在kvm虚拟机中安装win7所以需要下载win7的虚拟安装的驱动程序，不然在安装选择虚拟硬盘和虚拟网络时，都找不到对就的硬件。 

- 下载映像和虚拟驱动  
win7的映像就不在这里说明下载方式了，网上搜索win7安装的iso下载下来。
这里主要说一下下载kvm安装windows虚拟驱动程序：
通过浏览器打开`https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.136-1/`
  
![下载kvm安装windows虚拟驱动程序](/images/rancher/win7-kvm/15042851617902.jpg)  
 
下载两个文件后，传入当前系统中

```bash
[root@bogon iso]# ll -sh
总用量 4.2G
4.1G -rw-r--r--. 1 qemu qemu 4.1G 8月  31 22:49 cn_windows_7_ultimate_with_sp1_x64_oem.iso
163M -rwxr-xr-x. 1 qemu qemu 163M 9月   1 13:06 virtio-win.iso

```

- 继续安装win7-kvm虚拟机  

![选择win7安装iso映像](/images/rancher/win7-kvm/15042857117144.jpg)
![选择操作系统类型及版本](/images/rancher/win7-kvm/15042858163697.jpg)
![](/images/rancher/win7-kvm/15042861371348.jpg)
![](/images/rancher/win7-kvm/15042862327834.jpg)

- 安装前需要自定义一些虚拟化的配置，以便于后面的在ranchervm环境下运行kvm-win7虚拟机不出蓝屏的问题。

![NIC网络配置](/images/rancher/win7-kvm/15042872495377.jpg)
![虚拟硬盘配置](/images/rancher/win7-kvm/15042873659918.jpg)
![系统启动引导配置](/images/rancher/win7-kvm/15042874519000.jpg)

增加一个硬件IDE-CDROM2-选择前面下载的virtio-win.iso

![](/images/rancher/win7-kvm/15042878514430.jpg)
![开始安装](/images/rancher/win7-kvm/15042879820835.jpg)

- 进入安装win7的界面  

![win7安装界面](/images/rancher/win7-kvm/15043520681409.jpg)

![选择虚拟硬盘](/images/rancher/win7-kvm/15043522644121.jpg)

正常情况下因为没安装虚拟化的驱动程序，安装是找不到硬盘和网络的，所以需要在选择安装盘的界面，浏览安装虚拟化驱动程序，这个时候就要通过浏览找到之前加载的IDE-CDROM2中的virtio-win.isok中的内容大致如下：
`NetKVM/`: Virtio网络驱动
`viostor/`: Virtio块驱动
`vioscsi/`: Virtio SCSI驱动
`viorng/`: Virtio RNG驱动
`vioser/`: Virtio串口驱动
`Balloon/`: Virtio 内存气球驱动
`qxl/`: 用于Windows 7及之前版本的QXL显卡驱动. (virtio-win-0.1.103-1和之后版本会创建)
`qxldod/`: 用于Windows 8及之后版本的QXL显卡驱动. (virtio-win-0.1.103-2和之后版本会创建)
`pvpanic/`: QEMU pvpanic 设备驱动 (virtio-win-0.1.103-2和之后版本会创建)
`guest-agent/`: QEMU Guest Agent 32bit 和 64bit 安装包
`qemupciserial/`: QEMU PCI 串口设备驱动
`*.vfd`: 用于Windows XP下的VFD软驱镜像  

我在这里主要安装网络和硬盘的驱动：viostor、NetKVM找到对应win7系统版本安装完后，就会出现安装的硬盘，后面就安步骤安装win7就可以完成kvm-win7的安装了。

![](/images/rancher/win7-kvm/15043524690030.jpg)

安装虚拟硬盘驱动：

![安装虚拟硬盘驱动](/images/rancher/win7-kvm/15043525084334.jpg)

安装虚拟网络驱动：

![](/images/rancher/win7-kvm/15043526013165.jpg)

安装完后，就会出现虚拟的硬盘的安装界面  

![](/images/rancher/win7-kvm/15043526819944.jpg)

下一步，就等待系统的安装完成：

![](/images/rancher/win7-kvm/15043532760334.jpg)

然后在centos7下的/var/lib/libvirt/images下有刚安装好的win7-kvm虚拟机文件。

```bash
[root@localhost iso]# cd /var/lib/libvirt/images/
[root@localhost images]# ll -sh
总用量 12G
4.0K -rw-r--r--. 1 root root 101 9月   1 15:26 Dockerfile
3.4G -rw-------. 1 qemu qemu 21G 9月   2 19:46 win7-kvm-base.qcow2
8.1G -rw-------. 1 root root 21G 9月   1 15:13 win7-kvm-docker.qcow2
```

win7版的kvm虚拟机安装完成，后结通过虚拟机制作docker镜像
# 通过kvm虚拟机制作的docker镜像  

由于我们后续要通过ranchervm运行kvm虚拟机，所以默认的
RancherVM镜像就已经是捆绑的标准KVM软件的docker镜像。  

- 制作docker版本的kvm-win7镜像，首页压缩原kvm镜像，这样可以使kvm镜像压缩50%以上的空间。

```bash
[root@localhost images]# ll -sh
总用量 17G
4.0K -rw-r--r--. 1 root root 101 9月   1 15:26 Dockerfile
8.3G -rw-------. 1 root root 21G 9月   2 20:31 win7-kvm-base.qcow2
[root@localhost images]# qemu-img convert -O qcow2 -c win7-kvm-base.qcow2 win7-kvm-base.gz.img
[root@localhost images]# ll -sh
总用量 21G
4.0K -rw-r--r--. 1 root root  101 9月   1 15:26 Dockerfile
3.8G -rw-r--r--. 1 root root 3.8G 9月   2 22:53 win7-kvm-base.gz.img
8.3G -rw-------. 1 root root  21G 9月   2 20:31 win7-kvm-base.qcow2
```

通过rancher-base来构建win7-kvm的docker镜像：

```bash
[root@localhost win7-vm]# vim Dockerfile 
FROM rancher/vm-base
COPY win7-kvm-base.gz.img /base_image/win7-kvm-base.gz.img
CMD ["-m 2048m"]
[root@localhost win7-vm]# docker build -t rancher/win7-kvm-docker-base .
Sending build context to Docker daemon  4.322GB
Step 1/3 : FROM rancher/vm-base
 ---> 051656d3329d
Step 2/3 : COPY win7-kvm-base.gz.img /base_image/win7-kvm-base.gz.img
 ---> ffbfd3b91c42
Step 3/3 : CMD -m 2048m
 ---> Running in 7fc2475f49b5
 ---> 80e35bd6ff43
Removing intermediate container 7fc2475f49b5
Successfully built 80e35bd6ff43
Successfully tagged rancher/win7-kvm-docker-base:latest
[root@localhost win7-vm]# docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
rancher/win7-kvm-docker-base     latest              80e35bd6ff43        9 seconds ago       4.35GB
rancher/vm-base                  latest              051656d3329d        15 months ago       288MB
rancher/ranchervm                latest              f3005c29aa04        21 months ago       250MB
```

# 运行并测试docker中win7系统容器
RancherVM镜像是Docker镜像中捆绑的标准KVM镜像，下面我们要通过ranchervm来创建并运行win7的kvm镜像。
首先，确保Docker和KVM都安装在您的系统上。按照分发特定的说明确保KVM工作。我们只需要在内核中启用KVM。我们不需要像任何用户空间的工具qemu-kvm或libvirt。centos7确保启用了KVM，我们最开始就已经讲过。
一旦你设置了Docker和KVM，就运行：

```bash
[root@localhost ~]# docker run -v /var/run:/var/run -p 8080:80 -v /var/lib/rancher/vm:/vm rancher/ranchervm
[root@localhost ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
a926b7dd1740        rancher/ranchervm   "/var/lib/rancher/..."   2 days ago          Up 2 days           0.0.0.0:8080->80/tcp   xenodochial_b
anach
``` 

- 浏览器方式运行win7的docker容器  

打开浏览器输入：https://<KVM hostname>:8080，我们可以通过web浏览器创建虚拟机：

![](/images/rancher/win7-kvm/15043668394036.jpg)

![](/images/rancher/win7-kvm/15043669239599.jpg)

![](/images/rancher/win7-kvm/15043669629235.jpg)

![](/images/rancher/win7-kvm/15043671343957.jpg)

- docker run方式运行win7版的docker容器。
在运行了ranchervm的机器上，运行以下命令，就可以通过命令方式运行kvm的docker容器。


```bash
[root@localhost ~]# docker run -d -p 3389:3389 -e "RANCHER_VM=true" --cap-add NET_ADMIN -v /var/lib/rancher/vm:/vm --device /dev/kvm:/dev/kvm --device /dev/net/tun:/dev/net/tun -v /root/docker-kvm:/base_image  --name docker-win7   rancher/vm-base -m 2048  
37bf25f123f4301bfc0ffc7ccde09c381753d26ffb93ed6a1eae3d97ef271df3
[root@localhost ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                          PORTS                  NAMES
37bf25f123f4        rancher/vm-base     "/var/lib/rancher/..."   5 seconds ago        Up 4 seconds                    22/tcp                 docker-win7
a926b7dd1740        rancher/ranchervm   "/var/lib/rancher/..."   4 days ago           Up 4 days                       0.0.0.0:8080->80/tcp   xenodochial_banach
```
其中-v /root/docker-kvm:/base_image的挂载目录必须放入前面我们安装的kvm虚拟机文件win7-kvm-base.gz.img，这样我们就不需要用dockerfile构建一个新的镜像出来，直接用rancher/vm-base镜像就可以运行。

至此在centos7上通过docker运行一个win7的kvm虚拟机的安装全部完成。

参考：
[ranchervm运行kvm虚拟机参考：https://github.com/rancher/vm](https://github.com/rancher/vm)  
[安装kvm-win7的虚拟机：http://www.gblm.net/439.html](http://www.gblm.net/439.html)

