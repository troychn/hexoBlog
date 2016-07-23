---
title: docker系列(一)docker基础命令与dockerfile文件创建镜像
date: 7/17/2016 9:12:05 PM 
updated: 7/17/2016 9:12:11 PM 
categories: [docker]
tags: [linux,docker]

---

# 前言
近期烦心事太多，包括工作、生活上的。归根结底还是自己不够强大，对于博客还没有坚持一个星期两篇博文，那就先坚持一个星期一篇吧，从简单开始，从放下开始，尽量让自己每时每个，坚持一些向上的的东西。对于这篇文章，docker目前一直在用，所以总结了一些关于docker基础性的东西。

# 正文
## docker基础命令和含义
1. docker version
显示 Docker 版本信息。
2. docker info
显示 Docker 系统信息，包括镜像和容器数。
3. docker search
从 Docker Hub 中搜索符合条件的镜像。
4. docker pull
从 Docker Hub 中拉取或者更新指定镜像。
5. docker login
按步骤输入在 Docker Hub 注册的用户名、密码和邮箱即可完成登录。
6. docker logout
运行后从指定服务器登出，默认为官方服务器。
7. docker images
列出本地所有镜像。其中 [name] 对镜像名称进行关键词查询。
8. docker ps
列出所有运行中容器。
9. docker rmi
从本地移除一个或多个指定的镜像。
10. docker rm
移除一个或多个指定的容器
11. docker history
查看指定镜像的创建历史。
12. docker create|start|stop|restart|pause|unpause
创建、启动、停止、重启、暂停和恢复一个或多个指定容器。
13. docker kill
杀死一个或多个指定容器进程。
14. docker events
从服务器拉取个人动态，可选择时间区间。
15. docker save
docker save > "debian.tar"
将指定镜像保存成 tar 归档文件， docker load 的逆操作。保存后再加载（saved-loaded）的镜像不会丢失提交历史和层，可以回滚。
16. docker load
docker load < debian.tar
从 tar 镜像归档中载入镜像， docker save 的逆操作。保存后再加载（saved-loaded）的镜像不会丢失提交历史和层，可以回滚。
17. docker export
docker export <container>
将指定的容器保存成 tar 归档文件， docker import 的逆操作。导出后导入（exported-imported)）的容器会丢失所有的提交历史，无法回滚。
18. docker import
docker import url|-  "o">[repository[:tag "o">]]
从归档文件（支持远程文件）创建一个镜像， export 的逆操作，可为导入镜像打上标签。导出后导入（exported-imported)）的容器会丢失所有的提交历史，无法回滚。
19. docker top
docker top <running_container>  "o">[ps options]
查看一个正在运行容器进程，支持 ps 命令参数。
20. docker inspect
docker instpect nginx:latest
检查镜像或者容器的参数，默认返回 JSON 格式。
21. docker pause
暂停某一容器的所有进程。
22. docker unpause
docker unpause <container>
恢复某一容器的所有进程。
23. docker tag
docker tag [options "o">] <image>[:tag "o">] [repository/ "o">][username/]name "o">[:tag]
标记本地镜像，将其归入某一仓库。
24. docker push
docker push laozhu/nginx:latest
将镜像推送至远程仓库，默认为 Docker Hub 。
25. docker logs
docker logs [options "o">] <container>
docker logs -f -t --tail= "s2">"10" insane_babbage
获取容器运行时的输出日志。
26. docker run
docker run [options "o">] <image> [ "nb">command]  "o">[arg...]
启动一个容器，在其中运行指定命令。见下面详细说明。

