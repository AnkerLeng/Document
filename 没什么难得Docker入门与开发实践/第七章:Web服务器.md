7.2.2 安装Apache
docker run -it --name apache ubuntu:16.04 /bin/bash
apt-get update && apt-get -y install apache2

a2ensite default-ssl

a2enmod ssl


7.2.3
使用Dockerfile构建Apache镜像
```
#Apache  server
#VERSION 0.0.1

#基础镜像
FROM ubuntu:16.0.4

#MAINTAINER You Ming <youming@funcuter.org>

#安装 Apache
RUN apt-get update && apt-get -y install apache2 && apt-get clean

#开启HTTPS支持
RUN /usr/sbin/a2ensite default-ssl

RUN /usr/sbin/a2enmod ssl

#对外暴露HTTP使用的80端口和HTTPS的443端口

EXPOSE 80

EXPOSE 443

#启动命令,通过-D参数切换到前台运行

CMD ["/usr/sbin/apache2ctl","-D","FOREGROUND"]
```

7.2.4
测试Apache容器
docker run -d -P --name apache ymdot/apache

docker ps

7.3
Nginx
7.3.2
安装Nginx
docker run -it --name nginx debian:jessie /bin/bash

apt-get update


Nginx以模块化设计,所以可以很容易的将不同功能的模块进行组合,已达到用户需求,
为了降低镜像层的容量通过APT中的--no-install-recommends --no-install-suggests来禁止推荐安装
apt-get install --no-install-recommends --no-install-suggests -y nginx


因为可能会用到HTTPS,所以还需要安装CA证书来提供相关的支持
apt-get install -y ca-certificates


7.3.3
构建Nginx镜像
```
#Nginx server

#VERSION 0.0.1

#基础镜像

FROM debian:jessie

#维护者信息

#MAINTAINER You Ming <youming@funcuter.org>

#安装Nginx

RUN apt-get update && apt-get install --no-install-recommends --no-install-suggests -y ca-certificates nginx

#对外暴露HTTP使用的80端口和HTTPS的443端口

EXPOSE 80 443

#启动命令,通过-g参数修改配置,让Nginx使用前台运行模式

CMD ["nginx","-g","daemon off;"]
```

7.3.4
测试Nginx镜像
docker run -d --name nginx -P ymdot/nginx

docker port nginx

7.4
Tomcat

7.4.2
安装Tomcat
docker run -it --name tomcat java:8-jre /bin/bash

apt-get update

apt-get install -y tomcat8

7.4.3
构建Tomcat镜像

```
#Tomcat server

#VERSION 0.0.1

#基础镜像

FROM java:8-jre

#维护者信息

#MAINTAINER You Ming <youming@funcuter.org>

#安装Tomcat

RUN apt-get update && apt-get install -y tomcat8

#对外暴露Tomcat默认端口

EXPOSE 8080

#启动命令,通过-g参数修改配置,让Tomcat使用前台运行模式

CMD ["/usr/share/tomcat8/bin/catalina.sh","run"]
```
docker build -t ymdot.tomcat .























