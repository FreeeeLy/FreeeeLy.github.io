---
layout: post
title: Java中的并发控制-进阶篇
description: Java Concurrent Control
category: blog
---

## 进阶篇 ##
进阶篇主要以java.util.concurrent.locks包为基础，讲述更加灵活的并发控制手段，如锁，信号量，信号队列等等


### Condition ###
在基础篇我们说过Object的wait()，notify(),notifyAll()方法，Condition可以想象为这3个方法的一种分离实现，Condition为我们提供了更加精细并发控制方式，Let see

首先，Condition不是一个可以独立存在的对象，每个Condition一定是跟某个java.util.concurrent.locks.Lock关联起来的，由Lock的newCondition()生成，**同时一个Lock可以同时拥有多个Condition**,这个是Condition优越性之一，以官方文档的一个案例来说明:

    class BoundedBuffer {
    final Lock lock = new ReentrantLock();
    final Condition notFull  = lock.newCondition(); 
    final Condition notEmpty = lock.newCondition(); 

    final Object[] items = new Object[100];
    int putptr, takeptr, count;

    public void put(Object x) throws InterruptedException {
     lock.lock();
     try {
       while (count == items.length)
         notFull.await();
       items[putptr] = x;
       if (++putptr == items.length) putptr = 0;
       ++count;
       notEmpty.signal();
     } finally {
       lock.unlock();
     }
    }

    public Object take() throws InterruptedException {
     lock.lock();
     try {
       while (count == 0)
         notEmpty.await();
       Object x = items[takeptr];
       if (++takeptr == items.length) takeptr = 0;
       --count;
       notFull.signal();
       return x;
     } finally {
       lock.unlock();
     }
    }
    }

上述代码用condition构建了一个生产者-消费者的模型，Condition的await和signal方法对应的是Object的wait和notify方法，和Object的wait和notify一样，Conidtion的await和signal方法也是需要在获取了锁之后才能调用的方法，不然会抛出IllegalMonitorStateException异常.

在上述代码中，用多个condition来区分不同的等待队列，相比起Object的wait使用同一个等待队列，使用Condition可以更加精细地控制并发，减少不必要的资源消费，可以定向唤醒某些特定等待条件的线程，比如专门唤醒生产者或者专门唤醒消费者，避免生产者重复地唤醒生产者这种尴尬的场面。

其实Object也可以实现分类等待的功能，但是要使用多个Object，会在代码块中调用多次synchronized，一不小心就会造成死锁！

注意，使用condition的时候最好使用一个while循环去判断条件，防止假唤醒。

### Lock ###
在java.util.concurrent.locks中，提供了高级的同步类**Lock**，Lock需要在同步代码前后的分别调用lock()和unlock()方法去获取锁，释放锁

Lock的于synchronized的不同点主要有如下几个:

1. 可中断机制:通过Lock.lockInterruptibly()方法可以实现一个可中断的锁，即获取锁的这个行为可以被其他线程中断，而synchronized是不可中断的，一个线程如何正在等待获取锁，那它必须一直等待，直到获取了锁
2. 公平性选择,可以让一把锁实现公平性竞争，即先来申请的线程先获取锁
3. 读写锁，可以将锁分为读锁和写锁，可以实现多个线程同时读和单个线程排它写的功能

如果你用到以上的特性，请考虑Lock而不是synchronized，但是不要为了必要的"高逼格"而滥用Lock，如果是很简单的场景synchronized足矣，在普通的场景，synchronized的性能不会比Lock差！

### LockSupport ###
LockSupport提供一种同步原语，作用类似于Object.wait()和Object.notify(),但是有几点要注意的:

1. LockSupport的unpark和park调用没有先后的序列关系，A线程调用了park()方法,B线程无论在是A线程调用park()之前或者之后调用unpark()，A线程都会唤醒，我个人认为这是LockSupport最大的优点之一吧，可以很好的解决我们一些先后关系，防止出现类似死锁的现象，这是Object.wait和notify做不到。
2. LockSupport是不可重入的，无论调用了多少次unpark，只能唤醒一次调用了park()的进程.

        LockSupport.unpark(Thread.currentThread());
        LockSupport.park();
        LockSupport.unpark(Thread.currentThread());
        LockSupport.unpark(Thread.currentThread());
        LockSupport.park();
        LockSupport.park();
像这段代码，会在第三次park的时候进入阻塞的状态，即，在一次park()被唤醒之前，无论调用多少unpark，只能够唤醒一次park线程
3. 对于中断的响应不会抛出InterruptedException异常，会被直接唤醒，继续执行后面的语句,这点和Object.wait()是不同的

        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("Enter thread");
                LockSupport.park();
                System.out.printf("Quit thread");
            }
        });
        thread.start();
        thread.interrupt();
        Thread.sleep(1000000L);
以上代码会打印出Enter thread和Quit thread，无须额外捕捉异常，如果是使用wait方法，则需要去捕获InterruptedException.

### Semaphore ###
关于信号量的概念不在这里介绍了，网上很多文章经常说用Semaphore来控制一些有限资源的并发访问，其实我个人认为在Java中拥有多种并发工具的情况下，Semaphore的工程作用已经比较小，主要是要从语义上面去理解信号量的作用，这里就简单的带一带就过去了。

### CountDownLatch 和 CyclicBarrier ###
可以将CountDownLatch和CyclicBarrier理解为一种线程状态的屏障，让一个线程或者一组线程达到某个状态之后才继续运行。

先说CountDownLatch，CountDownLatch允许一个线程或者一组线程等待另外一组线程都完成了指定的任务之后才开始运行，举个简单的用途，我们的Map-Reduce任务，在所有Reduce任务完成之后，我们需要将所有的子结果汇总成一个最终的结果，这个时候就比较适合CountDownLatch的场景，我们的汇总任务需要等待所有Reduce任务完成之后才能够开始汇总。

CountDownLatch的构造函数接受一个int参数，用来指定需要等待完成的任务数，通过调用CountDownLatch.countDown来表明一个子任务已经完成，主任务调用CountDownLatch.await来等待所有子任务的完成

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(5);
        for (int i = 0; i < 5; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(1000L);
                        System.out.println("Worker job finish");
                        countDownLatch.countDown();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
        countDownLatch.await();
        System.out.println("Main end");
    }

如这段代码，主线程会等待5个子线程的任务完成之后，才会继续执行，另外CountDownLatch是不可以重复使用的，每次使用必须重新创建一个实例

CyclicBarrier作用是让一组线程互相等待，直到这一组线程同时到达某种状态之后，再同时继续执行
CyclicBarrier的构造函数可以收一个int参数来表示需要等待的线程，和一个Runnable参数，将会在所有等待的线程到达之后，但是线程任务重新开始之前执行.

    public static void main(String[] args) throws InterruptedException {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(2, new Runnable() {
            @Override
            public void run() {
                System.out.println("All player come,ready to start");
            }
        });
        for (int i = 0; i < 2; i++) {
            final String player = "player" + i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        System.out.println("Runner "+ player + " come");
                        cyclicBarrier.await();
                        System.out.println("The other comes");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }

同时CyclicBarrier和CountDownLatch不同的是，CyclicBarrier可以通过reset方法来进行重用，但是要注意的调用reset方法可能会导致正在等待的线程抛出BrokenBarrierException异常，停止等待。

其实CountDownLatch和CyclicBarrier的区别还可以从语义上面去理解，CountDownLatch是一组线程去等待另一组线程完成了某些任务之后再自己执行，因此所有线程是分为两组意义不同的线程的，而CyclicBarrier则是同一组线程之间相互等待。还是用赛跑的例子来说：

在没有裁判的情况下，运动员之间的等待是CyclicBarrier，所有人到了就可以开始赛跑了

在有裁判的情况下，对于裁判来说，他需要的是CountDownLatch，他需要判断所有的运动员到了，他才会发起开始赛跑的命令

### 并发控制核心之AQS - AbstractQueuedSynchronizer ###
AbstractQueuedSynchronizer是java并发包的核心组件之一，在官方的API里面有一句话:
>  Provides a framework for implementing blocking locks and related synchronizers

可见其重要性，实际上AQS也是我们之前提到的Condition,Lock,Semaphore,CountDownLatch以及CyclicBarrier的核心组件

从名字上去解读一下:

abstract证明这个类需要被继承，去具体化其功能，本文提到的并发工具都实现了其子类

Queued使用队列的机制去维护并发关系，实际上使用一个CLH队列去维护各个线程之间的关系

Synchronizer说明了用来同步的:)

