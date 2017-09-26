---
layout: post
title: Java中的并发控制
description: Java Concurrent Control
category: blog
---
这篇博文主要总结一下Java中常用的并发控制手段，也不会设计太多JVM或者字节码层面的介绍，想根据的了解各种并发控制的底层是怎么实现的，还请移步JVM规范。

## 基础篇 ##

### synchronized ###
synchronized关键字是Java中最基本的并发控制手段之一，目的是让程序只能够**串行**地执行某个代码块的内容，来确保正确的执行某段流程，如改变某些全局唯一的属性等等.

#### 作用域 ####
synchronized关键字可以用来修饰一个方法(静态方法或者普通方法)，也可以用来修饰一个代码块，样例如下：

    public synchronized static void test1()
    {
        
    }

    public synchronized void test2()
    {
        
    }
    
    public void test3()
    {
        synchronized (this)
        {
            
        }
    }

synchronized修饰之后的代码段，每次只能同时被一个线程执行

#### 原理 ####
在Java中，每个对象都会有一个与之关联的Monitor，只有当线程获取到特定对象的Monitor，才可以进入被synchronized修饰的同步块中，在JVM层面上，当java文件被编译成字节码的时候，针对同步代码块，会在代码块的入口和出口分别加入monitorenter和monitorexit指令，来获取特定对象的Monitor和释放该对象，针对同步方法，通过设置方法的标志位来实现。

synchronized作用于不同的域，需要获取的Monitor对象也不同，可以简单的理解一下：

1. 作用于静态方法，需要获取该类的Class对象的Monitor作为锁，这样子的锁自然是全局唯一的，所以说被synchronized修饰的静态方法是全局锁定的。
2. 作用于普通方法，需要该类的实例化对象的Monitor作为锁，即不同对象的被synchronized修饰的方法是可以同时执行的。
3. 作用于代码块，需要获取用户指定的对象的Monitor作为锁，锁范围由对象的实际情况决定。

### Object的wait()，notify(),notifyAll() ###
Object对象提供了wait()，notify(),notifyAll()，可以通过这三个方法来对并发任务就行条件控制，由于Java中所有对象的隐式父类都是Object，因此在所有对象上面都有这三个方法，任何一个对象都可以作为并发任务中的并发控制员。

先说一下这三个方法的使用条件:调用该方法的线程必须拥有同步对象的Monitor，否则会抛出异常IllegalMonitorStateException,获取Monitor的方式只有三种:
1. 调用静态同步方法
2. 调用普通同步方法
3. 进入同步代码块

简而言之就是，wait()，notify(),notifyAll()调用是需要在synchronized修饰的代码段中

各个方法的作用：

1. wait()，当前线程放弃Monitor的所有权，暂停自身的任务，等待被notify(),notifyAll()唤醒
2. notify(),任意选择一个调用了wait()的线程，让其唤醒并且重新开始执行
3. notifyAll(),唤醒所有调用wait()的线程，这些线程会被全部唤醒并且竞争Monitor对象，只有其中一个能够重新执行任务，没有争夺到Monitor的线程会转变成BLOCKED状态，等待一下次竞争Monitor对象

notify和notifyAll的区别，**大部分情况下你都应该用notifyAll()而不是notify()，尤其是当你不是很清楚该用哪个的时候**

在说明之前，先附带说一下线程状态的BLOCKED和WAITING的区别，BLOCKED的线程一般是在等待进入monitor区域，当monitor区域可用的时候，会自动转变成Runnable状态，而WAITING是必须要被其他线程notify或者notifyAll才能唤醒的，这里借用一个stackoverflow上面的一个问答来阐述为什么我们更应该使用notifyAll()

