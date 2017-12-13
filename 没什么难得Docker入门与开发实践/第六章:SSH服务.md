## 没什么难得Docker入门与开发实践




第二部分 实践篇

SSH服务

创建两个数据卷容器

存放nginx配置
docker create --name config -v /etc/nginx alpine echo Nginx Configuration

存放code
docker create --name code -v /var/web alpine echo Web Root

创建一个Nginx容器
docker create --name web --volumes-from code --volumes-from config -p 80:80 -p 443:443 nginx:1.10

创建ssh容器
sudo docker pull rastasheep/ubuntu-sshd


启动SSH容器并挂载两个数据卷容器中的数据卷
docker run -d -P --name sshd --volumes-from code --volumes-from config rastasheep/ubuntu-sshd


查询Docker随机分配的端口
docker port sshd 22


使用本地SSH工具连接这个SSH容器提供的SSH服务
ssh root@host -p 32768
默认密码root 进入容器后修改或创建新用户


通过SSH工具将配置文件和程序代码配置到对应的数据卷中所有的准备工作做完后就可以启动web服务了
docker start web

6.2
构建SSH服务镜像

6.2.1
两种方式构建容器 一种是创建容器并在容器中直接进行修改再将容器提交为新的镜像 另一种是将构镜像的指令全部写到Dockerfile中再通过docker build 构建镜像
推荐使用dockerfile 结构清晰可移植一致性   docker commit 无法保证构建容器之前的容器状态是稳定的  

6.2.2
通过提交构建
docker run -it ubuntu:16.04 /bin/bash

使用apt的更新命令对系统已安装软件进行更新
apt-get update

软件更新完成后就可以进行OpenSSH安装 OpenSSH的服务端程序在apt软件仓库名称为openssh-server 
apt-get install -y openssh-server


安装完成OpenSSH的SSH服务后不要急于退出   创建用来登录的账户
groupadd wgroup
useradd -g wgroup wuser
passwd wuser
1234

默认openssh禁止了root用户的登录
ssh服务配置文件放在 /etc/ssh/sshd_config 文件中 PermitRootLogin的配置项就是表述允许root用户登录  改为yes才能打开root用户密码登录

docker 系统镜像都比较干净 没有安装vim编辑器  vim非必须运行软件  受用sed编辑命令来修改配置

sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
more /etc/ssh/sshd_config |grep PermitRootLogin


关闭容器并将对容器的更改提交为镜像
exit
docker commit -m"SSH Server" 15f13f07a4fa ymdot/sshd


6.2.3
使用Dockerfile构建

根据前面搭建SSH服务的操作将整个过程写成一个Dockerfile文件
```
#SSH Server

#VERSION 0.0.1

#基础镜像

FROM ubuntu:16.04

#维护者信息

MAINTAINER You Ming <youming@funcuter.org>

#更新软件和安装OpenSSH

RUN apt-get update && apt-get install -y openssh-server

#建立OpenSSH的运行目录

RUN mkdir /var/run/sshd

#在构建时设置root用户的密码

RUN echo 'root:hellossh' | chpasswd

#打开root账户密码登录

RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

#对外暴露22端口供SSH客户端连接

EXPOSE 22

#启动命令,携带-D参数使SSHD能在前台运行

CMD ["/usr/sbin/sshd", "-D"]
```






