## docker run命令的中的参数以及含义
```bash
docker run [options "o">] <image> [ "nb">command]  "o">[arg...]
```
启动一个容器，在其中运行指定命令。
-a stdin 指定标准输入输出内容类型，可选 STDIN/
STDOUT / STDERR 三项；
-d 后台运行容器，并返回容器ID；
-i 以交互模式运行容器，通常与 -t 同时使用；
-t 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
--name="nginx-lb" 为容器指定一个名称；
--dns 8.8.8.8 指定容器使用的DNS服务器，默认和宿主一致；
--dns-search example.com 指定容器DNS搜索域名，默认和宿主一致；
-h "mars" 指定容器的hostname；
-e username="ritchie" 设置环境变量；
--env-file=[] 从指定文件读入环境变量；
--cpuset="0-2" or --cpuset="0,1,2"绑定容器到指定CPU运行；
-c 可以调整container的cpu优先级。默认情况下，所有的container享有相同的cpu优先级和cpu调度周期。但你可以通过Docker来通知内核给予某个或某几个container更多的cpu计算周期。
　　默认情况下，使用-c或者--cpu-shares 参数值为0，可以赋予当前活动container 1024个cpu共享周期。这个0值可以针对活动的container进行修改来调整不同的cpu循环周期。
　　比如，我们使用-c或者--cpu-shares =0启动了C0，C1，C2三个container，使用-c/--cpu-shares=512启动了C3.这时，C0，C1，C2可以100%的使用CPU资源(1024)，但C3只能使用50%的CPU资源(512)。如果这个host的OS是时序调度类型的，每个CPU时间片是100微秒，那么C0，C1，C2将完全使用掉这100微秒，而C3只能使用50微秒。
-m 可以很方便的调整container所使用的内存资源。如果host支持swap内存，那么使用-m可以设定比host物理内存还大的值。
--net="bridge" 指定容器的网络连接类型，支持 bridge /host / none container:<name|id> 四种类型；
--expose=[] 可以让container接受外部传入的数据。container内监听的port不需要和外部host的port相同。比如说在container内部，一个HTTP服务监听在80端口，对应外部host的port就可能是49880.
　　操作人员可以使用--expose，让新的container访问到这个container。具体有三个方式：
　　1. 使用-p来启动container。
　　2. 使用-P来启动container。
　　3. 使用--link来启动container。
-p 如果使用-p或者-P，那么container会开发部分端口到host，只要对方可以连接到host，就可以连接到container内部。当使用-P时，docker会在host中随机从49153 和65535之间查找一个未被占用的端口绑定到container。你可以使用docker port来查找这个随机绑定端口。
--link=[] 当你使用--link方式时，作为客户端的container可以通过私有网络形式访问到这个container。同时Docker会在客户端的container中设定一些环境变量来记录绑定的IP和PORT。
-v=[]: docker可以支持把一个宿主机上的目录挂载到镜像里。
```bash
docker run -it -v /home/dock/Downloads:/usr/Downloads ubuntu64 /bin/bash
```
通过-v参数，冒号前为宿主机目录，必须为绝对路径，冒号后为镜像内挂载的路径。
现在镜像内就可以共享宿主机里的文件了。
默认挂载的路径权限为读写。如果指定为只读可以用：ro
```bash
docker run -it -v /home/dock/Downloads:/usr/Downloads:ro ubuntu64 /bin/bash
```
docker还提供了一种高级的用法。叫数据卷。
数据卷：“其实就是一个正常的容器，专门用来提供数据卷供其它容器挂载的”。感觉像是由一个容器定义的一个数据挂载信息。其他的容器启动可以直接挂载数据卷容器中定义的挂载信息。
看示例：
```bash
docker run -v /home/dock/Downloads:/usr/Downloads  --name dataVol ubuntu64 /bin/bash
```
创建一个普通的容器。用--name给他指定了一个名（不指定的话会生成一个随机的名子）。
再创建一个新的容器，来使用这个数据卷。
```bash
docker run -it --volumes-from dataVol ubuntu64 /bin/bash
```
--volumes-from用来指定要从哪个数据卷来挂载数据。
-u: USERcontainer中默认的用户是root。但是开发人员创建新的用户之后，这些新用户也是可以使用的。开发人员可以通过Dockerfile的USER设定默认的用户，操作人员可以通过"-u"来覆盖这些参数。
-w: WORKDIR 
container中默认的工作目录是根目录(/)。开发人员可以通过Dockerfile的WORKDIR来设定默认工作目录，操作人员可以通过"-w"来覆盖默认的工作目录。

