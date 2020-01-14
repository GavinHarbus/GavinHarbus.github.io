---
layout:     post
title:      K8S API Server
subtitle:   概念
date:       2020-1-14
author:     Gavin
header-img: img/post-bg-sky-trees.jpg
catalog: true
tags:
    - 云计算
---

> 人本过客来无处
> 
> 休说故里在何方

#### REST

REST(Representational State Transfer)， 表现层状态转换，是由Roy Thomas Fielding提出的一种网络程序架构风格。没有REST之前，我们的网络应用一般都是在服务端进行渲染，即服务端将数据注入网页模版（末班引擎或网页直接嵌入后端语言），生成HTML文件，交给客户端。此时，客户端的一条HTTP／HTTPS请求包含了数据和页面展示两个部分。打个比方，`http://localhost/pictures/pic001`就既请求了图片，又得到了图片的展示网页。这样的话，如果前端页面需要修改，往往也会涉及到后端的模版，很麻烦，因此提出了REST。
![](http://gavinmandias.online/large/006tNbRwgy1gavb8bxjrhj30jg0bwwg5.jpg)
REST不返回HTML，只返回数据，那么此时这个URL便是一个Web API，如`http://localhost/api/pictures/pic001`，只返回机器可读取的图片数据。通常使用JSON作为REST数据格式，因为JSON可以直接被JS使用。如此一来，如上图所示，各种前端、移动端便只专注于其页面的开发，后端便只关注数据、模型便可了。K8S的API Server便是基于REST设计的，因此前言提及REST概念。

#### API Server

K8S API Server的核心功能是提供了K8S各类资源对象（pod、service等）的增、删、改、查、Watch等REST接口。实际群内各个功能模块的数据交互和通信的中心枢纽，是整个系统的数据总线和数据中心。总之：  

+ 是集群管理的API入口
+ 是资源配额控制的入口
+ 提供完备集群安全机制

其通过一个叫api-server的进程提供服务，一般运行在master节点上。默认非安全端口8080，安全端口6443。
![](http://gavinmandias.online/large/006tNbRwgy1gaweoszi4cj30ib06zjs7.jpg)
如图所示，未得到验证的调用，得到403错误。

API Server最主要的REST接口是对资源的增删改查，但也提供一种特殊的REST接口——Proxy API接口，这类接口的作用是将REST请求代理到某个Node上的kubelet进程的REST端口上，由其负责响应。

集群功能模块之间的通信也是由API Server负责的。各功能模块通过API Server将信息存入etcd，要获取信息时，调用API Server即可。常见的场景时kubelet和API Server的交互，每隔一定时间，kubelet会报告自身状况，将节点信息更新到etcd中；同时也通过Watch接口获取pod状态，并将其应用到本节点上。

**PS：各功能模块为缓解API Server压力，会缓存资源对象信息到本地，这就是List-and-Watch机制**