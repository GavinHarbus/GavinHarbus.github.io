---
layout:     post
title:      K8S集群内运行gitlab-runner
subtitle:   集群实践
date:       2019-12-17
author:     Gavin
header-img: img/post-bg-germany-bridge.jpg
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

9. 修改之前的configmap文件
实际运行时发现不仅要将config.toml挂载进pod，还需要挂载config.toml.lock，否则运行时gitlab-runner进程无法获取到锁，因此configmap.yaml文件修改为如下形式，为方便就不以范用形式进行展示了

	```
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: gitlab-runner
	  namespace: default
	data:
	  config.toml: |
	        concurrent = 1
	        check_interval = 0
	
	        [session_server]
	          session_timeout = 1800
	
	        [[runners]]
	          name = "kubernetes-runner"
	          url = "http://intranet.gitlabsz.siemens.com/"
	          token = "xEb-jrn1s_Viw1ENxytU"
	          executor = "shell"
	          [runners.custom_build_dir]
	          [runners.cache]
	            [runners.cache.s3]
	            [runners.cache.gcs]
	  config.toml.lock: |
	
	```

10. 如此一来，此gitlab-runner便可以使用此配置极其方便地部署了，别忘记启动gitlab-runner

	```
	kubectl exec -it pod-name gitlab-runner restart always
	```

#### helm部署gitlab-runner