## 通过Dockerfile构建镜像
### Dockerfiles基础说明
Dockerfiles是由一系列命令和参数构成的脚本，这些命令应用于基础镜像并最终创建一个新的镜像。它们简化了从头到尾的流程并极大的简化了 部署工作。Dockerfile从FROM命令开始，紧接着跟随者各种方法，命令和参数。其产出为一个新的可以用于创建容器的镜像。
### Dockerfile指令介绍：
**FROM**    
  语法：FROM <image>[:<tag>]
  解释：设置要制作的镜像基于哪个镜像，FROM指令必须是整个Dockerfile的第一个指令，如果指定的镜像不存在默认会自动从Docker Hub上下载。        
**MAINTAINER**
  语法：MAINTAINER <name>
  解释：MAINTAINER指令允许你给将要制作的镜像设置作者信息    
**RUN**
  语法：①RUN <command>        #将会调用/bin/sh -c <command>
  RUN ["executable", "param1", "param2"]    #将会调用exec执行，以避免有些时候shell方式执行时的传递参数问题，而且有些基础镜像可能不包含/bin/sh
  解释：RUN指令会在一个新的容器中执行任何命令，然后把执行后的改变提交到当前镜像，提交后的镜像会被用于Dockerfile中定义的下一步操作，RUN中定义的命令会按顺序执行并提交，这正是Docker廉价的提交和可以基于镜像的任何一个历史点创建容器的好处，就像版本控制工具一样。    
**CMD**
  语法：①CMD ["executable", "param1", "param2"]    #将会调用exec执行，首选方式
  CMD ["param1", "param2"]        #当使用ENTRYPOINT指令时，为该指令传递默认参数
  CMD <command> [ <param1>|<param2> ]        #将会调用/bin/sh -c执行
  解释：CMD指令中指定的命令会在镜像运行时执行，在Dockerfile中只能存在一个，如果使用了多个CMD指令，则只有最后一个CMD指令有效。当出现ENTRYPOINT指令时，CMD中定义的内容会作为ENTRYPOINT指令的默认参数，也就是说可以使用CMD指令给ENTRYPOINT传递参数。
  注意：RUN和CMD都是执行命令，他们的差异在于RUN中定义的命令会在执行docker build命令创建镜像时执行，而CMD中定义的命令会在执行docker run命令运行镜像时执行，另外使用第一种语法也就是调用exec执行时，命令必须为绝对路径。        
**EXPOSE**
  语法：EXPOSE <port> [ ...]
  解释：EXPOSE指令用来告诉Docker这个容器在运行时会监听哪些端口，Docker在连接不同的容器(使用–link参数)时使用这些信息。        
**ENV**
  语法：ENV <key> <value>
  解释：ENV指令用于设置环境变量，在Dockerfile中这些设置的环境变量也会影响到RUN指令，当运行生成的镜像时这些环境变量依然有效，如果需要在运行时更改这些环境变量可以在运行docker run时添加–env <key>=<value>参数来修改。
  注意：最好不要定义那些可能和系统预定义的环境变量冲突的名字，否则可能会产生意想不到的结果。    
