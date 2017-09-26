---
layout: post
title: Cassandra Java Driver优雅生成streamId
description: Cassandra Java Driver利用掩码和原子变量生成StreamId
category: blog
---

Cassandra作为一个高性能的分布式数据库，除了自身良好的设计之外，一个为自己量身定做，性能强大的数据库Driver也是必不可少的，本文就Cassandra Java Driver来说，Cassandra Java Driver的底层网络的IO主要也是通过Netty来实现的，通过客户端和服务端之间异步调用来达到良好的性能效果，有兴趣的可以去查看官方文档了解更详细的内容，这里就简单的叙述一下。

Cassandra的连接管理
------

这里借用官方文档的一个图:

+-------+1   n+-------+1   n+----+1   n+----------+1   128/32K+-------+

|Cluster+-----+Session+-----+Pool+-----+Connection+-----------+Request+

+-------+     +-------+     +----+     +----------+           +-------+

首先Cluster是Driver与C\*服务端建立连接之后获取到的第一个对象，一次连接只会建立一个Cluster对象，
通过Cluster对象可以获取到很多关于C*集群的元数据信息，如Keyspace，Table，Host等等，通过Cluster的connect(),和connect(String keyspace)方法可以获取到Session，Cluster和Session的关系是1：n

接下来的就是Session对象，Session对象由Cluster对象建立，有了Session对象就可以直接进行一些sql操作了，每个Session对象里面维护了与已经建立好连接的节点的ConnectionPool，Session与ConnectionPoll之间也是1:n的关系,以Map的形式表示:

    final ConcurrentMap<Host, HostConnectionPool> pools;

然后在ConnectionPool里面也会维护若干个Connection，每个Connection都是一个长连接，Connection下面维护了很多Request，Request是发送请求的真实载体，Cassandra Driver和Cassandra服务端之间都是异步的调用，Cassandra会在Request上面带上一个StreamId，来区分每个请求，从而能够正确的处理所有的异步调用,在V2版本，每个Connection最多维护128个StreamId，在V3以上版本每个Connection最多可以维护32678个StreamId，因此Cassandra对每个Host的最大支持并发数是：

Connection数量*可维护的StreamId的数量

StreamId生成方式
------

上面简单的说完Cassandra连接管理层次，接下来就说一下本文的要说的重点，如何优雅的在一个范围之内并发生成StreamId并且可以重复的循环使用,在Cassandra里面主要是通过位操作（掩码）和原子变量来实现的，这里就挑128位的StreamId来进行解析。

假如要StreamId的范围在0-127，一个Long变量是64位，这里我们需要2个Long变量，一共有128位，即可表示128个StreamId，我们约定当bit上面为0的时候，表示这个StreamId已经被使用，bit上面为1的时候，表示这个StreamId处于可用状态。因此我们需要在一开始的时候初始化Long变量，设置变量的初始值为-1，表示所有StreamId均可用。

接下来我们讨论两个过程
------

一.获取一个可用的StreamId
------

获取一个可用的StreamId，我们主要是通过Long向量中的每一位的0或者1来判断StreamId状态的，我们先看一下相关源码：

    public int next() throws BusyConnectionException {
        int previousOffset, myOffset;
        do {
            previousOffset = offset.get();
            myOffset = (previousOffset + 1) % bits.length();
        } while (!offset.compareAndSet(previousOffset, myOffset));

        for (int i = 0; i < bits.length(); i++) {
            int j = (i + myOffset) % bits.length();

            int id = atomicGetAndSetFirstAvailable(j);
            if (id >= 0)
                return id + (64 * j);
        }
        throw new BusyConnectionException();
    }

    private int atomicGetAndSetFirstAvailable(int idx) {
        while (true) {
            long l = bits.get(idx);
            if (l == 0)
                return -1;

            // Find the position of the right-most 1-bit
            int id = Long.numberOfTrailingZeros(l);
            if (bits.compareAndSet(idx, l, l ^ mask(id)))
                return id;
        }
    }

    private void atomicClear(int idx, int toClear) {
        while (true) {
            long l = bits.get(idx);
            if (bits.compareAndSet(idx, l, l | mask(toClear)))
                return;
        }
    }

    private static long mask(int id) {
        return 1L << id;
    }

首先我们先取得一个位移量myOffset，我个人的理解是因为一方面程序总是处于高并发的状态，要是每次都从同一个位置开始去寻找可用的StreamId，可能会造成大量的无用的竞争，另外一方面可以将StreamId相对均匀的分布在两个Long型变量中，因此这里每次都将位移量+1再去求模达到提高一定性能的目的，然后再去寻找可用的StreamId。在找到了合适的位移量之后，我们将位移量求模得到我们去哪一个Long型里面去取StreamId。

atomicGetAndSetFirstAvailable方法通过Long.numberOfTrailingZeros方法获取Long最右边的一位1的位置,并且用这个位置生成一个掩码，掩码和Long变量进行异或运算去改变StreamId的可用状态

假如说代表StreamId的Long变量是00000100100(忽略前面若干位0),通过numberOfTrailingZeros得到的值是2，根据2去生成一个掩码,生成的掩码是100

00000100100和100异或的结果是00000100000，表明StreamId 3已经被使用了

二.释放一个可用的StreamId
------

这个和获取可用的StreamId原理差不多，比如说要释放StreamId 3，计算出的掩码为100，此时和Long变量进行或运算00000100000 \| 100 结果是00000100100，表明StreamId 3又重新可用了

三.处理竞争关系
------

在并发生成StreamId的过程中没有显式的调用锁去处理并发获取资源的问题,而是通过不断的循环和利用原子变量的CAS操作来解决竞争的问题

四.总结
------

通过位变量和原子变量的巧妙利用，一方面用很少的内存来存储大量的StreamId，另外一方面利用原子变量的CAS特性来处理高并发情况下的资源竞争关系，优雅的在一个范围之内实现Id的分配以及复用
