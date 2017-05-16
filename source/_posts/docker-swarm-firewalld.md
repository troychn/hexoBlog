---
title: 如何在linux配置DockerSwarm的Linux防火墙  
toc: true  
date: 2017-05-16 19:18:23 Sunday  
updated: 2017-05-16 19:49:34 Sunday  
categories: [docker swarm]  
tags: [linux,docker,swarm]  

---

## 介绍

Docker Swarm是Docker的一个功能，可以轻松地在规模上运行Docker主机和容器。Docker Swarm或Docker集群由一个或多个Dockerized主机组成，它们作为管理器节点和任意数量的工作节点。设置这样的系统需要仔细操纵Linux防火墙。

Docker Swarm正常工作所需的网络端口有： 

**TCP端口 2376 **   
用于安全Docker客户端通信。Docker Machine可以使用此端口。Docker Machine用于编排Docker主机。  
**TCP端口 2377 **   
。该端口用于Docker群集或群集节点之间的通信。它只需要在管理器节点上打开。  
**TCP和UDP端口7946**   
用于节点间的通信（容器网络发现）。  
**4789**  
用于覆盖网络流量的UDP端口（容器入口组网）。  
注意：除了这些端口之外，  
**端口22  **  
（用于SSH流量）和特定服务在集群上运行所需的任何其他端口都必须打开。

在本文中，您将学习如何使用所有Linux发行版上可用的不同防火墙管理应用程序在Ubuntu 16.04上配置Linux防火墙。这些防火墙管理应用程序是防火墙，IPTables工具和UFW，简单防火墙。UFW是Ubuntu发行版上的默认防火墙应用程序，包括Ubuntu 16.04。虽然本教程涵盖三种方法，但每种方法都会产生相同的结果，因此您可以选择您最熟悉的方法。

# **先决条件**

在继续阅读本文之前，您应该：

设置构成集群的主机，包括至少一个群组管理器和一个群组工作程序。您可以按照本教程如何在Ubuntu 16.04上使用Docker Machine配置和管理远程Docker主机进行设置。  
**注意：**  
你会注意到命令（和本文中的所有命令）不是前缀sudo。这是因为假设您
docker-machine ssh
使用Docker Machine配置之后，使用该命令登录到服务器。

### 方法1 - 使用UFW打开Docker Swarm端口

如果您只是设置Docker主机，则UFW已经安装。您只需要启用和配置它。按照本指南了解有关在Ubuntu 16.04上使用UFW的更多信息。

在将用作Swarm管理器的节点上执行以下命令：

```
ufw allow 22/tcp
ufw allow 2376/tcp
ufw allow 2377/tcp
ufw allow 7946/tcp
ufw allow 7946/udp
ufw allow 4789/udp
```

之后，重新加载UFW：

```
ufw reload
```

如果UFW未启用，请使用以下命令：

```
ufw enable
```

这可能不是必需的，但是无论何时更改并重新启动防火墙，都不会重新启动Docker守护程序：

```
systemctl restart docker
```

然后在作为工作者的每个节点上执行以下命令：

```
ufw allow 22/tcp
ufw allow 2376/tcp
ufw allow 7946/tcp
ufw allow 7946/udp
ufw allow 4789/udp
```

之后，重新加载UFW：

```
ufw reload
```

如果UFW未启用，请启用它：

```
ufw enable
```

然后重新启动Docker守护进程：

```
systemctl restart docker
```

这就是您需要做的，使用UFW打开Docker Swarm的必要端口。

### 方法2 - 使用FirewallD打开Docker Swarm端口

FirewallD是基于Fedora，CentOS和其他Linux发行版的默认防火墙应用程序。但是FirewallD也可以在其他Linux发行版中使用，包括Ubuntu 16.04。

如果您选择使用FirewallD而不是UFW，请先卸载UFW：

```
apt-get purge ufw
```

然后安装FirewallD：

```
apt-get install firewalld
```

验证它是否正在运行：

```
systemctl status firewalld
```

如果没有运行，启动它：

```
systemctl start firewalld
```

然后启用它，以便它在启动时启动：

```
systemctl enable firewalld
```

在作为Swarm管理器的节点上，使用以下命令打开必要的端口：

```
firewall-cmd --add-port=22/tcp --permanent
firewall-cmd --add-port=2376/tcp --permanent
firewall-cmd --add-port=2377/tcp --permanent
firewall-cmd --add-port=7946/tcp --permanent
firewall-cmd --add-port=7946/udp --permanent
firewall-cmd --add-port=4789/udp --permanent
```


```
注意：如果您犯了错误，需要删除条目，请键入：。

firewall-cmd --remove-port=port-number/tcp —permanent
```


之后，重新加载防火墙：

```
firewall-cmd --reload
```

然后重新启动Docker。

```
systemctl restart docker
```

然后在将作为Swarm工作器的每个节点上执行以下命令：

```
firewall-cmd --add-port=22/tcp --permanent
firewall-cmd --add-port=2376/tcp --permanent
firewall-cmd --add-port=7946/tcp --permanent
firewall-cmd --add-port=7946/udp --permanent
firewall-cmd --add-port=4789/udp --permanent
```

之后，重新加载防火墙：

```
firewall-cmd --reload
```

然后重新启动Docker。

```
systemctl restart docker
```

您已成功使用FirewallD打开Docker Swarm所需的端口。

### 方法3 - 使用IPTables打开Docker群集端口

要在任何Linux发行版上使用IPtables，您必须先卸载任何其他防火墙工具。如果您正在从FirewallD或UFW切换，请先卸载它们。

然后安装iptables-persistent软件包，管理自动加载IPtables规则：

```
apt-get install iptables-persistent
```

接下来，使用以下命令刷新现有规则：

```
netfilter-persistent flush
```

现在您可以使用该iptables实用程序添加规则。该第一组命令应该在作为Swarm管理器的节点上执行。

```
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 2376 -j ACCEPT
iptables -A INPUT -p tcp --dport 2377 -j ACCEPT
iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 4789 -j ACCEPT
```

输入所有命令后，将规则保存到磁盘：

```
netfilter-persistent save
```

然后重新启动Docker。

```
sudo systemctl restart docker
```

在将用作Swarm工作的节点上，执行以下命令：

```
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 2376 -j ACCEPT
iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 4789 -j ACCEPT
```

将这些新规则保存到磁盘中：

```
netfilter-persistent save
```

然后重新启动Docker：

```
sudo systemctl restart docker
```

这就是为Docker Swarm使用IPTables打开所需的端口。您可以在教程如何使用Iptables防火墙中了解更多关于这些规则如何工作的信息。

如果您希望在使用此方法后切换到FirewallD或UFW，正确的方法是先停止防火墙：

```
sudo netfilter-persistent stop
```

然后冲洗规则：

```
sudo netfilter-persistent flush
```

最后，将现在的空表保存到磁盘中：

```
sudo netfilter-persistent save
```

然后你可以切换到UFW或FirewallD。

# 结论

FirewallD，IPTables工具和UFW是Linux世界中的三个防火墙管理应用程序。你刚刚学会了如何使用它来打开设置Docker Swarm所需的网络端口。您使用哪种方法只是个人偏好的问题，因为它们都具有同等的功能。