[notify vs notifyAll](https://stackoverflow.com/questions/37026/java-notify-vs-notifyall-all-over-again "-notify-vs-notifyall")

**xagyg**举了个生产者消费者的例子，来说明这个问题,详细请看上面链接

>STEP 1:
>- P1 puts 1 char into the buffer

>STEP 2:
>- P2 attempts put - checks wait loop - already a char - waits

>STEP 3:
>- P3 attempts put - checks wait loop - already a char - waits

>STEP 4:
>- C1 attempts to get 1 char 
>- C2 attempts to get 1 char - blocks on entry to the get method
>- C3 attempts to get 1 char - blocks on entry to the get method

>STEP 5:
>- C1 is executing the get method - gets the char, calls notify, exits method
>- The notify wakes up P2
>- BUT, C2 enters method before P2 can (P2 must reacquire the lock), so P2 blocks on entry to the put method
>- C2 checks wait loop, no more chars in buffer, so waits
>- C3 enters method after C2, but before P2, checks wait loop, no more chars in buffer, so waits

>STEP 6:
>- NOW: there is P3, C2, and C3 waiting!
>- Finally P2 acquires the lock, puts a char in the buffer, calls notify, exits method

>STEP 7:
>- P2's notification wakes P3 (remember any thread can be woken)
>- P3 checks the wait loop condition, there is already a char in the buffer, so waits.
>- NO MORE THREADS TO CALL NOTIFY and THREE THREADS PERMANENTLY SUSPENDED!

在这个案例中使用notify的主要问题发生在所有STEP7的时候，P2只唤醒了一个生产者(这时候我们期待唤醒的是消费者)，导致所有线程最终都进入WAITING状态，导致死锁。如果我们使用的是notifyAll，此时所有消费者都会从wait状态转变成blocked状态，就算是P3先争抢到monitor，在P3执行完之后，blocked状态的消费者会重新竞争锁，从而解决死锁。

### volatle ###
volatle是Java中一个并发中常用的变量，保障变量的值在各个线程之间是有**happens-before**的关系的，通俗的说作用有两个：

1. 保证后执行的线程读取的变量的值一定是前一个线程修改后的值
2. 阻止指令集的重排序

#### 原理 ####
要了解volatle的原理，要先了解Java的内存模型。

在现代的CPU中，为了匹配CPU和内存的速度关系，CPU中会有很多寄存器，多级缓存，CPU操作寄存器会比操作内存快很多，很多情况下，CPU会将内存中的数据存到寄存器中，然后进行一系列的修改，最终再把寄存器的存储的值重新写会内存，这就造成了变量的可见性问题。

Java中也是如此，每个线程都拥有一个独立的线程栈，线程访问某个变量的时候，也是通过变量的引用找到变量的值在内存中的位置，然后将变量取到自己的线程栈中，在操作完成之后再重新写回去内存。

被volatle修饰的变量，可以理解为在读取的时候，都要重新到主内存中读取，而不是直接读取本地栈中有可能已经过期了的值.

另外volatle在jdk1.5之后也优化了指令的重排序，有兴趣的可以看这个案例简单了解一下volatle重排序的作用:

[double-check-locking](http://www.javaworld.com/article/2074979/java-concurrency/double-checked-locking--clever--but-broken.html)

### interrupt(),interrupted(),isInterrupted() ###
在Java中，去中止一个运行的线程，一般不会采用类似于"Kill掉一个进程"这样粗暴的方法，而是通过通过设置一个interrupt的FLAG来控制进程的中断，这里先简单说下这3个方法的作用:

1. interrupt()：将进程的interrupt的FLAG设置为true
2. interrupted()：返回进程当前的interrupt的值，并且将interrupt设置为false
3. isInterrupted()：返回进程当前的interrupt的值，不改变interrupt的值

首先要说明的是，设置了interrupt变量不代表线程一定会中断，线程的中断应该是由具体的流程控制的，简而言之就是线程的流程里面需要有判断interrupt状态并且抛出异常的代码块存在，才能让进程中断，最常见的就是Object.wait(),Object.sleep()这样子的方法，这些方法都会抛出InterruptedException方法，就证明他们会去处理程序的中断信号

举个例子：

        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                long l = 0;
                while(true)
                {
                    l++;
                    if(l%1000000000==0)
                    {
                        System.out.println(l);
                    }
                }
            }
        });
        thread.start();
        thread.interrupt();

这样子的一段代码，是永远不会被中止的，因为我们代码块里面没有显式或者隐式的对interrupt信号进行处理

在如下的例子，我们调用了sleep方法，sleep方法是会去处理中断信号的，并且抛出InterruptedException异常的

        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                long l = 0;
                while(true)
                {
                    l++;
                    if(l%1000000000==0)
                    {
                        try {
                            Thread.sleep(1000L);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        });
        thread.start();
        thread.interrupt();

这个方法会在一般第一次sleep的时候直接被中止(这个主要是取决于interrupt和sleep的时序问题)

最后，不要指望interrupt能够一定能够中断你的线程，这个取决于你自己的实现！



### 进阶篇 ###
进阶篇将会讲述java.util.concurrent中一些主要的并发控制方法，如锁，信号量，信号队列等等
待续....


[Edward]:    http://Edward0205.github.io  "Edward"
