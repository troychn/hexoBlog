---
title: docker1.12.3 docker-swarm创建与管理集群（一）
toc: true
date: 2017-02-07 19:18:23 Sunday
updated: 2017-02-07 19:49:34 Sunday
categories: [docker]
tags: [linux,docker,swarm]

---
Docker Engine 1.12或更高版本中内置了集群管理和编排功能。Swarm模式侧重于微服务架构。它支持服务协调，负载平衡，服务发现，内置证书循环等。Swarm模式是Docker以简化的服务编排工具。  
# 功能特点：  
## a.与Docker Engine集成的集群管理： 
使用Docker Engine CLI创建一组Docker引擎，您可以在其中部署应用程序服务。您不需要其他编排软件来创建或管理群集。
## b.节点分散式设计：  
Docker Engine不是在部署时处理节点角色之间的差异，而是在运行时处理角色变化。您可以使用Docker Engine部署两种类型的节点，管理节点和工作节点。这意味着您可以从单个服务器构建整个群集。
## c.声明性服务模型：  
Docker Engine使用声明性方法来定义应用程序堆栈中各种服务的所需状态。例如，您可以描述由具有消息队列服务和数据库后端的Web前端服务组成的应用程序。
## d.可扩容与缩放容器：  
对于每个服务，您可以声明要运行的任务数。当您向上或向下缩放时，swarm管理器通过添加或删除任务来自动适应，以保持所需的任务数量来保证集群的可靠状态。
## e.容器容错状态协调：  
群集管理器节点不断监视群集状态，并协调您表示的期望状态的实际状态之间的任何差异。例如，如果设置一个服务以运行容器的10个副本，并且托管其中两个副本的工作程序计算机崩溃，则管理器将创建两个新副本以替换崩溃的副本。 swarm管理器将新副本分配给正在运行和可用的worker节点上。
## f.多主机网络：  
您可以为服务指定覆盖网络。当swarm管理器初始化或更新应用程序时，它会自动为覆盖网络上的容器分配地址。
## g.服务发现：  
Swarm管理器节点为swarm中的每个服务分配唯一的DNS名称，并负载平衡运行的容器。您可以通过嵌入在swarm中的DNS服务器查询在群中运行的每个容器。
## h.负载平衡：  
您可以将服务的端口公开给外部负载平衡器。在内部，swarm允许您指定如何在节点之间分发服务容器。
## m.缺省安全：  
群中的每个节点强制执行TLS相互验证和加密，以保护其自身与所有其他节点之间的通信。您可以选择使用自签名根证书或来自自定义根CA的证书。
## n.滚动更新：  
在已经运行期间，您可以增量地应用服务更新到节点。 swarm管理器允许您控制将服务部署到不同节点集之间的延迟。如果出现任何问题，您可以将任务回滚到服务的先前版本。

# 环境准备：

IP | 主机名 | 角色
---|---|--- 
172.19.6.222 | cloud01 | manager
172.19.6.223 | cloud02 | worker
172.19.6.224 | cloud03 | worker

# 创建swarm集群  
## 创建集群的前提工作：
### 开放swarm需求的端口：  
在创建集群前，如果开启了防火墙，请确认三台主机的防火墙能让swarm需求的端口开放，需要打开主机之间的端口，以下端口必须可用。在某些系统上，这些端口默认为打开。   
**2377**：TCP端口2377用于集群管理通信  
**7946**：TCP和UDP端口7946用于节点之间的通信  
**4789**：TCP和UDP端口4789用于覆盖网络流量  
如果您计划使用加密（--opt加密）创建覆盖网络，则还需要确保协议50（ESP）已打开。

可以直接禁用系统防火墙来让这些端口通信不受限制，一般测试环境我们都会禁用防火墙 
```bash
[root@localhost ~]# systemctl stop firewalld（立即生效）  
[root@localhost ~]# systemctl disable firewalld（重启生效）
```
### 在三台主机节点上安装好docker1.12或者更高的版本：  
执行docker安装: curl -fsSL https://get.docker.com/ | sh 查看docker最新版本的[安装方法](https://github.com/docker/docker/releases)

