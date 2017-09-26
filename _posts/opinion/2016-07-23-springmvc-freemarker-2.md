---
layout: post
title: 搭建SpringMVC-Freemarker-Tomcat环境 二
category: opinion
description: 搭建SpringMVC-Freemarker-Tomcat环境 二
---

##在Tomcat中配置Spring MVC+Freemarker

###1.添加SpringMVC以及Freemarker依赖
首先在pom.xml中添加相关依赖

	<dependencies>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
			<version>3.2.4.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-aop</artifactId>
			<version>3.2.4.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>3.2.4.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
			<version>3.2.4.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context-support</artifactId>
			<version>3.2.4.RELEASE</version>
		</dependency>

		<dependency>
			<groupId>org.freemarker</groupId>
			<artifactId>freemarker</artifactId>
			<version>2.3.20</version>
		</dependency>
	</dependencies>

不同Spring版本会存在配置上的差异，本文以Spring3.2.4和freemarker2.3.20为标准来进行项目构建
在pom.xml添加完依赖之后，等待依赖下载完毕。

###2.在web.xml配置SpringMVC Dispacher
接着在web.xml中添加如下信息:

	<welcome-file-list>
		<welcome-file>index.jsp</welcome-file>
	</welcome-file-list>
	
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath*:applicationContext-*.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/web/*</url-pattern>
    </servlet-mapping>

welcome-file-list标签标示的网站的默认页面

servlet-class中配置了org.springframework.web.servlet.DispatcherServlet，这是SpringMVC中给我们提供的一个默认的Dispatcher

servlet-mapping配置了URL路径和servlet的配对关系，在本项目中是将/web/*下的所有请求都交给DispatcherServlet处理

init-param中配置的Tomcat启动时去寻找spring配置文件的路径

###3.配置SpringMVC配置文件

在src/main/resources下面创建applicationContext-mvc.xml文件


    <beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:context="http://www.springframework.org/schema/context"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="
    	http://www.springframework.org/schema/beans     
    	http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
    	http://www.springframework.org/schema/context 
    	http://www.springframework.org/schema/context/spring-context-3.0.xsd">
	<context:component-scan base-package="com.huawei.demo" />

	<bean id="freemarkerConfig"
		class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
		<property name="templateLoaderPath" value="/view" />
		<property name="defaultEncoding" value="utf-8" />
		<property name="freemarkerSettings">
			<props>
				<prop key="template_update_delay">10</prop>
				<prop key="locale">zh_CN</prop>
				<prop key="datetime_format">yyyy-MM-dd</prop>
				<prop key="date_format">yyyy-MM-dd</prop>
				<prop key="number_format">#.##</prop>
			</props>
		</property>
	</bean>

	<bean
		class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter"></bean>
	<bean
		class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping">
		<property name="alwaysUseFullPath" value="true"></property>
	</bean>

	<bean id="viewResolver"
		class="org.springframework.web.servlet.view.freemarker.FreeMarkerViewResolver">

		<property name="viewClass"
			value="org.springframework.web.servlet.view.freemarker.FreeMarkerView"></property>
		<property name="suffix" value=".ftl" />
		<property name="contentType" value="text/html;charset=utf-8" />
		<property name="exposeRequestAttributes" value="true" />
		<property name="exposeSessionAttributes" value="true" />
		<property name="exposeSpringMacroHelpers" value="true" />
		</bean>
    </beans> 


接下来我们讲解一下里面标签的含义以及要关注的地方:

	<context:component-scan base-package="com.huawei.demo" />

这段表示Spring将会自动去com.huawei.demo包下面扫描java文件，如果扫描到有 @Component @Controller @Service等这些注解的类，则把这些类注册为bean

	<bean
		class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping">
		<property name="alwaysUseFullPath" value="true"></property>
	</bean>

这一段比较关键，其中我们将alwaysUseFullPath属性注册为true,这里解释一下这个值为true和false的区别:

我们在web.xml里面配置了我们的url-pattern为/web/*

假如我们SrpingMVC中有一个Controller:

	@Controller
	@RequestMapping("/web")
	public class Demo {

	@RequestMapping("/hello")
	public String Hello() {
		return "hello";
	 }
	}


如果alwaysUseFullPath为true，我们想访问我们这个controller,我们输入的URL应该是:
http://localhost:8080/{web-name}/web/hello
就可以正常访问我们这个Controller

但是如果alwaysUseFullPath为false,我们如果想访问的话，输入的URL应该是:http://localhost:8080/{web-name}/web/web/hello

可以看到true和false的区别就是SpringMVC框架是否用全路径去进行一个url-pattern的匹配

官方介绍 [SpringMVC](http://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/mvc.html)17.4章节

剩下的还有两个关于Freemarker配置的bean，这里就不详细叙述了，基本上可以看属性名就知道意义的了

###4.创建controller

在com.huawei.demo下面创建类：

	@Controller
	@RequestMapping("/web")
	public class Demo {

	@RequestMapping("/hello")
	public String Hello() {
		return "hello";
	 }
	}

###5.创建Freemarker页面

在src/main/webapp下面新建文件夹view，在view下面创建hello.ftl，在文本内直接输入Hello World

###6.启动Tomcat

启动Tomcat，访问

http://localhost:8080/{web-name}/web/hello

页面上现实hello.ftl的内容

至此SpringMVC+Freemarker服务端环境就配置完毕了