**ADD**
  语法：ADD <src> <dest>
  解释：ADD指令用于从指定路径拷贝一个文件或目录到容器的指定路径中，<src>是一个文件或目录的路径，也可以是一个url，路径是相对于该Dockerfile文件所在位置的相对路径，<dest>是目标容器的一个绝对路径，例如/home/yooke/Docker/Dockerfile这个文件中定义的，那么ADD /data.txt /db/指令将会尝试拷贝文件从/home/yooke/Docker/data.txt到将要生成的容器的/db/data.txt，且文件或目录的属组和属主分别为uid和gid为0的用户和组，如果是通过url方式获取的文件，则权限是600。
  注意：①如果执行docker build – < somefile即通过标准输入来创建时，ADD指令只支持url方式，另外如果url需要认证，则可以通过RUN wget …或RUN curl …来完成，ADD指令不支持认证。
  <src>路径必须与Dockerfile在同级目录或子目录中，例如不能使用ADD ../somepath，因为在执行docker build时首先做的就是把Dockerfile所在目录包含子目录发送给docker的守护进程。
  如果<src>是一个url且<dest>不是以”/“结尾，则会下载文件并重命名为<dest>。
  如果<src>是一个url且<dest>以“/”结尾，则会下载文件到<dest>/<filename>，url必须是一个正常的路径形式，“http://example.com”像这样的url是不能正常工作的。
  如果<src>是一个本地的压缩包且<dest>是以“/”结尾的目录，则会调用“tar -x”命令解压缩，如果<dest>有同名文件则覆盖，但<src>是一个url时不会执行解压缩。             
**COPY**
  语法：COPY <src> <dest>
  解释：用法与ADD相同，不过<src>不支持使用url，所以在使用docker build – < somefile时该指令不能使用。    
**ENTRYPOINT**
  语法：ENTRYPOINT ["executable", "param1", "param2"]        #将会调用exec执行，首选方式
  ENTRYPOINT command param1 param2             #将会调用/bin/sh -c执行
  解释：ENTRYPOINT指令中指定的命令会在镜像运行时执行，在Dockerfile中只能存在一个，如果使用了多个ENTRYPOINT指令，则只有最后一个指令有效。ENTRYPOINT指令中指定的命令(exec执行的方式)可以通过docker run来传递参数，例如docker run <images> -l启动的容器将会把-l参数传递给ENTRYPOINT指令定义的命令并会覆盖CMD指令中定义的默认参数(如果有的话)，但不会覆盖该指令定义的参数，例如ENTRYPOINT ["ls","-a"]，CMD ["/etc"],当通过docker run <image>启动容器时该容器会运行ls -a /etc命令，当使用docker run <image> -l启动时该容器会运行ls -a -l命令，-l参数会覆盖CMD指令中定义的/etc参数。
  注意：
   ①当使用ENTRYPOINT指令时生成的镜像运行时只会执行该指令指定的命令。
   ②当出现ENTRYPOINT指令时CMD指令只可能(当ENTRYPOINT指令使用exec方式执行时)被当做ENTRYPOINT指令的参数使用，其他情况则会被忽略。    
**VOLUME**
  语法：VOLUME ["samepath"]
  解释：VOLUME指令用来设置一个挂载点，可以用来让其他容器挂载以实现数据共享或对容器数据的备份、恢复或迁移，具体用法请参考其他文章。        
**USER**
  语法：USER [username|uid]
  解释：USER指令用于设置用户或uid来运行生成的镜像和执行RUN指令。        
**WORKDIR**
  语法：WORKDIR /path/to/workdir
  解释：WORKDIR指令用于设置Dockerfile中的RUN、CMD和ENTRYPOINT指令执行命令的工作目录(默认为/目录)，该指令在Dockerfile文件中可以出现多次，如果使用相对路径则为相对于WORKDIR上一次的值，例如WORKDIR /data，WORKDIR logs，RUN pwd最终输出的当前目录是/data/logs。        
