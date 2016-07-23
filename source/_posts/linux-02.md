---
title: linux系列(二)-linux相关系统进程查看及管理
date: 7/23/2016 5:26:15 PM 
updated: 7/23/2016 5:26:20 PM 
categories: [linux]
tags: [linux,shell]

---

# 前言
接上一篇linux文章，最近一直在学习linux相关的知识，之前也有看过相关对linux系统进程的文章，但一直没有总结和归纳。趁再次学习，把相关的知识记录下来。

# 正文
基于linux系统的相关进程管理命令有：ps、top、kill、
## ps命令：

- **ps aux 查看系统中所有进程，使用BSD操作系统格式**

```bash
[root@iZ23kbqqy35Z ~]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  19284  1508 ?        Ss    2015   0:01 /sbin/init
root         2  0.0  0.0      0     0 ?        S     2015   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S     2015   4:23 [ksoftirqd/0]
root         5  0.0  0.0      0     0 ?        S<    2015   0:00 [kworker/0:0H]
root         7  0.0  0.0      0     0 ?        S     2015   0:01 [migration/0]
root         8  0.0  0.0      0     0 ?        S     2015   0:00 [rcu_bh]
root         9  0.0  0.0      0     0 ?        S     2015  25:32 [rcu_sched]
root        10  0.0  0.0      0     0 ?        S     2015   1:59 [watchdog/0]
root        11  0.0  0.0      0     0 ?        S     2015   1:42 [watchdog/1]
root        12  0.0  0.0      0     0 ?        S     2015   0:01 [migration/1]
root        13  0.0  0.0      0     0 ?        S     2015   3:36 [ksoftirqd/1]
root        15  0.0  0.0      0     0 ?        S<    2015   0:00 [kworker/1:0H]
root        16  0.0  0.0      0     0 ?        S     2015   1:43 [watchdog/2]
root        17  0.0  0.0      0     0 ?        S     2015   0:01 [migration/2]
root        18  0.0  0.0      0     0 ?        S     2015   1:50 [ksoftirqd/2]
......

```
**ps命令的输出格式说明：**

格式 | 含意  
---|--- 
 USER | 该进程是由那个用户产生的；
 PID | 进程的ID号
 %CPU | 该进程占用的CPU资源的百分比，占用越高，进程越耗费资源
 %MEM | 该进程占用物理内存的百分比，占用越高，进程越耗费资源
 VSZ | 该进程占用虚拟内存的大小，单位KB
 RSS | 该进程占用实际物理内存的大小，单位KB
 TTY | 该进程是在那个终端中运行的，其中tty1-tty7代表本地控制台终端，tty1-tty6是本地的字符界面终端，tty7是图形终端。 pts/0-255代表虚拟终端
 STAT | 进程状态，常见的状态有：R：运行、S：睡眠、T：停止状态、s:包含子进程、+：位于后台
 START | 该进程的启动时间
 TIME | 该进程占用CPU的运算时间，注意不是系统时间
 COMMAND | 产生此进程的命令名

- **ps -le 查看系统中所有进程，使用linux标准命令格式**
  
选项 | 含意  
---|--- 
 a | 显示一个终端的所有进程，除了会话引线
 u | 显示进程的归属用户及内存的使用情况
 x | 显示没有控制终端的进程
-l | 长格式显示，显示更加详细的信息
-e | 显示所有进程，和-A作用一致

```bash
[root@iZ23kbqqy35Z ~]# ps -le
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
4 S     0     1     0  0  80   0 -  4821 poll_s ?        00:00:01 init
1 S     0     2     0  0  80   0 -     0 kthrea ?        00:00:00 kthreadd
1 S     0     3     2  0  80   0 -     0 smpboo ?        00:04:23 ksoftirqd/0
1 S     0     5     2  0  60 -20 -     0 worker ?        00:00:00 kworker/0:0H
1 S     0     7     2  0 -40   - -     0 smpboo ?        00:00:01 migration/0
1 S     0     8     2  0  80   0 -     0 rcu_gp ?        00:00:00 rcu_bh
1 S     0     9     2  0  80   0 -     0 rcu_gp ?        00:25:32 rcu_sched
5 S     0    10     2  0 -40   - -     0 smpboo ?        00:01:59 watchdog/0
5 S     0    11     2  0 -40   - -     0 smpboo ?        00:01:42 watchdog/1
1 S     0    12     2  0 -40   - -     0 smpboo ?        00:00:01 migration/1
1 S     0    13     2  0  80   0 -     0 smpboo ?        00:03:36 ksoftirqd/1
1 S     0    15     2  0  60 -20 -     0 worker ?        00:00:00 kworker/1:0H
......

```
- **查看进程树：pstree[选项]**

