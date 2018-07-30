---
title: ubuntu18.04中安装docker私有镜像仓库harbor
toc: true   // 在文章侧边增加文章目录
date: 14/07/2018 5:26:15 PM 
updated: 14/07/2018 5:26:20 PM 
categories: [harbor]
tags: [linux,ubuntu,docker,harbor,registry]

---

harbor是由VMware中国研发团队负责开发的开源Docker容器镜像仓库，可帮助用户迅速搭建私有的Registry服务，
它提供远程镜像同步功能，实现多数据中心跨云的镜像同步。   
项目地址:[https://github.com/vmware/harbor](https://github.com/vmware/harbor )  
参考官网安装说明:[https://github.com/vmware/harbor/blob/master/docs/installation_guide.md](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)

1. 下载离线安装包：  
   下载地址:[harbor-offline-installer-v1.5.1.tgz](https://storage.googleapis.com/harbor-releases/release-1.5.0/harbor-offline-installer-v1.5.1.tgz)
   
   解压离线压缩包后的目录为：
   
   ```
   root@docker-node07:/ceph-data/harbor# ls
   LICENSE  common  docker-compose.clair.yml   docker-compose.yml  harbor.cfg            install.sh  prepare
   NOTICE   data    docker-compose.notary.yml  ha                  harbor.v1.5.1.tar.gz  log
   root@docker-node07:/ceph-data/harbor# pwd
   /ceph-data/harbor
   ```
   
2. 修改harbor的配置文件
   修改harbor.cfg文件中的内容
   - 修改hostname主机名
   
   ```
    admin@docker-node07:/ceph-data/harbor$ vim harbor.cfg
    ......
    #The IP address or hostname to access admin UI and registry service.
    #DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
    hostname = hub.troylc.com.cn
    ......
   ```
   
   - 修改访问方式为https
   
   ```
    admin@docker-node07:/ceph-data/harbor$ vim harbor.cfg
    ......
    #The protocol for accessing the UI and token/notification service, by default it is http.
    #It can be set to https if ssl is enabled on nginx.
    #ui_url_protocol = http
    ui_url_protocol = https
    ......
   ```
   
   - 修改自签证书存储的位置
   
   ```
    admin@docker-node07:/ceph-data/harbor$ vim harbor.cfg
   ......
   #The path of cert and key files for nginx, they are applied only the protocol is set to https
   #ssl_cert = ./data/cert/server.crt
   #ssl_cert_key = ./data/cert/server.key
   ssl_cert = ./data/cert/hub.troylc.com.cn.crt
   ssl_cert_key = ./data/cert/hub.troylc.com.cn.key
   ......
    ```
    
3. 修改docker-compose启动文件,以及安装文件的权限
 
   - 修改挂载目录的位置（修改所有挂载目录为当前目录下，在挂载目录前加上`'.'` ）。
 
    ```
    version: '2'
    services:
        log:
         image: vmware/harbor-log:v1.5.1
         container_name: harbor-log
         restart: always
         volumes:
           - ./log/harbor/:/var/log/docker/:z
           - ./common/config/log/:/etc/logrotate.d/:z
         ports:
           - 127.0.0.1:1514:10514
         networks:
           - harbor
        registry:
         image: vmware/registry-photon:v2.6.2-v1.5.1
         container_name: registry
         restart: always
         volumes:
           - ./data/registry:/storage:z
           - ./common/config/registry/:/etc/registry/:z
         networks:
           - harbor
         environment:
           - GODEBUG=netdns=cgo
         command:
           ["serve", "/etc/registry/config.yml"]
         depends_on:
           - log
         logging:
           driver: "syslog"
           options:
             syslog-address: "tcp://127.0.0.1:1514"
             tag: "registry"
        mysql:
         image: vmware/harbor-db:v1.5.1
         container_name: harbor-db
         restart: always
         volumes:
           - ./data/database:/var/lib/mysql:z
         networks:
           - harbor
         env_file:
           - ./common/config/db/env
         depends_on:
           - log
         logging:
           driver: "syslog"
           options:
             syslog-address: "tcp://127.0.0.1:1514"
             tag: "mysql"
        adminserver:
         image: vmware/harbor-adminserver:v1.5.1
         container_name: harbor-adminserver
         env_file:
           - ./common/config/adminserver/env
         restart: always
         volumes:
           - ./data/config/:/etc/adminserver/config/:z
           - ./data/secretkey:/etc/adminserver/key:z
           - ./data/:/data/:z
         networks:
           - harbor
         depends_on:
           - log
         logging:
           driver: "syslog"
           options:
             syslog-address: "tcp://127.0.0.1:1514"
             tag: "adminserver"
        ui:
         image: vmware/harbor-ui:v1.5.1
         container_name: harbor-ui
         env_file:
           - ./common/config/ui/env
         restart: always
         volumes:
           - ./common/config/ui/app.conf:/etc/ui/app.conf:z
           - ./common/config/ui/private_key.pem:/etc/ui/private_key.pem:z
           - ./common/config/ui/certificates/:/etc/ui/certificates/:z
           - ./data/secretkey:/etc/ui/key:z
           - ./data/ca_download/:/etc/ui/ca/:z
           - ./data/psc/:/etc/ui/token/:z
         networks:
           - harbor
         depends_on:
           - log
           - adminserver
           - registry
         logging:
           driver: "syslog"
           options:
             syslog-address: "tcp://127.0.0.1:1514"
             tag: "ui"
        jobservice:
         image: vmware/harbor-jobservice:v1.5.1
         container_name: harbor-jobservice
         env_file:
           - ./common/config/jobservice/env
         restart: always
         volumes:
           - ./data/job_logs:/var/log/jobs:z
           - ./common/config/jobservice/config.yml:/etc/jobservice/config.yml:z
         networks:
           - harbor
         depends_on:
           - redis
           - ui
           - adminserver
         logging:
           driver: "syslog"
           options:
             syslog-address: "tcp://127.0.0.1:1514"
             tag: "jobservice"
        redis:
         image: vmware/redis-photon:v1.5.1
         container_name: redis
         restart: always
         volumes:
           - ./data/redis:/data
         networks:
           - harbor
         depends_on:
           - log
         logging:
           driver: "syslog"
           options:
             syslog-address: "tcp://127.0.0.1:1514"
             tag: "redis"
        proxy:
         image: vmware/nginx-photon:v1.5.1
         container_name: nginx
         restart: always
         volumes:
           - ./common/config/nginx:/etc/nginx:z
         networks:
           - harbor
         ports:
           - 80:80
           - 443:443
           - 4443:4443
         depends_on:
           - mysql
           - registry
           - ui
           - log
         logging:
           driver: "syslog"
           options:
             syslog-address: "tcp://127.0.0.1:1514"
             tag: "proxy"
    networks:
    harbor:
     external: false
    ```
   - 运行install.sh文件需要root权限，可以切换到root用户下进行安装，或者用sudo,
   sudo需要对提示没有权限的文件目录进行授权，这个可根据安装提示进行安装

4. 启动安装：

   ```
    admin@docker-node07:~$ sudo ./install.sh
    .......
    admin@docker-node07:~$ docker ps
    CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS                PORTS                                                              NAMES
    55cba7305648        vmware/nginx-photon:v1.5.1             "nginx -g 'daemon of…"   2 days ago          Up 2 days (healthy)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp   nginx
    3e5befa3066d        vmware/harbor-jobservice:v1.5.1        "/harbor/start.sh"       2 days ago          Up 2 days                                                                                harbor-jobservice
    363fd22720e0        vmware/harbor-ui:v1.5.1                "/harbor/start.sh"       2 days ago          Up 2 days (healthy)                                                                      harbor-ui
    2badb2c7dc60        vmware/registry-photon:v2.6.2-v1.5.1   "/entrypoint.sh serv…"   2 days ago          Up 2 days (healthy)   5000/tcp                                                           registry
    2a2d5be16cb3        vmware/harbor-adminserver:v1.5.1       "/harbor/start.sh"       2 days ago          Up 2 days (healthy)                                                                      harbor-adminserver
    4dfefeb9dc2a        vmware/redis-photon:v1.5.1             "docker-entrypoint.s…"   2 days ago          Up 2 days             6379/tcp                                                           redis
    13b7ac8a6a3a        vmware/harbor-db:v1.5.1                "/usr/local/bin/dock…"   2 days ago          Up 2 days (healthy)   3306/tcp                                                           harbor-db
    64f35ad699de        vmware/harbor-log:v1.5.1               "/bin/sh -c /usr/loc…"   2 days ago          Up 2 days (healthy)   127.0.0.1:1514->10514/tcp
   ```


5. 安装问题：

   -  **install.sh Fail to generate key file**
   
   ```
   $ sudo ./install.sh
   
   [Step 0]: checking installation environment ...
   
   Note: docker version: 18.05.0
   
   Note: docker-compose version: 1.17.1
   
   [Step 1]: loading Harbor images ...
   Loaded image: vmware/harbor-jobservice:v1.1.2
   Loaded image: vmware/nginx:1.11.5-patched
   Loaded image: photon:1.0
   Loaded image: vmware/notary-photon:server-0.5.0
   Loaded image: vmware/notary-photon:signer-0.5.0
   Loaded image: vmware/harbor-adminserver:v1.1.2
   Loaded image: vmware/harbor-ui:v1.1.2
   Loaded image: vmware/harbor-log:v1.1.2
   Loaded image: vmware/harbor-db:v1.1.2
   Loaded image: vmware/registry:2.6.1-photon
   Loaded image: vmware/harbor-notary-db:mariadb-10.1.10
   
   
   [Step 2]: preparing environment ...
   Generated and saved secret to file: /data/secretkey
   Generated configuration file: ./common/config/nginx/nginx.conf
   Generated configuration file: ./common/config/adminserver/env
   Generated configuration file: ./common/config/ui/env
   Generated configuration file: ./common/config/registry/config.yml
   Generated configuration file: ./common/config/db/env
   Generated configuration file: ./common/config/jobservice/env
   Generated configuration file: ./common/config/jobservice/app.conf
   Generated configuration file: ./common/config/ui/app.conf
   Fail to generate key file: ./common/config/ui/private_key.pem, cert file: ./common/config/registry/root.crt
   ```
   
   解决方案：
   
   ```
   admin@docker-node07:~$  sudo vim prepare
   to fix this on ubuntu18.04 just edit prepare script and change:
   empty_subj = "/C=/ST=/L=/O=/CN=/"
   to
   empty_subj = "/"
   
   works like a charm here
   
   ```
   
   参考链接：[install.sh Fail to generate key file #2920](https://github.com/vmware/harbor/issues/2920)

   - **docker 登录证书认证问题**
   
   ```
   admin@docker-node07:~$ docker login -u admin -p Harbor12345 hub.topsec.com.cn
   WARNING! Using --password via the CLI is insecure. Use --password-stdin.
   Error response from daemon: Get https://hub.troylc.com.cn/v1/users/: x509: certificate signed by unknown authority
   ```
   
   解决方法：
   
   ```
    admin@docker-node07:~$ sudo mkdir -p /etc/docker/certs.d/hub.troylc.com.cn
    admin@docker-node07:~$ sudo cp data/cert/hub.troylc.com.cn.crt /etc/docker/certs.d/hub.troylc.com.cn/hub.troylc.com.cn.crt
   ```

   参考链接：[关于HTTPS的配置问题 #2452](https://github.com/vmware/harbor/issues/2452)

   - **docker login登录问题**
   
   ```
   admin@docker-node07:~$ docker login -u admin -p Harbor12345 hub.topsec.com.cn
   WARNING! Using --password via the CLI is insecure. Use --password-stdin.
   Error saving credentials: error storing credentials - err: exit status 1, out: `Cannot autolaunch D-Bus without X11 $DISPLAY`
   ```
   解决方法：
   
   ```
   admin@docker-node07:~$ sudo apt-get remove golang-docker-credential-helpers
   sudo: unable to resolve host docker-node07: Resource temporarily unavailable
   [sudo] password for admin:
   Reading package lists... Done
   Building dependency tree
   Reading state information... Done
   The following packages were automatically installed and are no longer required:
     libsecret-1-0 libsecret-common python-asn1crypto python-backports.ssl-match-hostname python-cached-property python-certifi python-cffi-backend
     python-chardet python-cryptography python-dockerpty python-docopt python-enum34 python-funcsigs python-functools32 python-idna python-ipaddress
     python-jsonschema python-mock python-openssl python-pbr python-pkg-resources python-requests python-six python-texttable python-urllib3
     python-websocket python-yaml
   Use 'sudo apt autoremove' to remove them.
   The following packages will be REMOVED:
     docker-compose golang-docker-credential-helpers python-docker python-dockerpycreds
   0 upgraded, 0 newly installed, 4 to remove and 71 not upgraded.
   After this operation, 2438 kB disk space will be freed.
   Do you want to continue? [Y/n] y
   perl: warning: Setting locale failed.
   perl: warning: Please check that your locale settings:
   	LANGUAGE = (unset),
   	LC_ALL = (unset),
   	LC_CTYPE = "UTF-8",
   	LANG = "en_US.UTF-8"
       are supported and installed on your system.
   perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
   locale: Cannot set LC_CTYPE to default locale: No such file or directory
   locale: Cannot set LC_ALL to default locale: No such file or directory
   (Reading database ... 148673 files and directories currently installed.)
   Removing docker-compose (1.17.1-2) ...
   Removing python-docker (2.5.1-1) ...
   Removing python-dockerpycreds (0.2.1-1) ...
   Removing golang-docker-credential-helpers (0.5.0-2) ...
   Processing triggers for man-db (2.8.3-2) ...
   ```
   
   参考链接： [docker login fails while docker-compose is installed on Ubuntu 18.04](https://github.com/docker/compose/issues/6023)