**ONBUILD**
  语法：ONBUILD [INSTRUCTION]
  解释：ONBUILD指令用来设置一些触发的指令，用于在当该镜像被作为基础镜像来创建其他镜像时(也就是Dockerfile中的FROM为当前镜像时)执行一些操作，ONBUILD中定义的指令会在用于生成其他镜像的Dockerfile文件的FROM指令之后被执行，上述介绍的任何一个指令都可以用于ONBUILD指令，可以用来执行一些因为环境而变化的操作，使镜像更加通用。
  注意：
   ①ONBUILD中定义的指令在当前镜像的build中不会被执行。
   ②可以通过查看docker inspeat <image>命令执行结果的OnBuild键来查看某个镜像ONBUILD指令定义的内容。
   ③ONBUILD中定义的指令会当做引用该镜像的Dockerfile文件的FROM指令的一部分来执行，执行顺序会按ONBUILD定义的先后顺序执行，如果ONBUILD中定义的任何一个指令运行失败，则会使FROM指令中断并导致整个build失败，当所有的ONBUILD中定义的指令成功完成后，会按正常顺序继续执行build。
   ④ONBUILD中定义的指令不会继承到当前引用的镜像中，也就是当引用ONBUILD的镜像创建完成后将会清除所有引用的ONBUILD指令。
   ⑤ONBUILD指令不允许嵌套，例如ONBUILD ONBUILD ADD . /data是不允许的。
   ⑥ONBUILD指令不会执行其定义的FROM或MAINTAINER指令。

### dockerfile案例
#### 示例一：创建一个MongoDB的镜像
```bash
############################################################
# Dockerfile to build MongoDB container images
# Based on Ubuntu
############################################################
# Set the base image to Ubuntu
FROM ubuntu
# File Author / Maintainer
MAINTAINER Example McAuthor
# Update the repository sources list
RUN apt-get update
################## BEGIN INSTALLATION ######################
# Install MongoDB Following the Instructions at MongoDB Docs
# Ref: http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/
# Add the package verification key
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
# Add MongoDB to the repository sources list
RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/mongodb.list
# Update the repository sources list once more
RUN apt-get update
# Install MongoDB package (.deb)
RUN apt-get install -y mongodb-10gen
# Create the default data directory
RUN mkdir -p /data/db
##################### INSTALLATION END #####################
# Expose the default port
EXPOSE 27017
# Default port to execute the entrypoint (MongoDB)
CMD ["--port 27017"]
# Set default container command
ENTRYPOINT usr/bin/mongod
```
使用上述的Dockerfile，我们已经可以开始构建MongoDB镜像
```bash
sudo docker build -t my_mongodb .
```

#### 示例二：创建一个JAVA Tomcat的镜像
```bash
# VERSION 0.0.1
# 默认ubuntu server长期支持版本，当前是12.04
FROM ubuntu
# 签名啦
MAINTAINER yongboy "yongboy@gmail.com"

# 更新源，安装ssh server
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe"> /etc/apt/sources.list
RUN apt-get update

RUN apt-get install -y openssh-server
RUN mkdir -p /var/run/sshd

# 设置root ssh远程登录密码为123456
RUN echo "root:123456" | chpasswd 

# install vim
RUN apt-get -y remove vim-common
RUN apt-get -y install vim
  
# Install curl  
RUN apt-get -y install curl
RUN 
rm -rf /var/lib/apt/lists/*  
# Install JDK 7  
RUN cd /tmp &&  curl -L 'http://download.oracle.com/otn-pub/java/jdk/7u65-b17/jdk-7u65-linux-x64.tar.gz' -H 'Cookie: oraclelicense=accept-securebackup-cookie; gpw_e24=Dockerfile' | tar -xz  
RUN mkdir -p /usr/lib/jvm  
RUN mv /tmp/jdk1.7.0_65/ /usr/lib/jvm/java-7-oracle/  
  
# Set Oracle JDK 7 as default Java  
RUN update-alternatives --install /usr/bin/java java /usr/lib/jvm/java-7-oracle/bin/java 300    
RUN update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/java-7-oracle/bin/javac 300 
  
ENV JAVA_HOME /usr/lib/jvm/java-7-oracle/  
  
# Install tomcat7  
RUN cd /tmp && curl -L 'http://archive.apache.org/dist/tomcat/tomcat-7/v7.0.8/bin/apache-tomcat-7.0.8.tar.gz' | tar -xz  
RUN mv /tmp/apache-tomcat-7.0.8/ /opt/tomcat7/  
  
ENV CATALINA_HOME /opt/tomcat7  
ENV PATH $PATH:$CATALINA_HOME/bin  
  
ADD tomcat7.sh /etc/init.d/tomcat7  
RUN chmod 755 /etc/init.d/tomcat7  

# 容器需要开放SSH 22端口
EXPOSE 22

# 容器需要开放Tomcat 8080端口
EXPOSE 8080

# 设置Tomcat7初始化运行，SSH终端服务器作为后台运行
ENTRYPOINT service tomcat7 start && /usr/sbin/sshd -D && tail -f /opt/tomcat7/logs/catalina.out
```
**需要注意：**
ENTRYPOINT，表示镜像在初始化时需要执行的命令，不可被重写覆盖，需谨记
CMD，表示镜像运行默认参数，可被重写覆盖
ENTRYPOINT/CMD都只能在文件中存在一次，并且最后一个生效 多个存在，只有最后一个生效，其它无效！
需要初始化运行多个命令，彼此之间可以使用 && 隔开，但最后一个须要为无限运行的命令，需切记！
ENTRYPOINT/CMD，一般两者可以配合使用，比如：
ENTRYPOINT ["/usr/sbin/sshd"]
CMD ["-D"]
在Docker　daemon模式下，无论你是使用ENTRYPOINT，还是CMD，最后的命令，一定要是当前进程需要一直运行的，才能够防容器退出。
以下无效方式：
 ENTRYPOINT service tomcat7 start #运行几秒钟之后，容器就会退出
 CMD service tomcat7 start #运行几秒钟之后，容器就会退出
