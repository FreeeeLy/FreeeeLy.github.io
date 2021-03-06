---
layout: post
title: 关于性能测试的一些总结
description: 工作以来对性能测试方面的总结
category: blog
---

工作已经满一年了，这一年的主要工作是负责云服务存储中间件的一些开发，由于中间件是平台的基础服务最为重要环节之一，如何去不断的提高中间件的性能也是我日常最重要的工作内容之一，这里就简单的总结一下这一年下来的心得吧。

一.关于性能指标
======

关于性能的指标有很多，在我的工作中我一般主要关注以下几个：

**1.系统吞吐量TPS（OPS）**

系统吞吐量TPS简单的来说就是系统每秒钟能处理的请求数量

**2.系统能支持的并发线程数**

系统能够同时支撑的最大用户数量，或者说应用处理请求的工作线程数

**3.平均调用时延**

处理请求平均时间

二.做性能测试的一般思路
=======

**1.明确测试场景，采用单变量的方式去设计测试方案，并且选择合适的压测工具**

**2.启动压测程序，记录当前系统的性能情况，并且分析系统是否已经到达瓶颈**

**3.改变系统参数，重新进行压测**

三.如何观察系统是否到达瓶颈了
=======

一般来说系统的瓶颈有两个方面：

**1.应用程序之外的因素：**

机器的硬件性能有限，主要体现在CPU，内存，磁盘，带宽等资源不足以满足当前的请求量。

操作系统上的限制，比如linux操作系统一般会限制每个用户所能够使用的内存大小，能够使用的CPU时间长短，能够打开文件句柄数量等等，在linux下面可以通过ulimit命令来查看相关限制

**2.应用程序本身的因素**

应用程序本身设计及编码的问题,导致不能够充分利用好机器的资源，比如说不合理的使用锁和同步，逻辑处理代码冗杂，API使用方式不合理等等问题。这样的问题往往会导致在机器负载很低的情况，系统吞吐量却上不去情况

如果是用Java语言编写的代码，在系统吞吐量到一定的情况之后，很容易会出现GC问题，如果Java的GC出现了问题，对外的具体表现一般是请求平均响应时间变长，甚至出现应用程序无法响应的情况

四.用于监测系统性能的命令
=======

**1.vmstat**

使用vmstat能够很好的知道目前系统的运行情况，vmstat输出的数据格式一般如下：

    procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
    r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa

我一般比较关注的数据有如下几项：

r 和 b ： r和b两列数据分别代表系统中正在执行的任务和阻塞的任务，这列往往能够以最直观的方式让你知道你的程序或者现在是不是正遇上了某些问题，尤其b这一列，他代表的正在阻塞的任务数，如果说b的数值长期不为0或者经常比运行中的任务数多，这就代表系统已经有瓶颈的征兆了，我们可以进行进一步的瓶颈分析了

cs ： 每秒钟上下文切换次数，上下文的切换有可能是调用系统函数的时候产生的，也有可能是应用程序本身线程相互切换产生的，上下文切换会浪费大量的CPU的时间片，举个不一定那么恰当的例子，如果说切换一次上下文平均要花费1ms的时间，那么我在一个单核的CPU上，我一秒钟切换800次上下文，我看起来CPU在一直被使用，但是实际有用的处理时间只有200ms，当然会影响到系统的吞吐量了。

us 和 sy : 系统态和用户态CPU使用的情况，这个我觉得不能一概而论，具体需要
分析应用程序是计算密集型的，还是IO密集型的。

**2.iostat**

    Device:    rrqm/s wrqm/s   r/s   w/s  rsec/s  wsec/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await  svctm  %util

iostat能够很好的对挂载好的磁盘进行实时监控，查看磁盘的性能和使用情况

一般可以关注如下几个指标：

await ： io任务的平均等待时间，自然越短越好

svctm ： io任务的平均处理时间，自然越短越好

util : 磁盘的总体使用情况，100%就是最高，表示磁盘已经满载运行

rsec/s  wsec/s ： 这两项和util一起可以大致估算出磁盘的读写能力

**3.jstat**

我工作的主要语言是Java，就像之前所说的，Java程序在吞吐量达到一定的情况下，很容易会出现GC问题，常见的GC问题有：

1.频繁的YoungGC，比如说每1秒出现一次100ms以上的YoungGC

2.周期性的FullGC，比如说每隔几分钟出现一次长达几秒的FullGC

3.GC死循环，往往是JVM由于GC的时候对象晋升失败，需要进行一次FullGC，而又因某些原因FullGC进行的很慢但是此刻系统的吞吐量很高，导致JVM陷入一个GC的循环，基本所有时间都用在进行GC上面

我很喜欢使用jstat -gc pid这个命令对JVM进行初步的分析，排查JVM是否真的出现了GC问题

jstat -gc pid 如果你的垃圾回收期是使用CMS垃圾回收器的话，一般会出现如下数据:

    S0C S1C S0U S1U EC EU OC OU PC PU YGC YGCT FGC FGCT GCT

这里就不贴具体的含义了，有兴趣的同学自己去查一下吧。

我一般观察的话，会先去计算GC时间和总运行时间之间的比例关系，这个比例当然是越低越好，因为JVM在进行GC的时候，是不会去处理你的业务代码的！

jstat输出的数据是历史累加的，所以可以通过定时输出jstat的数据，然后计算出在这个时间段内GC所花费的时间，来计算出一个比例关系，GC时间占用越多，证明你的GC问题越严重

在估算完GC时间的占比之后，还可以分析数据来得出，大概程序每隔几秒做一次YoungGC，历时多长，大概程序每隔几秒做一次FullGC，历时多长，这些统计值的意义对于指导我们进行JVM调优是非常有帮助的，这些统计值在我看来非常有意义的原因是：

JVM调优我觉得并不是可以一种配置到处运行的参数，而且必须根据业务情况，自己Java程序的特性，请求访问量，响应时间要求等各个方面来综合考虑的，比如说每5秒1次50ms的YoungGC，和每50秒一次的500ms的FullGC（其他YoungGC忽略不计），这两种GC情况谁好，很难得出一个绝对的结论，比如说我的程序是一个在线聊天室，我每5秒有一次50ms的GC导致响应变慢了，用户对50ms的感知不一定有那么敏感，他会觉得这个聊天室响应一直都是那么快的，但是如果我们是50秒一次500ms的FullGC，用户是不是很明显会觉得，我每隔一段时间，总会觉得卡了那么一下，可能体验没有那么好。而在别的场景下，说不定又是50秒一次500ms的FullGC是更优的选择。所以JVM调优必须结合自身的特性来的。

我们还可以观察一下CMS中各代内存的使用情况，来判断我们的内存分配是不是合理

**4.程序Profile工具**

如果需要更加细致的性能分析，这里可以用一些Profile工具，这里我推荐一个：

[JProfiler](http://www.ej-technologies.com/products/jprofiler/overview.html)

能够详细的分析每个线程的调用栈及其调用时间

###四....暂时先到这，日后补充咯











