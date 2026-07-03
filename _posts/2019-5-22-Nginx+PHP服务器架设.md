---
layout:     post
title:      Nginx+PHP服务器架设
subtitle:   基于Cent OS 7
date:       2019-5-22
author:     Gavin Schnee
header-img: img/post-bg-e2e-ux.jpg
catalog: true
tags:
    - Nginx
    - Web
---

> 岁暮天寒
> 
> 彤云酿雪

# 前言

因为个人比较熟悉Python、Django，所以想在自己的服务器上架设相关服务，但是原有的服务是基于Apache、PHP的，因此想到用Nginx替代Apache。

---

# 介绍

![](https://crescentrose.blob.core.windows.net/blog-img/006tNc79ly1g3a1wm30fmj30he040mx8.jpg)  

Nginx与Apache一样，是web服务器程序。Nginx较之Apache是一个高性能、高并发、低消耗的Http和反向代理服务器，同时也提供了IMAP/POP3/SMTP服务。

---

# 流程

#### Nginx

1. 安装Nginx

	```
	sudo yum install nginx
	```
2. 检查Nginx状态
	
	```
	service nginx status
	```  
	![](https://crescentrose.blob.core.windows.net/blog-img/006tNc79ly1g3a2czr8e7j30ko026mxk.jpg)  
	此时尚未激活
3. 关闭Apache，释放80端口

	```
	sudo service httpd stop
	```
4. 启动Nginx服务
	
	```
	sudo service nginx start
	```  
	![](https://crescentrose.blob.core.windows.net/blog-img/006tNc79ly1g3a2gbb3m0j30za0bggnc.jpg)
	访问服务器看到已经可以使用了  
	默认文件路径：  
	* 网页根目录：/usr/share/nginx/html
	* 配置文件目录：/etc/nginx/nginx.conf  
5. 完成，等待接下来操作  

#### PHP

1. 安装php、php\-mysql和php\-fpm
	
	```
	sudo yum install php php-mysql php-fpm
	```
2. 检查php\-fpm状态
	
	```
	service php-fpm status
	```  
	![](https://crescentrose.blob.core.windows.net/blog-img/006tNc79ly1g3a2mhr0k6j30l0021gm1.jpg)  
	如同之前Nginx一样，此时尚未激活  
3. 完成，等待配置  

#### 配置Nginx+PHP

1. 配置PHP处理器  
	
	```
	sudo vi /etc/php.ini
	```  
	打开配置文件，我们寻找cgi.fix_pathinfo参数，其默认是被注释且值为1的，这很不安全，它可以允许PHP在不完全匹配的情况下执行脚本，因此我们将其设置为0。  
	
	```
	cgi.fix_pathinfo=0
	```
2. 配置php\-fpm
	
	```
	sudo vi /etc/php-fpm.d/www.conf
	```
	修改如下配置：  
	
	```
	listen = /var/run/php-fpm/php-fpm.sock
	user = nginx
	group = nginx
	```
3. 启动php\-fpm
	
	```
	sudo service php-fpm start
	```  
	![](https://crescentrose.blob.core.windows.net/blog-img/006tNc79ly1g3a347rivzj30l106ign3.jpg)  
	已启动
4. 配置Nginx以处理PHP页面
	
	```
	sudo vim /etc/nginx/conf.d/default.conf
	```  
	粘贴一下配置  
	
	```
	server {
	    listen       80;
	    server_name  server_domain_name_or_IP;
	
	    # note that these lines are originally from the "location /" block
	    root   /usr/share/nginx/html;
	    index index.php index.html index.htm;
	
	    location / {
	        try_files $uri $uri/ =404;
	    }
	    error_page 404 /404.html;
	    error_page 500 502 503 504 /50x.html;
	    location = /50x.html {
	        root /usr/share/nginx/html;
	    }
	
	    location ~ \.php$ {
	        try_files $uri =404;
	        fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
	        fastcgi_index index.php;
	        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	        include fastcgi_params;
	    }
	}
	```
5. 重启Nginx并访问原来的网站
	![](https://crescentrose.blob.core.windows.net/blog-img/006tNc79ly1g3a3jkgwrtj30yz0gq1bs.jpg)  
	成功！！！
6. 配置Nginx和php\-fpm自启动

	```
	sudo systemctl enable nginx
	sudo systemctl enable php-fpm
	```  

---

# 问题

测试时发现某些网页出现405错误，发现是由于后端要求的请求数据的方法和前端不匹配造成的，修改后就👌啦。
