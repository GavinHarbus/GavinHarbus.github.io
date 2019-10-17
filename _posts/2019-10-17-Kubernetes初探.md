---
layout:     post
title:      Kubernetes初探
subtitle:   概念和基本使用
date:       2019-10-17
author:     Gavin
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - 云计算
---

> 春水碧于天
> 
> 画船听雨眠

# 概念

#### Kubernetes

![](http://45.32.68.50/large/006y8mN6ly1g7z40z9vj4j30yl06pgn7.jpg)  

[**Kubernetes**](https://kubernetes.io)是一个由Google开源的基于容器技术的先进架构方案，目的是实现资源管理的自动化，以及跨多个数据中心的资源利用率最大化，又称为K8S。它是一个完备的分布式系统支撑平台，具有完备的集群管理能力，包括多层次的安全防护和进入机制、多租户应用支撑能力、透明的服务注册和服务发现机制、内建智能负载均衡器、强大的故障发现和自我修复能力、服务滚动升级和在线扩容能力、可拓展的资源自动调度机制，以及多粒度的资源配额管理能力。  
在K8S中，Service是分布式集群架构的核心，一个Service对象拥有如下关键特征：  

+ 拥有一个唯一指定的名字
+ 拥有一个虚拟IP
+ 能够提供某种远程服务能力
+ 能被映射到提供这种服务能力的一组容器应用上

容器具有强大的隔离能力，为此可以将一组提供服务的进程放入容器中进行隔离，为此K8S设计了Pod对象，将每个服务进程包装到相应的Pod中，使其成为Pod中运行的一个容器。Pod运行在一个节点（Node）上，节点可以是物理机，也可以是私有云或共有云上的一个虚拟机。每个节点可以运行几百个Pod，每个Pod中运行一个特殊容器Pause，其他容器则是业务容器，这些业务容器共享Pause容器的网络栈和Volume挂载卷。  

在集群管理方面，K8S将集群中的机器划分为一个Master节点和一群工作节点。其中Master节点上运行着集群管理的一组进程，Kube\-apiserver、kube\-controller\-manager、kube\-scheduler，它们实现整个集群的资源管理、Pod调度、弹性伸缩、安全控制、系统监控和纠错管理等。

---

# 初步使用

#### 安装

1. 安装K8S

	```shell
	sudo yum install -y etcd kubernetes
	```  
	
	但是，由于我之前安装过docker，安装过程中报错了，报错信息如下：  
	
	```
	Error: docker-ce-cli conflicts with 2:docker-1.13.1-103.git7f2769b.el7.centos.x86_64
	Error: docker-ce conflicts with 2:docker-1.13.1-103.git7f2769b.el7.centos.x86_64
	```  
		
	因此先卸载docker  
	
	```shell
	sudo yum remove docker-ce
	sudo yum remove docker-ce-cli
	```  
		
	再安装，成功！！

2. 修改配置文件  

	```shell
	vim /etc/sysconfig/docker
	
	OPTIONS='--selinux-enabled=false --insecure-registry gcr.io'
	
	vim /etc/kubernetes/apiserver
	
	KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
	```  
	
3. 按顺序启动服务

	```shell
	sudo service etcd start
	sudo service docker start
	sudo service kube-apiserver start
	sudo service kube-controller-manager start
	sudo service kube-scheduler start
	sudo service kubelet start
	sudo service kube-proxy start
	```
	
#### 使用

1. 为MySQL创建一个RC定义文件：mysql\-rc.yaml  

	```yaml
	apiVersion: v1
	kind: ReplicationController
	metadata: 
	  name: mysql
	spec:
	  replicas: 1
	  selector:
	    app: mysql
	  template:
	    spec:
	      containers:
	      - name: mysql
	        image: mysql
	        ports:
	        - containerPort: 13306
	        env:
	        - name: MYSQL_ROOT_PASSWORD
	          value: "114114"
	    metadata:
	      labels:
	        app: mysql
	```

2. 在Master节点执行命令

	```shell
	kubectl create -f mysql-rc.yaml
	
	replicationcontroller "mysql" created
	```  
	
3. 检查刚刚创建的RC

	```shell
	kubectl get rc
	
	NAME      DESIRED   CURRENT   READY     AGE
mysql     1         1         0         1m
	```

4. 查看Pod创建情况

	```shell
	kubectl get pods
	
	NAME          READY     STATUS              RESTARTS   AGE
mysql-c6pk4   0/1       ContainerCreating   0          2m
	```  

5. 查看正在运行的容器

	```shell
	docker ps -a
	```  
	
	发现没有容器，再检查Pod，发现状态一直是ContainerCreating，检查问题  
	
	```shell
	kubectl describe pod mysql
	
	image pull failed for registry.access.redhat.com/rhel7/pod-infrastructure:latest
	```  
	
	修复问题：  
	
	```shell
	wget http://mirror.centos.org/centos/7/os/x86_64/Packages/python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm
	sudo rpm -ivh python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm
	
	#PS:实在不行就在/etc/rhsm/ca/下创建一个空redhat-uep.pem
	
	kubectl delete pod mysql-xxx
	kubectl delete rc mysql
	```  
	
6. 编写一个Service定义文件mysql\-svc.yaml

	```yaml
	apiVersion: v1
	kind: Service
	metadata: 
	  name: mysql
	spec:
	  ports: 
	  - port: 13306
	  selector:
	    app: mysql
	
	```  
	
7. 创建一个service服务

	```shell
	kubectl create -f mysql-svc.yaml
	```	  
	
	查看创建的服务：  
	
	```shell
	kubectl get svc
	
	NAME         CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
kubernetes   10.254.0.1      <none>        443/TCP     1d
mysql        10.254.196.28   <none>        13306/TCP   3m
	```  
	
8. 以相同的步骤启动一个tomcat服务

	```yaml
	apiVersion: v1
	kind: ReplicationController
	metadata: 
	  name: myweb
	spec:
	  replicas: 2
	  selector:
	    app: myweb
	  template:
	    spec:
	      containers:
	      - name: myweb
	        image: kubeguide/tomcat-app:v1
	        ports:
	        - containerPort: 8080
	        env:
	        - name: MYSQL_SERVICE_HOST
	          value: "mysql"
	        - name: MYSQL_SERVICE_PORT
	          value: "13306"
	    metadata:
	      labels:
	        app: myweb
	```  
	
	```yaml
	apiVersion: v1
	kind: Service
	metadata: 
	  name: myweb
	spec:
	  type: NodePort
	  ports:
	    -port: 8080
	    nodeport: 30000
	  selector:
	    app: myweb
  
	```  
	
9. 通过相应端口访问服务