1. 准备values.yaml，为helm模板生成提供所需参数
	
	```
	## GitLab Runner Image
	##
	## By default it's using gitlab/gitlab-runner:alpine-v{VERSION}
	## where {VERSION} is taken from Chart.yaml from appVersion field
	##
	## ref: https://hub.docker.com/r/gitlab/gitlab-runner/tags/
	##
	# image: gitlab/gitlab-runner:alpine-v11.6.0
	
	## Specify a imagePullPolicy
	## 'Always' if imageTag is 'latest', else set to 'IfNotPresent'
	## ref: http://kubernetes.io/docs/user-guide/images/#pre-pulling-images
	##
	imagePullPolicy: IfNotPresent
	
	## The GitLab Server URL (with protocol) that want to register the runner against
	## ref: https://docs.gitlab.com/runner/commands/README.html#gitlab-runner-register
	##
	gitlabUrl: http://139.24.120.248/
	
	## The Registration Token for adding new Runners to the GitLab Server. This must
	## be retrieved from your GitLab Instance.
	## ref: https://docs.gitlab.com/ce/ci/runners/README.html
	##
	runnerRegistrationToken: "vrLSyXrLNXKvq5o_5nGo"
	
	## The Runner Token for adding new Runners to the GitLab Server. This must
	## be retrieved from your GitLab Instance. It is token of already registered runner.
	## ref: (we don't yet have docs for that, but we want to use existing token)
	##
	#runnerToken: "vrLSyXrLNXKvq5o_5nGo"
	#
	## Unregister all runners before termination
	##
	## Updating the runner's chart version or configuration will cause the runner container
	## to be terminated and created again. This may cause your Gitlab instance to reference
	## non-existant runners. Un-registering the runner before termination mitigates this issue.
	## ref: https://docs.gitlab.com/runner/commands/README.html#gitlab-runner-unregister
	##
	unregisterRunners: true
	
	## When stopping ther runner, give it time to wait for it's jobs to terminate.
	##
	## Updating the runner's chart version or configuration will cause the runner container
	## to be terminated with a graceful stop request. terminationGracePeriodSeconds
	## instructs Kubernetes to wait long enough for the runner pod to terminate gracefully.
	## ref: https://docs.gitlab.com/runner/commands/#signals
	terminationGracePeriodSeconds: 3600
	
	## Set the certsSecretName in order to pass custom certficates for GitLab Runner to use
	## Provide resource name for a Kubernetes Secret Object in the same namespace,
	## this is used to populate the /home/gitlab-runner/.gitlab-runner/certs/ directory
	## ref: https://docs.gitlab.com/runner/configuration/tls-self-signed.html#supported-options-for-self-signed-certificates
	##
	# certsSecretName:
	
	## Configure the maximum number of concurrent jobs
	## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-global-section
	##
	concurrent: 10
	
	## Defines in seconds how often to check GitLab for a new builds
	## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-global-section
	##
	checkInterval: 10
	
	## Configure GitLab Runner's logging level. Available values are: debug, info, warn, error, fatal, panic
	## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-global-section
	##
	# logLevel:
	
	## Configure GitLab Runner's logging format. Available values are: runner, text, json
	## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-global-section
	##
	# logFormat:
	
	## For RBAC support:
	rbac:
	  create: false
	  ## Define specific rbac permissions.
	  # resources: ["pods", "pods/exec", "secrets"]
	  # verbs: ["get", "list", "watch", "create", "patch", "delete"]
	
	  ## Run the gitlab-bastion container with the ability to deploy/manage containers of jobs
	  ## cluster-wide or only within namespace
	  clusterWideAccess: false
	
	  ## Use the following Kubernetes Service Account name if RBAC is disabled in this Helm chart (see rbac.create)
	  ##
	  # serviceAccountName: default
	
	## Configure integrated Prometheus metrics exporter
	## ref: https://docs.gitlab.com/runner/monitoring/#configuration-of-the-metrics-http-server
	metrics:
	  enabled: true
	
	## Configuration for the Pods that that the runner launches for each new job
	##
	runners:
	  ## Default container image to use for builds when none is specified
	  ##
	  image: ubuntu:16.04
	
	  ## Specify one or more imagePullSecrets
	  ##
	  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
	  ##
	  # imagePullSecrets: []
	
	  ## Specify the image pull policy: never, if-not-present, always. The cluster default will be used if not set.
	  ##
	  # imagePullPolicy: ""
	
	  ## Defines number of concurrent requests for new job from GitLab
	  ## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runners-section
	  ##
	  # requestConcurrency: 1
	
	  ## Specify whether the runner should be locked to a specific project: true, false. Defaults to true.
	  ##
	  # locked: true
	
	  ## Specify the tags associated with the runner. Comma-separated list of tags.
	  ##
	  ## ref: https://docs.gitlab.com/ce/ci/runners/#using-tags
	  ##
	  tags: "kubernetes-helm"
	
	  ## Specify if jobs without tags should be run.
	  ## If not specified, Runner will default to true if no tags were specified. In other case it will
	  ## default to false.
	  ##
	  ## ref: https://docs.gitlab.com/ce/ci/runners/#allowing-runners-with-tags-to-pick-jobs-without-tags
	  ##
	  # runUntagged: true
	
	  ## Specify whether the runner should only run protected branches.
	  ## Defaults to False.
	  ##
	  ## ref: https://docs.gitlab.com/ee/ci/runners/#protected-runners
	  ##
	  # protected: true
	
	  ## Run all containers with the privileged flag enabled
	  ## This will allow the docker:dind image to run if you need to run Docker
	  ## commands. Please read the docs before turning this on:
	  ## ref: https://docs.gitlab.com/runner/executors/kubernetes.html#using-docker-dind
	  ##
	  privileged: false
	
	  ## The name of the secret containing runner-token and runner-registration-token
	  # secret: gitlab-runner
	
	  ## Namespace to run Kubernetes jobs in (defaults to the same namespace of this release)
	  ##
	  # namespace:
	
	  ## The amount of time, in seconds, that needs to pass before the runner will
	  ## timeout attempting to connect to the container it has just created.
	  ## ref: https://docs.gitlab.com/runner/executors/kubernetes.html
	  pollTimeout: 180
	
	  ## Set maximum build log size in kilobytes, by default set to 4096 (4MB)
	  ## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runners-section
	  outputLimit: 4096
	
	  ## Distributed runners caching
	  ## ref: https://gitlab.com/gitlab-org/gitlab-runner/blob/master/docs/configuration/autoscale.md#distributed-runners-caching
	  ##
	  ## If you want to use s3 based distributing caching:
	  ## First of all you need to uncomment General settings and S3 settings sections.
	  ##
	  ## Create a secret 's3access' containing 'accesskey' & 'secretkey'
	  ## ref: https://aws.amazon.com/blogs/security/wheres-my-secret-access-key/
	  ##
	  ## $ kubectl create secret generic s3access \
	  ##   --from-literal=accesskey="YourAccessKey" \
	  ##   --from-literal=secretkey="YourSecretKey"
	  ## ref: https://kubernetes.io/docs/concepts/configuration/secret/
	  ##
	  ## If you want to use gcs based distributing caching:
	  ## First of all you need to uncomment General settings and GCS settings sections.
	  ##
	  ## Access using credentials file:
	  ## Create a secret 'google-application-credentials' containing your application credentials file.
	  ## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runnerscachegcs-section
	  ## You could configure
	  ## $ kubectl create secret generic google-application-credentials \
	  ##   --from-file=gcs-applicaton-credentials-file=./path-to-your-google-application-credentials-file.json
	  ## ref: https://kubernetes.io/docs/concepts/configuration/secret/
	  ##
	  ## Access using access-id and private-key:
	  ## Create a secret 'gcsaccess' containing 'gcs-access-id' & 'gcs-private-key'.
	  ## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runners-cache-gcs-section
	  ## You could configure
	  ## $ kubectl create secret generic gcsaccess \
	  ##   --from-literal=gcs-access-id="YourAccessID" \
	  ##   --from-literal=gcs-private-key="YourPrivateKey"
	  ## ref: https://kubernetes.io/docs/concepts/configuration/secret/
	  cache: {}
	    ## General settings
	    # cacheType: s3
	    # cachePath: "gitlab_runner"
	    # cacheShared: true
	
	    ## S3 settings
	    # s3ServerAddress: s3.amazonaws.com
	    # s3BucketName:
	    # s3BucketLocation:
	    # s3CacheInsecure: false
	    # secretName: s3access
	
	    ## GCS settings
	    # gcsBucketName:
	    ## Use this line for access using access-id and private-key
	    # secretName: gcsaccess
	    ## Use this line for access using google-application-credentials file
	    # secretName: google-application-credentials
	
	  ## Build Container specific configuration
	  ##
	  builds: {}
	    # cpuLimit: 200m
	    # memoryLimit: 256Mi
	    # cpuRequests: 100m
	    # memoryRequests: 128Mi
	
	  ## Service Container specific configuration
	  ##
	  services: {}
	    # cpuLimit: 200m
	    # memoryLimit: 256Mi
	    # cpuRequests: 100m
	    # memoryRequests: 128Mi
	
	  ## Helper Container specific configuration
	  ##
	  helpers: {}
	    # cpuLimit: 200m
	    # memoryLimit: 256Mi
	    # cpuRequests: 100m
	    # memoryRequests: 128Mi
	    # image: gitlab/gitlab-runner-helper:x86_64-latest
	
	  ## Service Account to be used for runners
	  ##
	  # serviceAccountName:
	
	  ## If Gitlab is not reachable through $CI_SERVER_URL
	  ##
	  cloneUrl: http://139.24.120.248/
	
	  ## Specify node labels for CI job pods assignment
	  ## ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
	  ##
	  # nodeSelector: {}
	
	  ## Specify pod labels for CI job pods
	  ##
	  # podLabels: {}
	
	  ## Specify annotations for job pods, useful for annotations such as iam.amazonaws.com/role
	  # podAnnotations: {}
	
	  ## Configure environment variables that will be injected to the pods that are created while
	  ## the build is running. These variables are passed as parameters, i.e. `--env "NAME=VALUE"`,
	  ## to `gitlab-runner register` command.
	  ##
	  ## Note that `envVars` (see below) are only present in the runner pod, not the pods that are
	  ## created for each build.
	  ##
	  ## ref: https://docs.gitlab.com/runner/commands/#gitlab-runner-register
	  ##
	  # env:
	  #   NAME: VALUE
	
	
	## Configure securitycontext
	## ref: http://kubernetes.io/docs/user-guide/security-context/
	##
	securityContext:
	  fsGroup: 65533
	  runAsUser: 100
	
	
	## Configure resource requests and limits
	## ref: http://kubernetes.io/docs/user-guide/compute-resources/
	##
	resources: {}
	  # limits:
	  #   memory: 256Mi
	  #   cpu: 200m
	  # requests:
	  #   memory: 128Mi
	  #   cpu: 100m
	
	## Affinity for pod assignment
	## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
	##
	affinity: {}
	
	## Node labels for pod assignment
	## Ref: https://kubernetes.io/docs/user-guide/node-selection/
	##
	nodeSelector: {}
	  # Example: The gitlab runner manager should not run on spot instances so you can assign
	  # them to the regular worker nodes only.
	  # node-role.kubernetes.io/worker: "true"
	
	## List of node taints to tolerate (requires Kubernetes >= 1.6)
	## Ref: https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/
	##
	tolerations: []
	  # Example: Regular worker nodes may have a taint, thus you need to tolerate the taint
	  # when you assign the gitlab runner manager with nodeSelector or affinity to the nodes.
	  # - key: "node-role.kubernetes.io/worker"
	  #   operator: "Exists"
	
	## Configure environment variables that will be present when the registration command runs
	## This provides further control over the registration process and the config.toml file
	## ref: `gitlab-runner register --help`
	## ref: https://docs.gitlab.com/runner/configuration/advanced-configuration.html
	##
	# envVars:
	#   - name: RUNNER_EXECUTOR
	#     value: kubernetes
	
	## list of hosts and IPs that will be injected into the pod's hosts file
	hostAliases: 
	  # Example:
	  - ip: "139.24.120.248"
	    hostnames:
	    - "intranet.gitlabsz.siemens.com"
	  # - ip: "10.1.2.3"
	  #   hostnames:
	  #   - "foo.remote"
	  #   - "bar.remote"
	
	## Annotations to be added to manager pod
	##
	podAnnotations: {}
	  # Example:
	  # iam.amazonaws.com/role: <my_role_arn>
	
	## Labels to be added to manager pod
	##
	podLabels: {}
	  # Example:
	  # owner.team: <my_cool_team>
	
	## HPA support for custom metrics:
	## This section enables runners to autoscale based on defined custom metrics.
	## In order to use this functionality, Need to enable a custom metrics API server by
	## implementing "custom.metrics.k8s.io" using supported third party adapter
	## Example: https://github.com/directxman12/k8s-prometheus-adapter
	##
	#hpa: {}
	  # minReplicas: 1
	  # maxReplicas: 10
	  # metrics:
	  # - type: Pods
	  #   pods:
	  #     metricName: gitlab_runner_jobs
	  #     targetAverageValue: 400m
	
	```

2. 部署
	```
	helm install -f values.yml gitlab/gitlab-runner --generate-name
	```
3. 若无问题，一个高可用的gitlab-runner就成功运行了

#### 对比

gitlab-runner(shell excutor 手动部署）| gitlab-runner（kubernetes excutor helm部署）
--- | ---
需要手动提前注册好runner，以及手动注销 | 自动注册/注销runner
job运行在本身pod上 | job被分发到新pod上，新pod在job运行结束后，立即销毁
hpa支持比较麻烦，需要提前注册好一定数量runner | hap支持程度高
定制性高，随便用户修改 | 修改比较麻烦（但是目前提供的功能已经足够使用了，一般不需要用户进行底层修改）

#### 遗留问题

+ 官方推荐使用helm配置k8s gitlab-runner，但是网络似乎有问题
+ gitlab-runner的executor中有kubernetes，但是如何将ip和域名的对应在其启动前注入是个问题（内网环境下）