AbstractQueuedSynchronizer概括性地来说通过一个双向的链表来维护一组线程之间的状态

CLH队列中的一个节点主要包含4个属性:

1. prev-前一个节点
2. next-后一个节点
3. thread-正在等待资源的某线程
4. state-该线程的状态

到目前为止，我们一直说AQS维护的状态而不是维护一把锁，这点其实很关系，状态的含义是由具体的子类去赋予的，状态可以是锁，也可以是其他的一些资源，这样为AQS提供了很大的拓展性，思维也不要局限在锁的理解上.

AbstractQueuedSynchronizer通过CLH队列以及CAS操作(通过unsafe完成)来完成对资源的控制

以一个锁为例,流程大致如下:

1. 一个线程通过CAS去尝试获取AbstractQueuedSynchronizer代表的锁资源，如果设置成功，那么该线程可以直接进入临界区，进行一些操作
2. 如果线程获取锁失败，那么把这个线程加入CLH队列，这里也是通过CAS，把线程加到队列尾部
3. 当AbstractQueuedSynchronizer的锁资源被释放的时候，唤醒CLH队列的头结点，然后线程之间再次竞争这个锁资源:这里可以实现有很强的扩展性，可以是公平锁-按照先进先获取的原则获取锁，可以是非公平锁，大家再次在同一起跑线上竞争，可以是共享锁，让多个线程同时获取资源，可以通过CLH节点中的state状态来巧妙的完成这些事情.

因此实现一个锁的关键是实现一个AbstractQueuedSynchronizer子类,并且去实现其tryAcquire和tryRelease(实现独占锁从简来说是这两个，还有其他的一些拓展方法)

这里不想花太大篇幅再去剖析像ReenstrantLock等源码了，相信读者了解了AQS的原理之后，再回去看其具体实现会简单不少

### Unsafe ###
JAVA屏蔽了很多底层语言的特性，个人认为一个方面是使用语言的学习成本和维护性大大降低，比如像C语言中的内存管理；另外跟Java的**一次编译 到处运行**的宗旨也有关系，因为如果代码中对基础平台的特性是有依赖的，那么必然是跟平台有关的，很可能会失去跨平台的特性

但是JAVA中也开了个口子，让我们了解的风险的情况下，仍然去做一些底层语言才能完成的事情,这个就是Unsafe类，下面有一篇文章会说到Unsafe的作用，有兴趣的朋友可以简单了解下

[https://dzone.com/articles/understanding-sunmiscunsafe](https://dzone.com/articles/understanding-sunmiscunsafe "Unsafe")

简单概括一下，作用有如下几点:

1. 跳过构造函数去构造一个对象,常常用于一些反序列化的工作中
2. 开辟内存空间，就像C语言一样，允许单独开辟一块内存空间，比如Java的NIO。
3. 操作内存空间，如直接给某个对象的某个属性赋值
4. 并发控制，Unsafe.park和Unsafe.unpark来挂起/唤醒一个线程，上面说的LockSupport的底层实现就是Unsafe




[Edward]:    http://Edward0205.github.io  "Edward"
