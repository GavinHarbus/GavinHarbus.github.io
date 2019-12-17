---
layout:     post
title:      K8S集群内运行gitlab-runner
subtitle:   集群实践
date:       2019-12-17
author:     Gavin
header-img: img/post-bg-germany-brdge.jpg
catalog: true
tags:
    - 云计算
---

> 长安一片月
> 
> 万户捣衣声

#### 前提

+ 可用的k8s集群
+ 可用的gitlab service
+ k8s集群和gitlab service之间可通信

#### 部署gitlab-runner

1. 拉取gitlab-runner镜像
	因为处于内网，k8s自动拉取镜像太慢，因此我在一台可直连外网的机器上先手动拉取镜像
	```
	docker pull gitlab/gitlab-runner:latest
	```
2. 打包导出镜像，并传给各个工作节点
	```
	docker save -o gitlab-runner.tar gitlab-runner:latest
	```
3. 各节点导入镜像
	```
	docker load -i gitlab-runner.tar
	```
4. 准备configmap配置文件

	```
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: gitlab-runner
	  namespace: default
	data:
	  config.toml: |
	    concurrent = 2
	
	    [[runners]]
	      name = "Kubernetes Runner"
	      url = "http://xxx.xxx.xxx"
	      token = "xxxxx"
	      executor = "shell"
	      [runners.kubernetes]
	        namespace = "xxxx"
	        image = "gitlab/gitlab-runner"
	```

	***PS：这里有一个坑，gitlab-runner只有注册之后才会得到其本身的token，因此这个配置文件一开始可以不使用，注册完之后，为了每次新开gitlab-runner服务避免重复注册，可以使用***

5. 准备deployment文件

	```
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: gitlab-runner
	  namespace: xxxx
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      name: gitlab-runner
	  template:
	    metadata:
	      labels:
	        name: gitlab-runner
	    spec:
	      containers:
	      - args:
	        - run
	        image: gitlab/gitlab-runner:latest
	        imagePullPolicy: IfNotPresent
	        command: [ "/bin/bash", "-ce", "tail -f /dev/null" ]
	        name: gitlab-runner
	        volumeMounts:
	        - mountPath: yourpath
	          name: config
	        - mountPath: yourpath
	          name: cacerts
	          readOnly: true
	      restartPolicy: Always
	      volumes:
	      - configMap:
	          name: gitlab-runner
	        name: config
	      - hostPath:
	          path: yourpath
	        name: cacerts
	
	```

6. 部署

	```
	kubectl apply -f gitlab-runner-configmap.yml
	kubectl apply -f gitlab-runner-deployment.yml
	```

7. 注册以及使用gitlab-runner

	```
	kubectl exec -it gitlab-runner-xxxx /bin/bash
	#做好ip和gitlab service的域名匹配
	vim /etc/hosts
	gitlab-runner restart always
	
	#注册
	gitlab-runner register
	...
	```

	运气好的话，你已经可以提交job了

8. gitlab-ci.yml示例

	```
	# This file is a template, and might need editing before it works on your project.
	# see https://docs.gitlab.com/ce/ci/yaml/README.html for all available options
	
	# you can delete this line if you're not using Docker
	image: busybox:latest
	
	before_script:
	  - echo "Before script section"
	  - echo "For example you might run an update here or install a build dependency"
	  - echo "Or perhaps you might print out some debugging details"
	
	after_script:
	  - echo "After script section"
	  - echo "For example you might do some cleanup here"
	
	build1:
	  stage: build
	  script:
	    - echo "Do your build here"
	  tags:
	    - kubernetes
	
	test1:
	  stage: test
	  script:
	    - echo "Do a test here"
	    - echo "For example run a test suite"
	  tags:
	    - kubernetes
	
	test2:
	  stage: test
	  script:
	    - echo "Do another parallel test here"
	    - echo "For example run a lint test"
	  tags:
	    - kubernetes
	
	deploy1:
	  stage: deploy
	  script:
	    - echo "Do your deploy here"
	  tags:
	    - kubernetes
	
	```

#### 遗留问题

+ 官方推荐使用helm配置k8s gitlab-runner，但是网络似乎有问题
+ gitlab-runner的executor中有kubernetes，但是如何将ip和域名的对应在其启动前注入是个问题（内网环境下）