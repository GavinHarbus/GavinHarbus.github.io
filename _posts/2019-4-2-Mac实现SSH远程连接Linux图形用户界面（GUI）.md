---
layout:     post
title:      SSH远程连接Linux图形用户界面（GUI）
subtitle:   macOS Sierra系统
date:       2019-4-2
author:     Gavin
header-img: img/post-bg-debug.jpg
catalog: true
tags:
    - Linux
    - macOS
    - SSH
---

> 来如春梦几多时
> 
> 去似朝云无觅处

# 前言

Windows端通常使用 [**XShell**](https://xshell.en.softonic.com/) 和 [**Xmanager**](http://www.xshellcn.com/) 来直接调用Linux服务器上软件的GUI界面，那么Mac下该怎么做呢？经过调研，也很简单，[**XQuartz**](https://www.xquartz.org/) + 自带Terminal就可以实现啦。

---

# 介绍

![](https://ws3.sinaimg.cn/large/006tKfTcly1g1oiu46p77j30sp0ergoh.jpg)  

XQuartz是运行在OS X下的 **X Window System**，是一种计算机软件系统和网络协议，提供了一个基础的图形用户界面（GUI）和丰富的输入设备能力联网计算机。其中软件编写使用广义的命令集，它创建了一个硬件抽象层，允许设备独立性和重用方案的任何计算机上实现。  

---

# 步骤

#### 编辑/etc/ssh/sshd_config配置文件

```
X11Forwarding yes
X11DisplayOffset 10
```
配置转发参数为yes,重启ssh 服务  

```
service sshd restart
```

#### 下载安装XQuartz

![](https://ws3.sinaimg.cn/large/006tKfTcly1g1oizxg15ej30vw0fn7hq.jpg)  

#### Mac OS系统下修改ssh配置

```
vim ~/.ssh/config
```
加入如下参数

```
Host *
	ForwardX11Trusted yes
	ForwardX11 yes
	XAuthLocation /opt/X11/bin/xauth
```

打开XQuartz
![](https://ws2.sinaimg.cn/large/006tKfTcly1g1oj2s8ao8j30dk09idg0.jpg)  

#### 连接服务器  

```
ssh -X user@ip
```

启动程序
![](https://ws2.sinaimg.cn/large/006tKfTcly1g1oj5iqavxj30in0jyt9j.jpg)  
成功！！

---

# 后记

可能会遇到 **XRequest.149: BadMatch (invalid parameter attributes) 0xa00105** 的错误，解决方法：  

1. 退出XQuartz  
2. 终端中输入如下

	```
	defaults write org.macosforge.xquartz.X11 enable_iglx -bool true 
	```
3. 重启XQuartz 