## 初始化swarm集群
### manager节点操作
执行docker swarm init --adverties0addr [manager节点的IP]   

```docker
[root@cloud01 ~]# docker swarm init --advertise-addr 172.19.6.222
Swarm initialized: current node (ap235xkm64tj0al4gdzqbpwmi) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2qlhzljar0kw10ke623lvfy9mi9w3fgqgg8z9croosj9d7nl4n-b6789pdldcal2j5c4bs4uyl06 \
    172.19.6.222:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

```
执行以上命令后，操作界面有对应的提示：  
如果想要把worker节点加入到集群中执行：  

```
docker swarm join \
    --token SWMTKN-1-2qlhzljar0kw10ke623lvfy9mi9w3fgqgg8z9croosj9d7nl4n-b6789pdldcal2j5c4bs4uyl06 \
    172.19.6.222:2377
```

### worker节点 cloud02、cloud03上执行加入到集群：
**cloud02:**  
```docker
[root@cloud02 ~]# docker swarm join \
>     --token SWMTKN-1-2qlhzljar0kw10ke623lvfy9mi9w3fgqgg8z9croosj9d7nl4n-b6789pdldcal2j5c4bs4uyl06 \
>     172.19.6.222:2377
This node joined a swarm as a worker.

```
**cloud03:** 

```docker
[root@cloud03 ~]# docker swarm join \
>     --token SWMTKN-1-2qlhzljar0kw10ke623lvfy9mi9w3fgqgg8z9croosj9d7nl4n-b6789pdldcal2j5c4bs4uyl06 \
>     172.19.6.222:2377
This node joined a swarm as a worker.

```

如果后续我们有新的主机被当做manager、worker节点想要加入到集群中来，但要不记得当时创建集群时的token，执行以下命令：

```docker
[root@cloud01 ~]# docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2qlhzljar0kw10ke623lvfy9mi9w3fgqgg8z9croosj9d7nl4n-bpldk53d5yaxqa4gx0rbjjg86 \
    172.19.6.222:2377
    
[root@cloud01 ~]# docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2qlhzljar0kw10ke623lvfy9mi9w3fgqgg8z9croosj9d7nl4n-b6789pdldcal2j5c4bs4uyl06 \
    172.19.6.222:2377
    

```
在另一个想要加入集群的manager、worker节点中执行,以上命令执行的结果；  


在manager节点上执行查看节点命令：  
执行命令docker node ls，查看节点信息。

```docker
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Active        
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        

```
参考：  
[https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/)
[https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/](https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/)

# 集群的基本管理
##  列表节点列表
执行命令docker node ls，查看节点信息。

```docker
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Active        
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        

```
说明：  
**AVAILABILITY列**：  
显示调度程序是否可以将任务分配给节点：  
- Active 意味着调度程序可以将任务分配给节点。  
- Pause 意味着调度程序不会将新任务分配给节点，但现有任务仍在运行。  
- Drain 意味着调度程序不会向节点分配新任务。调度程序关闭所有现有任务并在可用节点上调度它们。  

**MANAGER STATUS列**  
显示节点是属于manager或者worker 
- **没有值** 表示不参与群管理的工作节点。
- **Leader** 意味着该节点是使得群的所有群管理和编排决策的主要管理器节点。
- **Reachable** 意味着节点是管理者节点正在参与Raft共识。如果领导节点不可用，则该节点有资格被选为新领导者。
- **Unavailable** 意味着节点是不能与其他管理器通信的管理器。如果管理器节点不可用，您应该将新的管理器节点加入群集，或者将工作器节点升级为管理器。

##  查看节点的详细信息
您可以在管理器节点上运行docker node inspect<NODE-ID>来查看单个节点的详细信息。 输出默认为JSON格式，但您可以传递--pretty标志以以可读的yml格式打印结果。 例如：

