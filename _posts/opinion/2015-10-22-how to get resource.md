---
layout: post
title: 如何在Java下正确而获取资源文件
category: opinion
description: 初探获取Java项目的资源文件
---

相信很多Java的初学者在工作或者学习中都会遇到要加载Java项目中的资源，或者某Jar包下资源的情况，笔者也是常常遇到这一类的问题，在以前遇到的时候总是糊里糊涂的一顿乱搞先糊弄过去，没有去仔细分别几种常用的获取资源的方法的区别，今天又遇到类似的问题，着实让我恼火了很久，于是决心先初探一下相关的方法，本文不涉及JDK源码，只是先探讨几种可用的方法。

这里先探讨4种获取资源文件的方法：</br>
    1.通过File类进行相关操作</br>
    2.通过Class的getResource()方法</br>
    3.通过ClassLoader的getResource()方法</br>
    4.通过SystemClassLoader的getResource()方法</br>

在这里，我们先看看第一种情况，我们直接在Java Project中引用一些Java资源文件,每种方法均有两种情况，路径以/开头和路径不以/开头
    项目的基本结构如下:</br>
    src</br>
    |___com.csu.ly.test</br>
    |___A.txt</br>

	public static void fileGet() throws IOException {
		
		File file = new File("");
		File fileWithSlash = new File("/");
		
		System.out.println("file location : " + file.getCanonicalPath());
		System.out.println("fileWith location : " + fileWithSlash.getCanonicalPath());
	}

	public static void classLoaderGetResource() {
		try {
			
			URL url = Test.class.getClassLoader().getResource("");
			URL urlWithSlash = Test.class.getClassLoader().getResource("/");

			System.out.println("classloader url : " + url.getPath());
			System.out.println("classloader urlWithSlash : " + urlWithSlash.getPath());
			
		} catch (Exception e) {
			// TODO: handle exception
			System.out.println("classloader urlWithSlash fail ");

		}

	}

	public static void classGetResource() {
		try {
			
			URL url = Test.class.getResource("");
			URL urlWithSlash = Test.class.getResource("/");

			System.out.println("class url : " + url.getPath());
			System.out.println("class urlWithSlash : " + urlWithSlash.getPath());
			
		} catch (Exception e) {
			// TODO: handle exception
			System.out.println("class urlWithSlash fail ");
		}
	}

	public static void systemLoaderGetResource() {
		try {
			
			URL url = ClassLoader.getSystemClassLoader().getResource("");
			URL urlWithSlash = ClassLoader.getSystemClassLoader().getResource("/");

			System.out.println("systemLoader url : " + url.getPath());
			System.out.println("systemLoader urlWithSlash : " + urlWithSlash.getPath());
			
		} catch (Exception e) {
			// TODO: handle exception
			System.out.println("systemLoader urlWithSlash fail ");
		}
	}

输出结果如下：</br>
    注释:C:\Users\Administrator\workspace\classloader 为Java Project的根目录</br>

    file location : C:\Users\Administrator\workspace\classloader
	fileWith location : C:\
	class url : /C:/Users/Administrator/workspace/classloader/bin/com/csu/ly/test/
	class urlWithSlash : /C:/Users/Administrator/workspace/classloader/bin/
	classloader url : /C:/Users/Administrator/workspace/classloader/bin/
	classloader urlWithSlash fail 
	systemLoader url : /C:/Users/Administrator/workspace/classloader/bin/
	systemLoader urlWithSlash fail 


接着，我们先看第二种情况，就是我们在自己的Java Project中，想获取某Jar包下的资源文件，这种情况可能更加的常见
    我们新建一个项目，修改一下4个方法，在getResource()方法的参数加上A.txt,然后export出来成为jar包，导入第一种情况的Jar包，然后调用4个方法

得到的结果如下：</br>

	file location : C:\Users\Administrator\workspace\classLoaderB\A.txt
	fileWith location : C:\A.txt
	class urlWithSlash fail 
	classloader url : file:/C:/Users/Administrator/Desktop/getresource.jar!/A.txt
	classloader urlWithSlash fail 
	systemLoader url : file:/C:/Users/Administrator/Desktop/getresource.jar!/A.txt
	systemLoader urlWithSlash fail 

得到两种情况的结果之后，按照我的理解来分析一下规律：</br>
    1.以File方式获取资源，其实是与文件在操作系统上面的路径来定位的，笔者在Windows系统上面运行，当不加/的时候，路径是相对于JavaProject的根目录的，加了之后则是相对于当前盘符的位置，这种方法感觉比较适合资源在操作系统一个相对固定而且是位置是可知的文件资源。</br>
    2.以Class.getResource()方式获取资源，当不加/的时候,路径是相对于Class文件所在目录的，当加了/之后，路径是相对当前JavaProject或者Jar包的ClassPath的。</br>
    3.ClassLoader和SystemClassLoader方法获取资源，不加/路径都是相对JavaProject或者Jar包的ClassPath的，加了/之后会抛出运行时异常，暂时原因我还没去深究，留待日后解决。</br>

这两种情况下获取资源的最大区别就是，Jar包本身就是一个文件，在我们的Java程序中，并不能够通过类似 *.jar/src/A.txt的文件路径方式来获取资源，我们最好是通过ClassPath为相对路径来获取我们想要的资源。细心的读者可能已经发现了在第二种情况下ClassLoader获取资源文件的路径中是带有！号，因为我们Jar本身就是一个文件的原因。

本文并没有涉及JDK的大量源码来探讨，只是简单的从使用的层面上面去认识一下如何读取资源，有不足或者错误的地方希望各位大牛指点。

本次探讨暂时到这里了，以笔者粗浅的理解去简单的认识一下加载资源的各种方式，据我了解在Web项目中来获取资源文件。又有所不同，但是目前还没时间去写出一些代码实例来验证。