选项 | 含意  
---|--- 
-p | 显示进程的PID
-u | 显示进程的所属用户

```bash
[root@iZ23kbqqy35Z ~]# pstree -p
init(1)─┬─AliHids(9903)─┬─{AliHids}(9904)
        │               ├─{AliHids}(9905)
        │               ├─{AliHids}(9907)
        │               ├─{AliHids}(9908)
        │               ├─{AliHids}(9909)
        │               ├─{AliHids}(9911)
        │               ├─{AliHids}(9912)
        │               └─{AliHids}(9915)
        ├─AliYunDun(20201)─┬─{AliYunDun}(20202)
        │                  ├─{AliYunDun}(20203)
        │                  ├─{AliYunDun}(20204)
        │                  ├─{AliYunDun}(20205)
        │                  ├─{AliYunDun}(20206)
        │                  ├─{AliYunDun}(20207)
        │                  ├─{AliYunDun}(20208)
        │                  └─{AliYunDun}(20209)
        ├─AliYunDunUpdate(19913)─┬─{AliYunDunUpdat}(19914)
        │                        ├─{AliYunDunUpdat}(19915)
        │                        └─{AliYunDunUpdat}(19921)
        ├─crond(2028)
        ├─gshelld(2075)─┬─{gshelld}(2082)
```

## top命令
- **top [选项]**

```bash
[root@iZ23kbqqy35Z ~]# top
top - 18:30:57 up 255 days,  7:28,  1 user,  load average: 0.00, 0.01, 0.05
Tasks:  87 total,   1 running,  86 sleeping,   0 stopped,   0 zombie
Cpu(s):  0.0%us,  0.2%sy,  0.0%ni, 99.7%id,  0.0%wa,  0.0%hi,  0.0%si,  0.2%st
Mem:    377672k total,   322332k used,    55340k free,    32592k buffers
Swap:   397308k total,    67192k used,   330116k free,    71900k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                                                                                          
 6902 root      20   0 4372m 338m  12m S  0.3  4.4 135:51.36 java                                                                                                                              
 9903 root      20   0  881m  10m 6988 S  0.3  0.1 256:33.01 AliHids                                                                                                                           
20201 root      20   0 63696 5596 3896 S  0.3  0.1  10:07.02 AliYunDun                                                                                                                         
23664 root      20   0 15084 1136  848 R  0.3  0.0   0:00.01 top                                                                                                                               
    1 root      20   0 19284 1508 1228 S  0.0  0.0   0:01.45 init                                                                                                                              
    2 root      20   0     0    0    0 S  0.0  0.0   0:00.02 kthreadd                                                                                                                          
    3 root      20   0     0    0    0 S  0.0  0.0   4:23.54 ksoftirqd/0                                                                                                                       
    5 root       0 -20     0    0    0 S  0.0  0.0   0:00.00 kworker/0:0H                                                                                                                      
    7 root      RT   0     0    0    0 S  0.0  0.0   0:01.79 migration/0                                                                                                                       
    8 root      20   0     0    0    0 S  0.0  0.0   0:00.00 rcu_bh                                                                                                                            
    9 root      20   0     0    0    0 S  0.0  0.0  25:32.77 rcu_sched                                                                                                                         
   10 root      RT   0     0    0    0 S  0.0  0.0   1:59.69 watchdog/0                                                                                                                        
   11 root      RT   0     0    0    0 S  0.0  0.0   1:42.38 watchdog/1                                                                                                                        
   12 root      RT   0     0    0    0 S  0.0  0.0   0:01.76 migration/1
```