```docker
[root@cloud01 ~]# docker node inspect self --pretty
ID:			ap235xkm64tj0al4gdzqbpwmi
Hostname:		cloud01
Joined at:		2016-11-26 05:34:11.095107616 +0000 utc
Status:
 State:			Ready
 Availability:		Active
Manager Status:
 Address:		172.19.6.222:2377
 Raft Status:		Reachable
 Leader:		Yes
Platform:
 Operating System:	linux
 Architecture:		x86_64
Resources:
 CPUs:			4
 Memory:		7.912 GiB
Plugins:
  Network:		bridge, host, null, overlay
  Volume:		local
Engine Version:		1.12.3

[root@cloud01 ~]# 

```
self为当前manager节点
##  更新节点的可见性状态  

```docker
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Active        
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        
[root@cloud01 ~]# docker node update --availability Drain cloud02 （修改cloud02为不可用状态）
cloud02
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Drain         
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        
[root@cloud01 ~]# docker node update --availability Active cloud02 （修改cloud02为可用状态）
cloud02
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Active        
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        

```
如果想把manager节点，只做为调试的节点，不参于容器的运行，可以按照上面的方式 把manager节点设置为Drain.这样所有的容器运行进，都不会在manager节点上创建或者运行容器，可以提高manager的高可用性。
##  添加或删除标签元数据  
节点标签提供了一种灵活的节点组织方法。 您还可以在服务约束中使用节点标签。 在创建服务时应用约束，以限制调度程序为服务分配任务的节点。

在管理器节点上运行docker node update --label-add以将标签元数据添加到节点。 --label-add标志支持<key>或<key> = <value>对。
对要添加的每个节点标签传递一次--label-add标志：

```docker
[root@cloud01 ~]# docker node update --label-add manager --label-add manager=baz cloud01
cloud01
[root@cloud01 ~]# docker node inspect self --pretty
ID:			ap235xkm64tj0al4gdzqbpwmi
Labels:
 - manager = baz
Hostname:		cloud01
Joined at:		2016-11-26 05:34:11.095107616 +0000 utc
Status:
 State:			Ready
 Availability:		Active
Manager Status:
 Address:		172.19.6.222:2377
 Raft Status:		Reachable
 Leader:		Yes
Platform:
 Operating System:	linux
 Architecture:		x86_64
Resources:
 CPUs:			4
 Memory:		7.912 GiB
Plugins:
  Network:		bridge, host, null, overlay
  Volume:		local
Engine Version:		1.12.3

```

使用docker节点更新为节点设置的标签仅适用于群集内的节点实体。不要将它们与dockerd的docker守护程序标签混淆。
##  升级或降级节点
您可以将工作程序节点提升为manager角色。这在管理器节点不可用或者您希望使管理器脱机以进行维护时很有用。 类似地，您可以将管理器节点降级为worker角色。  
无论您升级或降级节点，您应该始终在群中维护奇数个管理器节点。  
要升级一个节点或一组节点，请从管理器节点运行docker node promote：  

```docker
[root@cloud01 ~]# docker node promote cloud02 cloud03
Node cloud02 promoted to a manager in the swarm.
Node cloud03 promoted to a manager in the swarm.
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Active        Reachable
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        Reachable

```
要降级一个节点或一组节点，请从管理器节点运行docker node demote：

```docker
[root@cloud01 ~]# docker node demote cloud02 cloud03
Manager cloud02 demoted in the swarm.
Manager cloud03 demoted in the swarm.
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Active        
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active   
```
##  退出docker swarm集群
首页查看各节点的状态，在manager节点上  

```
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Ready   Active        
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        

```

在worker节点上运行如下命令

```
[root@cloud02 ~]# docker swarm leave
Node left the swarm.
[root@cloud02 ~]# 
```
然后在manager节点上查看状态：  

```
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
4sy84tap36r6vglmt4y0f3kyb    cloud02  Down    Active        
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        
```
如果想把manager节点上已经退出集群的节点信息删除，执行：

```
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        
[root@cloud01 ~]# docker node rm cloud02
cloud02
[root@cloud01 ~]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
ap235xkm64tj0al4gdzqbpwmi *  cloud01  Ready   Active        Leader
don6u2knphk92ugdvfgdymu2t    cloud03  Ready   Active        

```
参考：  
[https://docs.docker.com/engine/swarm/manage-nodes/](https://docs.docker.com/engine/swarm/manage-nodes/)

