---
title: linux系列(三)-linux在线调整分区大小-centos7
toc: true   // 在文章侧边增加文章目录
date: 6/2/2017 5:26:15 PM 
updated: 6/2/2017 5:26:20 PM 
categories: [linux]
tags: [linux,shell]

---

在使用和维护Linux服务器的过程中有时会出现需要调整分区大小的情况。如果配置了lvm（logical volume management）的话，  
可以很方便使用lvextend/lvreduce无损失增加和减少lvm分区的大小。做任何磁盘操作请做好备份！


## summary（概要）

- 系统环境: centos7
- 情况：
  1. home：50G
  2. root：50G
  3. root分区不够用
- 思路：把home分区的空间划一部分到root分区

```
# 设置home分区大小为200G，释放300G空间
$ lvreduce -L 200G /dev/centos/home

# 将空闲空间扩展到root分区
$ lvextend -l +100%FREE /dev/centos/root

# 使用XFS文件系统自带的命令集增加分区空间
$ xfs_growfs /dev/mapper/centos-root
```

## 实例

### situation

挂载在根目录的分区 `/dev/mapper/centos-root` 爆满，占用100%

```
$ df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   50G   50G   19M 100% /
devtmpfs                  32G     0   32G   0% /dev
tmpfs                     32G     0   32G   0% /dev/shm
tmpfs                     32G  2.5G   29G   8% /run
tmpfs                     32G     0   32G   0% /sys/fs/cgroup
/dev/mapper/centos-home  50G   33M  49G   1% /home
/dev/sda1                497M  238M  259M  48% /boot
tmpfs                    6.3G     0  6.3G   0% /run/user/0
```

### 分析

挂载在根目录的分区空间太小，只有50G，而服务器 `home` 目录为非常用目录，挂在了近50G的空间。

思路：从 `centos-home` 分区划出40G空间到 `centos-root` 分区。

### 操作

#### 1.查看各分区信息

```
[root@registry-jenkins ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/centos/swap
  LV Name                swap
  VG Name                centos
  LV UUID                16Tima-Q9Us-F2NC-sPqb-YmPo-qxjl-zQZdRB
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain, 2016-11-09 15:45:17 +0800
  LV Status              available
  # open                 2
  LV Size                4.00 GiB
  Current LE             1024
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:1
   
  --- Logical volume ---
  LV Path                /dev/centos/home
  LV Name                home
  VG Name                centos
  LV UUID                r42xE3-WwVn-an7C-WQzJ-NK3R-vg10-aWnHwE
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain, 2016-11-09 15:45:18 +0800
  LV Status              available
  # open                 1
  LV Size                50.45 GiB
  Current LE             3444
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2
   
  --- Logical volume ---
  LV Path                /dev/centos/root
  LV Name                root
  VG Name                centos
  LV UUID                d74yiv-jdym-sg5h-v49g-cpxZ-6Ttn-JtUoa7
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain, 2016-11-09 15:45:20 +0800
  LV Status              available
  # open                 1
  LV Size                50.05 GiB
  Current LE             23054
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
```

#### 2.umount卸载/home目录

```
[root@localhost ~]# umount /home/
```

#### 3.减少/home分区空间

```
# 释放 /dev/centos/home 分区 40G 的空间
# 命令设置 /dev/centos/home 分区 10G空间
$ lvreduce -L 10G /dev/centos/home
WARNING: Reducing active logical volume to 200.00 GiB.
 THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce centos/home? [y/n]: y
 Size of logical volume centos/home changed from 50.70 GiB (121778 extents) to 10.00 GiB (51200 extents).
 Logical volume centos/home successfully resized.
```

#### 4.增加/root分区空间

```
$ lvextend -l +100%FREE /dev/centos/root
Size of logical volume centos/root changed from 50.06 GiB (12816 extents) to 90.76 GiB (83394 extents).
Logical volume centos/root successfully resized.
```

#### 5.扩展XFS文件空间大小

```
$ xfs_growfs /dev/mapper/centos-root
meta-data=/dev/mapper/centos-root isize=256    agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal               bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 13107200 to 85395456
```

由于xfs文件系统不能执行分区减小的调整！

```
[root@localhost ~]# xfs_growfs /dev/mapper/centos-home
xfs_growfs: /dev/mapper/centos-home is not a mounted XFS filesystem
[root@localhost ~]# mount /dev/mapper/centos-home /home/
mount: /dev/mapper/centos-home：不能读超级块
```

这样，只能通过重新格式化这个分区，格式化后才能再次挂载到home下

```
[root@localhost ~]# mkfs.xfs /dev/mapper/centos-home -f
meta-data=/dev/mapper/centos-home isize=512    agcount=4, agsize=41156608 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=164626432, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=80384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
  
[root@localhost ~]# mount /dev/mapper/centos-home /home/

[root@localhost ~]# reboot
```

再次查看分区，发现home分区已经减小了100G,只不过这个分区里之前的数据都没有了。

完成