---
title: 自签根证书和多子域名证书在浏览器上变安全绿
toc: true   // 在文章侧边增加文章目录
date: 11/27/2017 5:26:15 PM 
updated: 11/27/2017 5:26:20 PM 
categories: [certificate,nginx,macOs]
tags: [certificate nginx,chrome,macOs,linux]

---

在很多情况下，我们有内部开发环境中，希望用到https访问，并且如果没有正式的证书的话，都是用自签名的证书来进行一根多子的签发不同域名的证书。然后通过在浏览器配置一个根证书就可以得到多个子域名的https访问的证书安全绿的认证。  
本文主要是用sh脚本通过openssl来生成一个自签名的根证书，然后通过这个根证书，签发出多个子域名的子证书，通过web服务器nginx上进行自签名证书的https的配置，在浏览器上通过设置信任证书可以让我们自签的证书也变成安全绿。

# 自签证书的脚本编写

自签证书目录结构如下：

```bash
├── certificate  #存放两个子域名证书的目录
│   ├── dev.troylc.com.cn.crt
│   ├── dev.troylc.com.cn.csr
│   ├── dev.troylc.com.cn.key
│   ├── cloud.troylc.com.cn.crt
│   ├── cloud.troylc.com.cn.csr
│   └── cloud.troylc.com.cn.key
├── createRootCA.sh   #根证书生成脚本
├── createselfsignedcertificate-dev.sh     #dev子域名证书生成脚本
├── createselfsignedcertificate-cloud.sh   #cloud子域名证书生成脚本
├── readme.md   #说明文件
├── rootcertificate     #存放根证书的目录
│   ├── cloudrootCA.key
│   ├── cloudrootCA.pem
│   └── cloudrootCA.srl
├── server.csr.cnf  #生成子证书需要的CN配置文件
├── v3-dev.ext  #生成子证书需要的DNS域名配置文件
└── v3-cloud.ext  #生成子证书需要的DNS域名配置文件

```
## 根证书脚本内容

```bash
#!/usr/bin/env bash
mkdir rootcertificate/
openssl genrsa -des3 -out rootcertificate/cloudrootCA.key 2048
openssl req -x509 -new -nodes -key rootcertificate/cloudrootCA.key -sha256 -days 3650 -out rootcertificate/cloudrootCA.pem
```
运行这个脚本后会首先让我们设置一个证书的密码。然后让我们输入CN的信息。

```bash
troylc@zqy: ls
certificate                          createselfsignedcertificate-cloud.sh readme.md                            server.csr.cnf                       v3-dev.ext
createRootCA.sh                      createselfsignedcertificate-dev.sh   rootcertificate                      v3-cloud.ext
troylc@zqy: ./createRootCA.sh 
mkdir: rootcertificate/: File exists
Generating RSA private key, 2048 bit long modulus
...................................................+++
..................................+++
e is 65537 (0x10001)
Enter pass phrase for rootcertificate/cloudrootCA.key:   #输入密码 我这里全是123456
Verifying - Enter pass phrase for rootcertificate/cloudrootCA.key:   #输入确认密码 123456
Enter pass phrase for rootcertificate/cloudrootCA.key:   #输入上面的密码验证 123456
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----   #以下输入的内容需要和生成子证书所需要的CN配置文件server.csr.cnf 中的CN中的内容一致，如果不一致在浏览器上验证时会报证书与签发的信息不一致。
Country Name (2 letter code) []:CN  
State or Province Name (full name) []:Beijing
Locality Name (eg, city) []:Beijing
Organization Name (eg, company) []:cloud
Organizational Unit Name (eg, section) []:troylc
Common Name (eg, fully qualified host name) []:*.troylc.com.cn
Email Address []:troylc@163.com
troylc@zqy: ll rootcertificate 
total 16
-rw-r--r--  1 troylc  wheel   1.7K Nov 25 23:22 cloudrootCA.key
-rw-r--r--  1 troylc  wheel   1.3K Nov 25 23:24 cloudrootCA.pem

```
## 子证书生成脚本
在生成子证书之前，需要增加三个配置文件和两个生成子域名的脚本文件 
### 子证书配置文件
一个就是上面提到的server.csr.cnf生成子证书的CN配置文件，另两个是两个子域名的配置文件v3-dev.ext、v3-cloud.ext
- server.csr.cnf文件内容如下  

```
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C= CN
ST= Beijing
L= Beijing
O= cloud
OU= troylc
emailAddress= troylc@topsec.com.cn
CN = *.troylc.com.cn
```

- v3-dev.ext和v3-cloud.ext文件内容如下

```
# v3-dev.ext:
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = dev.troylc.com.cn

# v3-cloud.ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = cloud.troylc.com.cn
```
### 子证书的生成脚本
一个生成dev.troylc.com.cn域名的证书脚本文件，一个生成cloud.troylc.com.cn域名的证书脚本文件  

- createselfsignedcertificate-dev.sh内容如下：

```bash
#!/usr/bin/env bash
openssl req -new -sha256 -nodes -out certificate/dev.troylc.com.cn.csr -newkey rsa:2048 -keyout certificate/dev.troylc.com.cn.key -config <( cat server.csr.cnf )

openssl x509 -req -in certificate/dev.troylc.com.cn.csr -CA rootcertificate/cloudrootCA.pem -CAkey rootcertificate/cloudrootCA.key -CAcreateserial -out certificate/dev.troylc.com.cn.crt -days 3650 -sha256 -extfile v3-dev.ext

```
- createselfsignedcertificate-cloud.sh内容如下：