- **第一行：**

格式 | 含意  
---|--- 
 18: 30 :57 | 系统当前时间 
 up 255 days,  7:28 | 系统开机到现在经过了多少时间 
1 users | 当前2用户在线 
load average: 0.00, 0.01, 0.05 | 系统1分钟、5分钟、15分钟的CPU负载信息

- **第二行：**

格式 | 含意  
---|--- 
Tasks | 任务; 
87 total| 很好理解，就是当前有87个任务，也就是87个进程。 
1 running | 1个进程正在运行 
86 sleeping | 86个进程睡眠 
0 stopped | 停止的进程数 
0 zombie | 僵死的进程数

- **第三行：**

格式 | 含意  
---|--- 
Cpu(s) | 表示这一行显示CPU总体信息 
0.0%us | 用户态进程占用CPU时间百分比，不包含renice值为负的任务占用的CPU的时间。 
0.7%sy | 内核占用CPU时间百分比 
0.0%ni | 改变过优先级的进程占用CPU的百分比 
99.3%id | 空闲CPU时间百分比 
0.0%wa | 等待I/O的CPU时间百分比 
0.0%hi | CPU硬中断时间百分比 
0.0%si | CPU软中断时间百分比 
注：这里显示数据是所有cpu的平均值，如果想看每一个cpu的处理情况，按1即可；折叠，再次按1；

- **第四行：**

格式 | 含意  
---|--- 
Men | 内存的意思 
8175320kk total | 物理内存总量 
8058868k used | 使用的物理内存量 
116452k free | 空闲的物理内存量 
283084k buffers | 用作内核缓存的物理内存量

- **第五行：**

格式 | 含意  
---|--- 
Swap | 交换空间 
6881272k total | 交换区总量 
4010444k used | 使用的交换区量 
2870828k free | 空闲的交换区量 
4336992k cached | 缓冲交换区总量

- **第六行：进程信息**

格式 | 含意  
---|--- 
PID | 进程的ID 
USER | 进程所有者 
PR | 进程的优先级别，越小越优先被执行 
NInice | 值 
VIRT | 进程占用的虚拟内存 
RES | 进程占用的物理内存 
SHR | 进程使用的共享内存 
S | 进程的状态。S表示休眠，R表示正在运行，Z表示僵死状态，N表示该进程优先值为负数 
%CPU | 进程占用CPU的使用率 
%MEM | 进程使用的物理内存和总内存的百分比 
TIME+ | 该进程启动后占用的总的CPU时间，即占用CPU使用时间的累加值。 
COMMAND | 进程启动命令名称

- **top命令交互操作指令**

格式 | 含意  
---|--- 
q | 退出top命令
<Space> | 立即刷新
s | 设置刷新时间间隔
c | 显示命令完全模式
t: | 显示或隐藏进程和CPU状态信息
m | 显示或隐藏内存状态信息
l | 显示或隐藏uptime信息
f | 增加或减少进程显示标志
S | 累计模式，会把已完成或退出的子进程占用的CPU时间累计到父进程的MITE+
P | 按%CPU使用率排行
T | 按MITE+排行
M | 按%MEM排行
u | 指定显示用户进程
r | 修改进程renice值
k | kill进程
i | 只显示正在运行的进程
W | 保存对top的设置到文件~/.toprc，下次启动将自动调用toprc文件的设置。
h | 帮助命令。
q | 退出


**注：**强调一下，使用频率最高的是P、T、M，因为通常使用top，我们就想看看是哪些进程最耗cpu资源、占用的内存最多； 
注：通过”shift + >”或”shift + <”可以向右或左改变排序列 
如果只需要查看内存：可用free命令。只查看uptime信息（第一行），可用uptime命令；

# 参考：
[linux怎样使用top命令查看系统状态](http://jingyan.baidu.com/article/4d58d5412917cb9dd4e9c0ed.html "没创新")

---

