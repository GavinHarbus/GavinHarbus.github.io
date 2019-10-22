---
layout:     post
title:      Docker创建SSH镜像
subtitle:   基本使用
date:       2019-10-22
author:     Gavin
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 云计算
---

> 柴门闻犬吠
> 
> 风雪夜归人

# 前言

#### 容器的管理

之前学习了一些进入容器的方法，比如attach、exec等命令，但这些命令无法远程管理容器，因此需要远程管理容器的时候，就需要容器支持SSH了。

---

# 操作

#### 基于commit命令

1. 使用ubuntu创建一个容器  
	
	```shell
	docker run -it ubuntu:lastest /bin/bash
	```  
2. 更新apt缓存，并安装启动openssh\-server

	```shell
	apt update && apt install openssh-server -y
	
	mkdir -p /var/run/sshd
	/usr/sbin/sshd -D &
	```

3. 修改SSH安全登录配置，取消PAM登陆限制
4. 在root目录下创建.ssh目录，并复制登陆需要的公钥信息
5. 创建自启动的SSH服务  

	```shell
	#! /bin/bash
	/usr/sbin/sshd -D
	```
6. 保存镜像
	
	```shell
	docker commit fc1 sshd:ubuntu
	```
	
7. 使用镜像

	```shell
	docker run -p 10022:22 -d sshd:ubuntu /run.sh
	```
	
#### 使用Dockerfile创建

1. 创建工作目录和Dockerfile  
	
	```shell
	mkdir sshd_ubuntu
	cd sshd_ubuntu
	touch Dockerfile run.sh
	```
2. 编写Dockerfile

	```text
	FROM ubuntu:14.04
	MAINTAINER docker_user (user@email.com)
	RUN apt update \
	&& apt install -y openssh-server \
	&& mkdir -p /var/run/sshd \
	&& mkdir -p /root/.ssh \
		
	ADD run.sh /run.sh
		
	RUN chmod 755 /run.sh
		
	EXPOSE 22
		
	CMD ["run.sh"]
	```
3. 创建镜像

	```shell
	docker build -t sshd:Dockerfile .
	```