```bash
#!/usr/bin/env bash
openssl req -new -sha256 -nodes -out certificate/cloud.troylc.com.cn.csr -newkey rsa:2048 -keyout certificate/cloud.troylc.com.cn.key -config <( cat server.csr.cnf )

openssl x509 -req -in certificate/cloud.troylc.com.cn.csr -CA rootcertificate/cloudrootCA.pem -CAkey rootcertificate/cloudrootCA.key -CAcreateserial -out certificate/cloud.troylc.com.cn.crt -days 3650 -sha256 -extfile v3-cloud.ext

```

### 运行子证书生成脚本


```bash
troylc@zqy: ls
certificate                          createselfsignedcertificate-cloud.sh readme.md                            server.csr.cnf                       v3-dev.ext
createRootCA.sh                      createselfsignedcertificate-dev.sh   rootcertificate                      v3-cloud.ext
troylc@zqy: ./createselfsignedcertificate-dev.sh 
Generating a 2048 bit RSA private key
...+++
.............................................+++
writing new private key to 'certificate/dev.troylc.com.cn.key'
-----
Signature ok
subject=/C=CN/ST=Beijing/L=Beijing/O=cloud/OU=troylc/emailAddress=troylc@topsec.com.cn/CN=*.troylc.com.cn
Getting CA Private Key
Enter pass phrase for rootcertificate/cloudrootCA.key:
troylc@zqy: ll certificate 
total 24
-rw-r--r--  1 troylc  wheel   1.6K Nov 25 23:49 dev.troylc.com.cn.crt
-rw-r--r--  1 troylc  wheel   1.0K Nov 25 23:49 dev.troylc.com.cn.csr
-rw-r--r--  1 troylc  wheel   1.7K Nov 25 23:49 dev.troylc.com.cn.key
troylc@zqy: ./createselfsignedcertificate-cloud.sh 
Generating a 2048 bit RSA private key
......................................................................................+++
.................................................+++
writing new private key to 'certificate/cloud.troylc.com.cn.key'
-----
Signature ok
subject=/C=CN/ST=Beijing/L=Beijing/O=cloud/OU=troylc/emailAddress=troylc@topsec.com.cn/CN=*.troylc.com.cn
Getting CA Private Key
Enter pass phrase for rootcertificate/cloudrootCA.key:
troylc@zqy: ll certificate                        
total 48
-rw-r--r--  1 troylc  wheel   1.6K Nov 25 23:49 cloud.troylc.com.cn.crt
-rw-r--r--  1 troylc  wheel   1.0K Nov 25 23:49 cloud.troylc.com.cn.csr
-rw-r--r--  1 troylc  wheel   1.7K Nov 25 23:49 cloud.troylc.com.cn.key
-rw-r--r--  1 troylc  wheel   1.6K Nov 25 23:49 dev.troylc.com.cn.crt
-rw-r--r--  1 troylc  wheel   1.0K Nov 25 23:49 dev.troylc.com.cn.csr
-rw-r--r--  1 troylc  wheel   1.7K Nov 25 23:49 dev.troylc.com.cn.key

```



# nginx配置https
nginx的默认配置default.conf中做如下配置：

```
upstream nexusserver {
# Tomcat is listening on default 8090 port
#    ip_hash;
#    server tscweb1:9518 ;
    server nexus:8081 fail_timeout=0;
}
server {
        listen 443 ssl http2;
        server_name dev.troylc.com.cn; #换成你的域名
        ssl on;
        ssl_certificate /etc/nginx/certificate/dev.troylc.com.cn.crt; #证书文件
        ssl_certificate_key /etc/nginx/certificate/dev.troylc.com.cn.key; #秘钥文件
        ssl_session_timeout 5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers  ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers   on;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;   ##访问网站后，由浏览器方记住该域名是受保护的https://，以后的http://访问不用走上边的return

        location /nexus/ {
            proxy_pass  http://nexusserver/nexus/;
            proxy_redirect   off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_set_header   X-Frame-Options  DENY;
            proxy_connect_timeout 60;
            proxy_read_timeout 3600s;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }

       location /home {try_files $uri /index.html;}

       location = / {
           rewrite ^ /home;
       }

       error_page  404              /404.html;

       # redirect server error pages to the static page /50x.html
       #
       error_page   500 502 503 504  /50x.html;
       location = /50x.html {
           root   /usr/share/nginx/html;
       }
}
```
配置上面的配置后，重启启动nginx,让以上配置生成效

# 浏览器验证
当nginx启动后，我们需要去浏览器验证，因为是自签名的证书，所以需要把自签的根证书导入到系统的证书管理的信任证书中去，我这把mac电脑的配置。
在程序中找到钥匙串访问-系统-文件-导入项目找到我们上面生成的根证书中的cloudrootCA.pem文件导入进来，如下图：
![](/images/certificate/15116263436759.jpg)
浏览器上访问nginx提供的服务
![](/images/certificate/15116697191126.png)

![](/images/certificate/15116696692303.png)