这样有效：
ENTRYPOINT service tomcat7 start && tail -f /var/lib/tomcat7/logs/catalina.out
或者
CMD service tomcat7 start && tail -f /var/lib/tomcat7/logs/catalina.out
这样也有效：
 ENTRYPOINT ["/usr/sbin/sshd"]
 CMD ["-D"]
具体请参考官方文档：Dockerfiles for Images
构建镜像

脚本写好了，需要转换成镜像：
docker build -t yongboy/java7 .
-t： 为构建的镜像制定一个标签，便于记忆/索引等
. ： 指定Dockerfile文件在当前目录下
网速不太好，会等待很长时间。很多操作可能需要科学上网，逼得我只能一直挂着VPN，方能畅通无阻。
构建镜像完成之后，看看运行效果：
docker run -d -p 22 -p 8080:8080 yongboy/java7
在运行命令中，还得需要显式指定 -p 22 -p 8080:8080，否则在Docker 0.8.1版本中不会主动映射到宿主机上。据悉在Docker 0.4.8版本时，就不担心这个问题。 或者，您要有好的方式，不妨告知于我，谢谢。
在Dockerfile中，若没有使用ENTRYPOINT/CMD指令，若运行多个命令，可以这样做：
docker run -d -p 22 -p 8080 yongboy/java7 /bin/sh -c "service tomcat7 start && /usr/sbin/sshd -D"
提交/保存镜像

创建好的镜像，可以保存到索引仓库中，便于下次使用（当然，我们直接共享Dockerfile，是最简单的事情，:)) ），但毕竟镜像可以做到开箱即用。
https://index.docker.io/ 注册一个账号，例如yongboy
构建镜像
docker build -t yongboy/java7 .
上面已经构建OK的话，可省略此步。
登陆
docker login
提交到Docker索引仓库
docker push yongboy/java7
现在可以起来喝杯热水，出去溜达会，也不一定能够上传完毕，那叫一个慢啊！
上传OK的话，可以得到类似地址：https://index.docker.io/u/yongboy/java7/
如何使用镜像
docker pull yongboy/java7
剩下的步骤，就很简单了。

# 参考：
[docker命令语句](http://wenku.baidu.com/link?url=wP5bW_rRwEDM71Uum8liLgL_aLTBV1JX2tjEijExkRqs-mvcSJdyhmtGoAhfU9v45LU0k0ltyzzCEk67KYxpatINlftgntGl5zRbdEgfJAO "命令")
[dockerfile详解](http://my.oschina.net/2xixi/blog/516951 "dockerfile详解")
