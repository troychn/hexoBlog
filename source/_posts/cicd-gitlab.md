---
title: 持续集成(CI)与持续交付(CD)之gitlab服务安装与配置
toc: true   // 在文章侧边增加文章目录
date: 01/04/2018 5:26:15 PM 
updated: 01/04/2018 5:26:20 PM 
categories: [CI/CD,gitlab]
tags: [gitlab,docker,linux]
---

创建高质量应用程序并不是件容易的事情。要怎样做才能更快地创建用户高质量应用程序并且能够不断改进它们呢？这就是需要引入持续集成和持续交付（CI / CD）。团队里的人都在同一个产品上进行实时工作，所以在软件开发过程中使用CI时，你可以实现更快的速度、更好的稳定性和更强的可靠性。并且在开发过程的早期，开发人员能够发现和解决任何编码问题，使它们在成为下游主要问题之前得到纠正。这样可以降低错误代码导致的长期开发（和业务）的成本。实施持续交付的主要好处是能够加快应用程序的上市时间。使用CD的公司能大大增加他们的应用程序发行频率。在没有使用CD之前，应用程序发布的频率通常是几个月一次。然而现在使用CD，你可以一个星期发布一次、甚至每天发布多次应用。在竞争激烈的行业中，速度的提高将会使你处于主要优势。
本文主要来介绍安装gitlab服务,以及配置gitlab发送邮件、连接ldap、配置https证书认证，通过生成客户端公钥KEY，可以实现ssh免密码登录，设置全局忽律https证书以主自动保存用户名密码来实现https的免密码登录，实现CI/CD流程中的分支和合并的过程。

