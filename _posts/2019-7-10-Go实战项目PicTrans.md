---
layout:     post
title:      Go项目实战PicTrans
subtitle:   图片风格转换
date:       2019-7-10
author:     Gavin
header-img: img/post-bg-js-version.jpg
catalog: true
tags:
    - Web
    - Go
    - Iris
---

> 曲终人不见
> 
> 江上数峰青

# 前言

#### 介绍

之前学习了Go和其框架Iris的部分内容，便想练练手，毕竟实践是最好的学习方式。因此基于本科的兴趣制作了在线一个图像风格转换工具[***PicTrans***](http://45.32.68.50/PicTrans)。
![](http://45.32.68.50/large/006tNc79ly1g4ut49d59wj30rk0gpwnd.jpg)

---

# 准备工作

#### Go环境配置

服务器环境为Cent OS 7，首先安装Go

1. 下载Go分发包

	```
	wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
	```
2. 解压
	
	```
	tar -C /usr/local -xzf go1.12.7.linux-amd64.tar.gz
	```
3. 导入路径
	
	```
	export PATH=$PATH:/usr/local/go/bin
	```  
4. 测试是否安装成功

	```
	package main

	import "fmt"
	
	func main() {
		fmt.Printf("hello, world\n")
	}
	```
	
	```
	go run test.go
	```  
	得到如下回显：  
	![](http://45.32.68.50/large/006tNc79ly1g4upda7refj306z0143yj.jpg)  
	安装成功！！  
5. Iris框架安装
	
	```
	go get -u github.com/kataras/iris
	```  
	
#### Python依赖安装

	```
	pip install Pillow
	```  
	
---

# 核心功能实现

#### 图片风格转换

此功能由python实现，是我本科时偶然兴趣写的一个图片风格转换小工具，今天练手，将其嵌入Go Web后台中。  

```
cmd := exec.Command("./pictrans.py", "--input", filename)
_, err = cmd.Output()
```  

利用Go运行此工具，此工具为与Go Web后端磨合，我做了一些修改，使其内部路径适应了Go的需要，其中--input后跟用户上传的图片编号，完整代码如下：  

```
from PIL import Image
import argparse as apa

def cleanPic(pic_name):
    try:
        img = Image.open("./static/pics/"+pic_name+".jpg")
        out = img.point(lambda x:255 if x>75 else 0) 
        out.convert('L').save("./static/pics/res"+pic_name+".jpg")
        print("true")
    except Exception:
        print("false")

if __name__ == '__main__':
    parser = apa.ArgumentParser(prog="convert")
    parser.add_argument("--input", required=True, help="The xml file path",type=str)
    args = parser.parse_args()
    cleanPic(args.input)
```  

#### Go Web控制逻辑和API实现

利用了Iris框架实现这里的需要，我只有一个静态模版网页，但实现了一个上传图像并转换其风格的接口，利用模版引擎完成前后端数据传递。  

```
app.Post("upload", func (ctx iris.Context) {
		file, _, err := ctx.FormFile("upload")
		if err != nil {
			ctx.StatusCode(iris.StatusInternalServerError)
			ctx.ViewData("message","Upload failed!")
			ctx.View("index.html")
			return
		}

		defer file.Close()

		//filename := info.Filename
		filename :=	time.Now().Format("20060102150405")

		out, err := os.OpenFile("./static/pics/"+string(filename)+".jpg", os.O_WRONLY|os.O_CREATE, 0666)
		if err != nil {
			ctx.StatusCode(iris.StatusInternalServerError)
			ctx.ViewData("message","Save failed!")
			ctx.View("index.html")
			return
		}
		defer out.Close()

		io.Copy(out,file)

		cmd := exec.Command("./pictrans.py", "--input", filename)
		_, err = cmd.Output()

		if err != nil {
			ctx.ViewData("message","Transform failed!")
			ctx.View("index.html")
			return
		}
		ctx.ViewData("filename",filename+".jpg")
		ctx.ViewData("rawpath","/static/pics/"+filename+".jpg")
		ctx.ViewData("respath","/static/pics/res"+filename+".jpg")
		ctx.ViewData("message","Upload and transform success!")
		ctx.View("index.html")
	})
```  

---

# 部署

#### Nginx反向代理Go Web

修改Nginx配置，添加虚拟目录／PicTrans，注意端口号一致  

```
location /PicTrans {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://localhost:9001;
    }
```  

#### 启动Go进程

```
go run picTransController.go
```  

![](http://45.32.68.50/large/006tNc79ly1g4uv1ypqikj30q30gq7cy.jpg)  

可以工作！！  

#### 部署Go Web

1. 编译

	```
	go build picTransController.go
	```  
	得到可执行文件picTransController
2. 执行
	
	```
	nohup ./picTransController > log &
	```  
	
---

# 总结

#### 问题

1. 注意开发时和部署时的路径问题
2. 部署后，静态资源交由Nginx转发，因此要尤其注意静态资源的路径问题
3. 后台运行Go Web进程的方式比较暴力，由于我是初学，还未找到优雅的部署方式
4. 有可能会遇到Nginx 413 Request Entity Too Large的问题
	
	这是请求太大，超过上限了，解决如下：  
	
	```
	server {  
	    ...  
	    client_max_body_size 20m;  
	    ...  
	}
	```

#### 资源

* [**Promosis**](https://github.com/promosis/file-upload-with-preview)用于前端上传并预览图片


