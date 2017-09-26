---
layout: post
title: 搭建SpringMVC-Freemarker-Tomcat环境 一
category: opinion
description: 搭建SpringMVC-Freemarker-Tomcat环境 一
---

由于最近要帮忙开发一个简单的Portal，目前考虑用SpringMVC+Freemarker+Tomcat来搭建一个服务器环境

##环境准备:
1.Maven [Maven](http://maven.apache.org)

2.Tomcat [Tomcat](http://tomcat.apache.org/index.html)

##使用Eclipse创建Web项目
这里可以直接用maven的webapp模板来创建一个项目，不过博主不太喜欢这种方式，这里介绍另外一种方式:

###1.新建Maven Project
![Create-1](http://www.liangye.info/images/springmvc/create-1.png)
![Create-2](http://www.liangye.info/images/springmvc/create-2.png)

这里然后点击next，然后根据自己情况写入Group Id和Artifact Id，Packaging选择war 然后点击Finish即可
	
###2.将Maven Project转化成Maven Web Project
右键项目-Properties-Project Facets，如下图所示，勾选红框内选项，保存即可

![Create-3](http://www.liangye.info/images/springmvc/create-3.png)

可以看到项目结构如下：

![Create-4](http://www.liangye.info/images/springmvc/create-4.png)

多出了WebContent内容，WebContent就是在Tomcat中部署项目时候的根目录，接着我们更改一下部署路径


右键项目-new-Source Floder，创建新的源码文件夹，文件夹名我这里取src/main/webapp

右键项目-Properties-Deployment Assembly

![Create-5](http://www.liangye.info/images/springmvc/create-5.png)

按图片修改部署路径

然后在src/main/webapp下创建文件夹WEB-INF，并且在WEB-INF下面创建web.xml,web.xml内容如下:
	
	
	<?xml version="1.0" encoding="UTF-8"?>
        <web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="http://java.sun.com/xml/ns/javaee 
		http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">

		<welcome-file-list>
			<welcome-file>index.jsp</welcome-file>
	    </welcome-file-list>
    </web-app> 
    
至此Web Project的夹子基本是构成了,结构如下图：

![Create-6](http://www.liangye.info/images/springmvc/create-6.png)
	
	
##在Tomcat上面运行项目
###1.添加默认页面,在WEB-INF下面创建index.jsp,内容如下:
	
	<html>
		<body>
				Hello
		</body>
	</html>
	
	
如果在添加完index.jsp发现项目报错：The superclass "javax.servlet.http.HttpServlet" was not found on the Java Build Path 原因就是当前项目没有加入servlet相关的类库，因为jsp本质上其实就是一个servlet。

可以通过maven来添加依赖，也可以直接引用tomcat的运行时类库，这里选择直接引用tomcat运行时类库:

![Create-7](http://www.liangye.info/images/springmvc/create-7.png)

![Create-8](http://www.liangye.info/images/springmvc/create-8.png)
	
最后通过Tomcat去运行项目，这里就不详细说怎么运行了，不知道的童鞋请去google一下吧。

然后在IE的地址栏中输入http://localhost:8080/{项目名称}/ 出现Hello World页面的话，基本的Tomcat运行环境已经配置完毕
	
	