## GitLab-CE安装
gitlab安装方式有很多，官网提供了[各个OS环境的安装文档](https://about.gitlab.com/installation/)，为了便于编排、管理，我选择是通过docker方式来安装最新版的GitLab Community Edition.
参考官方docker安装文档：[https://docs.gitlab.com/omnibus/docker/README.html](https://docs.gitlab.com/omnibus/docker/README.html)

### 使用docker compose部署gitlab服务
 
 编写docker-compose.yml文件  
 
```yml
web:
  image: 'gitlab/gitlab-ce:latest'
  restart: always
  container_name: 'gitlab'
  hostname: 'gitlab.troylc.com'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'https://gitlab.troylc.com'
      gitlab_rails['gitlab_shell_ssh_port'] = 10022
  ports:
    - '80:80'
    - '443:443'
    - '10022:22'
  volumes:
    - '/etc/localtime:/etc/localtime'  #配置时钟与宿主机同步
    - '/data/devdata/gitlab/config:/etc/gitlab'
    - '/data/devdata/gitlab/logs:/var/log/gitlab'
    - '/data/devdata/gitlab/data:/var/opt/gitlab'

```
在上面的yml中：

- image: 指定了使用gitlab-ce版本
- container_name: 指定运行的容器名，指定以后比较方便操作
- ports: 指定了需要暴露的端口，其中80和443为http/https端口，10022:22是ssh端口，由于22端口被主机的sshd所占用，所以要另外指定一个端口用来和容器内的ssh通信，值得注意的是，docker会自动配置iptables，添加暴露的端口到入站规则，添加dockerfile中指定的entrypoint端口到NAT转发规则中，也就是说不必再额外配置iptables，方便了我这种每次配置iptables都要现查命令和规则的人
- environment: 指定环境变量，其中GITLAB_OMNIBUS_CONFIG是gitlab的安装配置，安装脚本会读取其值来进行安装前的配置。官方文档中有各配置项的用途和用法。
- volumes: 指定了数据卷的配置，没有具体研究过各volumes的作用，只是从字面上猜测是持久化保存配置、日志、（未知）数据，防止重建容器后丢失这些数据

在docker-compose.yml的路径下执行docker-compose.yml up -d，就会自动安装、运行Gitlab服务，通过 httpS://$hostIP测试是否正常运行

### gitlab服务配置
#### gitlab-设置HTTPS证书
因为我们在docker-compose.yml设置GITLAB_OMNIBUS_CONFIG的参数external_url为https的访问方式,所以我们需要添加对应的证书文件
存放证书的目录是在/etc/gitlab/ssl目录下
参考官网配置https文档:[https://docs.gitlab.com/omnibus/settings/nginx.html#enable-https](https://docs.gitlab.com/omnibus/settings/nginx.html#enable-https)

```
[root@docker-gitlab config]# pwd
/data/devdata/gitlab/config
[root@docker-gitlab config]# mkdir ssl
[root@docker-gitlab config]# ll
总用量 86
-rw------- 1 root root 73047 12月 26 09:21 gitlab.rb
-rw------- 1 root root  9659 12月 26 09:21 gitlab-secrets.json
-rw------- 1 root root   227 12月 20 14:18 ssh_host_ecdsa_key
-rw-r--r-- 1 root root   187 12月 20 14:18 ssh_host_ecdsa_key.pub
-rw------- 1 root root   419 12月 20 14:18 ssh_host_ed25519_key
-rw-r--r-- 1 root root   107 12月 20 14:18 ssh_host_ed25519_key.pub
-rw------- 1 root root  1679 12月 20 14:18 ssh_host_rsa_key
-rw-r--r-- 1 root root   407 12月 20 14:18 ssh_host_rsa_key.pub
drwxr-xr-x 1 root root  4385 12月 20 13:58 ssl
drwxr-xr-x 1 root root     0 12月 20 14:23 trusted-certs
[root@docker-gitlab config]# ll ssl #把生成的自签名的证书放到此目录下
总用量 6
-rw-r--r-- 1 root root 1627 12月 20 13:58 gitlab.troylc.com.crt
-rw-r--r-- 1 root root 1054 12月 20 13:58 gitlab.troylc.com.csr
-rw-r--r-- 1 root root 1704 12月 20 13:58 gitlab.troylc.com.key
[root@docker-gitlab config]# 

```
怎么生成自签名的证书，请参考[自签根证书和多子域名证书在浏览器上变安全绿](http://www.troylc.cc/certificate/nginx/macOs/2017/11/27/nginx-chrome-ce.html)

#### gitlab-设置邮件通知配置
修改容器中的配置文件/etc/gitlab/gitlab.rb，参考官网配置文档：[https://docs.gitlab.com/omnibus/settings/smtp.html#smtp-settings](https://docs.gitlab.com/omnibus/settings/smtp.html#smtp-settings)

```
[root@docker-gitlab config]# docker exec -it gitlab vim /etc/gitlab/gitlab.rb
#修改发件人邮箱地址
gitlab_rails['gitlab_email_from'] = 'gitlab@troylc.com'

#配置gitlab发送邮件信息
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.163.com"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "troylc@163.com"
gitlab_rails['smtp_password'] = "邮箱密码"
gitlab_rails['smtp_domain'] = "163.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = false
:wq
[root@docker-gitlab config]# docker exec -it gitlab gitlab-ctl stop #停止ctl服务
[root@docker-gitlab config]# docker exec -it gitlab gitlab-ctl reconfigure #重新加载配置
[root@docker-gitlab config]# docker exec -it gitlab gitlab-ctl start #重新ctl启动

```

#### gitlab-设置LDAP用户同步配置
修改容器中的配置文件/etc/gitlab/gitlab.rb，参考官网配置文档:[https://docs.gitlab.com/omnibus/settings/ldap.html](https://docs.gitlab.com/omnibus/settings/ldap.html)

```
gitlab_rails['ldap_enabled'] = true

###! **remember to close this block with 'EOS' below**
 gitlab_rails['ldap_servers'] = YAML.load <<-'EOS'
   main: # 'main' is the GitLab 'provider ID' of this LDAP server
     label: 'TROYLC LDAP'
     host: '172.19.61.223' #ldap服务器IP地址
     port: 10389 #ldap端口
     uid: 'cn' 
     bind_dn: 'uid=admin,ou=system'  #管理员登录用户名
     password: 'ladp密码'  #登录密码
     encryption: 'plain' # "start_tls" or "simple_tls" or "plain"
     verify_certificates: true
     active_directory: false
     allow_username_or_email_login: false
     block_auto_created_users: false
     base: 'ou=users,dc=troylc,dc=com'
     user_filter: ''
     attributes:
      username: ['cn']
      email:    ['mail']
      name:      'sn'
EOS

```

### git客户端安装与配置
从官网下载对应系统的git客户端:[https://git-scm.com/downloads](https://git-scm.com/downloads)
![](/images/cicd/gitlab/15151157733275.jpg)
安装完成后，就会有一个git bash在windows系统下
![](/images/cicd/gitlab/15151167378745.jpg)
#### 配置SSH-KEY
为方便于客户端以ssh方式clone，push等命令时免输入用户名密码。
官网参考：[https://docs.gitlab.com/ce/ssh/README.html](https://docs.gitlab.com/ce/ssh/README.html)

- 登录git客户端所在主机的bash，注意登录的用户名
- 确认是否有现成ssh公钥,如果有直接查看,目前我已经是生成过，如下图

![](/images/cicd/gitlab/15151175616117.jpg)

- 如果没有，创建一个 SSH key

```
$ ssh-keygen -t rsa -C "your.email@example.com" -b 4096
```
所有的输入请直接回车，
然后在用户的.ssh/目录下有id_rsd.pub这文件，和第二步一样，查看并复制里面的内容，到gitlab服务器下，用自己的用户名密码登录进入后。
**点击用户头像->setting->SSH Keys,把刚才复制的内容，填写到KEY的文本框中保存。**
![](/images/cicd/gitlab/15151264335404.jpg)

#### 配置访问git仓库时发生SSL证书错误
有时候访问gitlab仓库时提示证书认证错误，为了解决这个问题，可以在客户端的git环境中执行以下命令：

```
$ git config --global http.sslverify “false”
```

#### 配置git客户端全局记住用户名密码
如果上面的方式，觉得太繁琐，可以在客户端设置保存用户名密码，这样也可以每次与服务端交互操作时，也不用输入用户密码。
只需在客户执行一下命令：

```
$ git config --global credential.helper store
```
第一输入用户名密码后，会生成~/.git-credentials文件并保存用户名密码


## 结束
至此gitlab有服务端与客户端已经准备就绪，下一步我会针对git的fork工作流进行一次讲解与分析


