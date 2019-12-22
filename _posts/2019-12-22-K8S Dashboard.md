---
layout:     post
title:      K8S Dashboard
subtitle:   集群实践
date:       2019-12-22
author:     Gavin
header-img: img/post-bg-night-lights.jpg
catalog: true
tags:
    - 云计算
---

> 细雨斜风作晓寒
> 
> 淡烟疏柳媚晴滩

#### install and use

1. deploy dashboard

	```
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
	```
	
2. create role and bind role

	```
	vim dashboard-adminuser.yaml	
	```
	
	
		apiVersion: v1
		kind: ServiceAccount
		metadata:
		  name: admin-user
		  namespace: kubernetes-dashboard
		  	
		---
		
		apiVersion: rbac.authorization.k8s.io/v1
		kind: ClusterRoleBinding
		metadata:
		  name: admin-user
		roleRef:
		  apiGroup: rbac.authorization.k8s.io
		  kind: ClusterRole
		  name: cluster-admin
		subjects:
		- kind: ServiceAccount
		  name: admin-user
		  namespace: kubernetes-dashboard

3. apply role

	```
	kubectl apply -f dashboard-adminuser.yaml
	```
	
4. get the login token

	```
	kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
	```

5. create certs

	```
	grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
	```
	
	```
	grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
	```
	
	```
	openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
	```

6. browser import p12 file

#### tips

Dashboard功能相比OpenShift确实简陋了很多，但之后内部PaaS项目也可以参考