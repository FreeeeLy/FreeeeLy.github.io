---
layout: post
title: Map-Reduce
description: Map-Reduce
category: blog
---

## 为什么选择系统地的学习分布式计算的课程 ##
工作也快有两年了，也是在从事分布式相关的中间件开发工作，本身我也不是科班出身，以往在工作中都是磕磕碰碰的走过来，哪怕是遇到很简单的概念性的术语，也得Google一下，随着工作的发展，自己也渐渐的感到基础的重要性，现在IT的发展日新月异，各种新的技术，新的软件层出不穷，其中归根到底还是一些基本理论的工程化实现，所以重新学基础的理论，其实对于自己去接受新的事务，理解新的事物也有明显的帮助，可以更快的如庖丁解牛般的去学习新的技术。最近选择了学习一边Mit的6.824的课程，选课这门课程有两点吧，一个是因为推荐率很高（笑），看了下内容确实也挺适合，第二个就是有相应的Lab，而且是用Go语言编写的，最近我对Go语言也比较有兴趣，所以就挑了这个课程

## 关于Map-Reduce ##
关于Map-Reduce的概念不想在这里粘贴太多了，有一定英语基础的可以阅读英语的论文：[论文原文](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf "mapreduce-paper")，简单的了解一下的话就看下百科:[Wiki](https://zh.wikipedia.org/wiki/MapReduce "Wiki")

Map任务:根据用户自定义的Map函数，将输入信息进行分类

Reduce任务:根据用户自定义的Reduce函数，将Map任务生成的中间结果进行一次归并

这里简单的介绍一个完整的MapReduce任务的流程:

1. 将原始的任务输入文件分解为K个固定大小的小文件
2. Master接收到任务，会发起M个Map任务和和R个Reduce任务
3. 每个接收到Map任务的Worker去读取步骤1中的若干个文件(读取哪些子文件由master去分配)，并且将结果按照一定的规则分别持久化到本地的R个文件中
4. Worker向Master上报生成的中间态的文件的地址
5. Reduce任务的Worker从Master中了解到Worker生成的中间文件的地址，然后开始远程读取Map生成的中间文件，并且使用用户定义的Reduce函数去处理数据，并且把结果进行持久化输出
6. Master等待所有Map和Reduce任务完成，向客户端返回结果

Map-Reduce主要是屏蔽了底层复杂的实现，向用户提供了一个友好的大规模的数据处理手段

## Lab实验小记 ##
学习这门课程，一定要做相应的Lab实验，才能够更好的理解Map-Reduce

Part I，比较简单，只需要写一下doMap和doReduce两个函数，简单的用json编码一下输入输出结果即可，在doRecude中有一点注意的是，要将读取到的文件里面的key先进行一下排序，然后再传给Reduce函数处理,否则在遇到大文件处理的时候很容易导致内存溢出

Part II，写个word count的map和reduce函数，没有要特别注意的点

Part III，使用RPC来模拟分布式环境下的Map-Reduce任务，这里需要用到Go语言的并发编程，没有基础的需要好好的看一下，比较有意思的是Go语言的channel机制，使用类似信号量的方式来控制共享变量的访问

Part IV，在Part III的基础上加入错误处理，没有特别注意的地方

Part V，inverted index的一个小任务，建议做一下，可以顺便了解文档索引的基础

做完Lab对课程有更加深刻的理解，同时学到Go的channel机制，对于指导我们使用其他语言进行并发控制也有很帮助，使用信号而不是锁来控制共享资源，这在Cassandra和ElasticSearch的编程里面都很常见



[Edward]:    http://Edward0205.github.io  "Edward"
