---
layout:     post
title:      Https配置
subtitle:   基于Nginx与Letsencrypt
date:       2019-1-4
author:     Gavin
header-img: img/post-bg-ioses.jpg
catalog: true
tags:
    - Nginx
    - Https
    - SSL
---

> 莫失莫忘
> 
> 仙寿恒昌

# 前言

最近项目推进，Boss要求为了跟上时代潮流，将组里所有Http项目全部升级为Https项目，因此学习并实践了这方面的内容，同时做了一个记录。

---

# 介绍

#### HTTP与HTTPS

HTTP（超文本传送协议）定义了浏览器怎样向服务器请求资源，以及服务器如何将资源传送给服务器。HTTP是**面向事务**的**应用层**协议，它是网络上可靠交换文件的基础。HTTP使用了**面向连接**的**TCP**作为运输层协议，保证了数据的可靠传输，因此HTTP不必考虑丢失重传的问题（*注：Http协议本身是无连接、无状态的*）。

![](https://ws2.sinaimg.cn/large/006tNc79ly1fyukhxflc0j30ch08w765.jpg)

HTTPS（提供安全服务的HTTP协议）则确保了（1）用户请求的服务器属于真正的服务商（2）报文内容在传输过程中没有被更改（3）传输过程中敏感信息不被窃听。要保证以上安全服务，需要使用运输层的安全协议，现在广泛使用的有如下两个： 

* **安全套接字层SSL（Secure Socket Layer）**
* **运输层安全TLS（Transport Layer Security）**

SSL协议作用在端系统应用层的HTTP和运输层之间，在TCP之上建立一个安全通道，为通过TCP传输的应用层数据提供安全保障。之后，IETF在SSL 3.0的基础上对其进行了标准化，设计了TLS协议，为所有基于TCP的网络应用提供安全数据传输服务。（*注：SSL应该是运输层协议，然而实际上，需要使用安全运输的应用程序（如HTTP）却把SSL驻留在应用层，因而应用层扩大了*）

![](https://ws3.sinaimg.cn/large/006tNc79ly1fyul65innvj30d706omy5.jpg)

应用层使用SSL最多的就是HTTP，但SSL并非仅用于HTTP，而是可用于**任何**应用层的协议。HTTP调用SSL时，对整个网页进行加密。这时，在发送方，SSL从SSL套接字接收应用层的数据（如HTTP报文或IMAP报文），对数据进行加密，然后把加密的数据送往TCP套接字；在接收方，SSL从TCP套接字读取数据，解密后，通过SSL套接字把数据交给应用层。

SSL提供的安全服务可归纳为以下三种： 
 
1. SSL服务器鉴别，允许用户鉴别服务器身份。支持SSL的客户端通过验证来自服务器的证书，来鉴别服务器的真实身份并获得服务器的公钥
2. SSL客户鉴别，SSL的可选安全服务，允许服务器证实客户的身份
3. 加密的SSL会话，对客户和服务器之间发送的所用报文进行加密，并检测报文是否被篡改

![](https://ws1.sinaimg.cn/large/006tNc79ly1fyuljdd597j30dx099abr.jpg)

#### Let's Encrypt

**[Let's Encrypt](https://letsencrypt.org/)**作为一个公共且免费SSL的项目逐渐被广大用户传播和使用，是由Mozilla、Cisco、Akamai、IdenTrust、EFF等组织人员发起，主要的目的也是为了推进网站从HTTP向HTTPS过渡的进程，目前已经有越来越多的商家加入和赞助支持。

![](https://ws4.sinaimg.cn/large/006tNc79ly1fyultiysl7j30w30d611a.jpg)

---

# 过程

#### 1.安装 Let’s Encrypt 客户端

```
yum install git python#安装git
git clone https://github.com/letsencrypt/letsencrypt#克隆仓库到本地
```

#### 2.验证安装是否成功

使用以下命令运行一次客户端，将自动检查更新并升级（letsencrypt启动后，总是会自动检查更新并升级，除非使用--no-self-upgrade参数显示指定），如果一切正常（事实上，升级后letsencrypt在某些系统、某些云服务商的机器上常常不能正常运行，因为涉及到各种源，版本依赖等问题），将会显示完整的帮助文档。

```
cd letsencrypt
./letsencrypt-auto --help all
```

#### 3.验证域名所有权并获取证书

认证插件通过certonly命令启用，认证功能用于确认你是域名的所有者，并为你的域名获取证书，证书被放置在你的域名所在服务器的/etc/letsencrypt/live/[domain]目录。如果你一次性对多个域名进行认证，则这些域名将共用一个证书文件。

```
./letsencrypt-auto certonly
```
正常情况下，进入交互式界面，提示你输入邮箱（在证书失效前收到通知邮件），并同意官方协议

```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator apache, Installer apache
Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel): ********

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(A)gree/(C)ancel: A

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about our work
encrypting the web, EFF news, campaigns, and ways to support digital freedom.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y
```
验证域名，选择方式3，文件验证

```
Requesting to rerun ./letsencrypt-auto with root privileges...
[sudo] password for zfy:
Saving debug log to /var/log/letsencrypt/letsencrypt.log

How would you like to authenticate with the ACME CA?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: Apache Web Server plugin (apache)
2: Spin up a temporary webserver (standalone)
3: Place files in webroot directory (webroot)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-3] then [enter] (press 'c' to cancel): 3
Plugins selected: Authenticator webroot, Installer None
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): lilab.jysw.suda.edu.cn
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for lilab.jysw.suda.edu.cn
Input the webroot for lilab.jysw.suda.edu.cn: (Enter 'c' to cancel): /home/web/public/htdocs
Waiting for verification...
Cleaning up challenges

```
验证成功，获得证书

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/lilab.jysw.suda.edu.cn/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/lilab.jysw.suda.edu.cn/privkey.pem
   Your cert will expire on 2019-04-04. To obtain a new or tweaked
   version of this certificate in the future, simply run
   letsencrypt-auto again. To non-interactively renew *all* of your
   certificates, run "letsencrypt-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

#### 4.安装证书（基于nginx）

```
./letsencrypt-auto install --nginx --nginx-server-root <nginx conf path> --nginx-ctl <nginx binary path>
```
证书生成成功后，会让你选择是否将所有的 HTTP 请求重定向到 HTTPS（输入 1 或者 2）。如果选 1，则通过 HTTP 和 HTTPS 都可以访问。如果选 2，则所有通过 HTTP 来的请求，都会被 301 重定向到 HTTPS。

```
Requesting to rerun ./letsencrypt-auto with root privileges...
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator None, Installer nginx

Which certificate would you like to install?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: lilab.jysw.suda.edu.cn
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press 1 [enter] to confirm the selection (press 'c' to cancel): 1
Deploying Certificate to VirtualHost /usr/local/nginx/conf/nginx.conf

Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 1
```
nginx配置文件如下：

```
server {
    listen       443 ssl;
    server_name  localhost;

    ssl_certificate /etc/letsencrypt/live/lilab.jysw.suda.edu.cn/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/lilab.jysw.suda.edu.cn/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    location / {
        root   /home/web/htdocs;
        index  index.html index.htm;
    }
}
```
安装成功
![](https://ws3.sinaimg.cn/large/006tNc79ly1fyumvzequaj30dh07ljsb.jpg)

#### 5.证书管理

1. 查看letsencrypt在当前服务器获取的证书

```
./letsencrypt-auto certificates
```
返回：

```
Found the following certs:
  Certificate Name: lilab.jysw.suda.edu.cn
    Domains: lilab.jysw.suda.edu.cn
    Expiry Date: 2019-04-04 00:39:44+00:00 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/lilab.jysw.suda.edu.cn/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/lilab.jysw.suda.edu.cn/privkey.pem
```

2. 基本操作  
通过--cert-name来指定证书的名称，并对证书执行操作，可选的操作有run、certonly、certificates、renew、delete

```
./letsencrypt-auto certonly --cert-name <name> [operate]
```
**run**：获取和安装证书  
**certonly**：获取证书  
**certificates**：查看和--cert-name指定的名称匹配的证书信息  
**renew**：更新快要过期的证书  
**delete**：删除证书

3. 更新证书  
证书的更新命令是renew，renew命令会在本机找出所有的证书，并检查证书的过期时间，它只会对有效期不足30天的证书执行更新。如果证书不需要更新，它不会和letsencrypt服务器产生通信，因此，renew命令可以频繁地执行而不会受到letsencrypt服务器的连接次数限制的影响。也是基于这一特点，可以在crontab设置定期任务，频繁地执行renew操作，确保证书不会过期。

```
./letsencrypt-auto renew
```
设置定时任务

```
crontab -e

0 3 * * * ./letsencrypt-auto renew#在每天凌晨3点运行。该命令将检查服务器上的证书是否将在未来30天内过期，如果是，则进行更新
```

---

# 资料
1. *《计算机网络》——谢希仁* 
2. **[Nginx 实现 HTTPS（基于 Let's Encrypt 的免费证书）](https://blog.csdn.net/kikajack/article/details/79122701)** 
3. **[https 证书工具 Letsencrypt 简单教程](https://blog.csdn.net/dancen/article/details/81311688)** 

