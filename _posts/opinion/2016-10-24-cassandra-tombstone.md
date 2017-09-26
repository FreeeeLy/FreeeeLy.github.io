---
layout: post
title: Cassandra墓碑探究
category: opinion
description: Cassandra Tombstone
---

## 什么是TombStone ##
在传统的关系型数据库中，删除的操作往往是先对将要删除的列查找出来，然后再将其从硬盘上和缓存中抹去，但是在NoSQL数据库Cassandra中，在执行删除操作的时候，并不会先把将要删除从磁盘上或缓存中查找出来，而是直接向数据中插入一条新的记录，并且做上特定的标志，这条记录就是我们所说的TombStone。TombStone用于标识已经被删除的记录，因此Cassandra中删除一条记录实际上是往数据库中插入了一条记录，所以Cassandra在进行删除操作的时候也有很高的性能。

## TombStone怎么起作用的 ##
由于Cassandra是用时序判断集群中数据的一致性关系的，简单来说就是按照时间戳谁大谁赢来决定哪条记录是最新插入的,当Cassandra在归并处理数据的时候发现Tombstone的时间戳比其他非Tombstone的时间戳要新的时候，那么就表明这条记录已经被删除了，不向客户端返回记录。

## TombStone对C*集群的影响 ##
由于Cassandra删除的时候实际上只是往数据库中插入了一条相对特殊的记录，在我们的应用程序中，对数据库增删改查的操作是非常之多的，实际上所有的数据写操作都是直接往磁盘里面插入了一条拥有更加新的时间戳的记录，但是问题在，非墓碑的记录会在SSTable(Cassandra数据在磁盘上的载体)在进行COMPACT操作的时候进行合并，但是墓碑却需要有其他额外的条件才会被消除（条件在本文后续叙述）。那么在Cassandra长时间运行的过程中就会出现一个问题，一条同一个主键的记录被重复删除了成千上万次之后，我们根据主键去查询这条记录，这个时候，成千上万条墓碑记录和若干有效的记录同时被读取出来，然后进行合并筛选给客户端返回最终的结果。在高并发的情况下，海量读取拥有大量墓碑的行记录，会对Cassandra的堆内存产生巨大的压力，Cassandra进程开始GC，对外响应变慢，甚至造成Cassandra内存泄漏崩溃或者不断的死循环GC，对整个集群的稳定性产生影响。所以当我们查询记录的时候，虽然最终返回给我们的结果仅仅是一条记录，但是内里的差异可能是非常巨大的。

此外，虽然说Cassandra的节点都是对等的，没有中心和普通节点之分，而且在多副本的情况下也不会有单点失效的情况，但是这只是对于整个集群而言的，如果说我们的应用将其请求发送给一台正在严重GC的节点，这个请求将会被一直挂住，直到请求被正确处理(可能等待长达10s以上的GC)或者超过设定的请求超时时间才能返回，那么对于我们的应用来说，这种情况实质上跟单点失效区别也不大了，这么长的等待时间是无法忍受的。

所以说解决墓碑问题是很有必要的。

## Cassandra Table的Key类型 ##
1.Single Primary Key 单一主键例子:

    Create table mykeyspace.test (
           id int,
           name text,
           primary key(id) );

Primary key只包含一列，该Key用以计算Cassandra的PartitionKey

2.Compund Primary Key 混合主键例子:

    Create table mykeyspace.test2 (
           id int,
           age int,
           name text,
           primary key(id,age) );

Primary Key包含两列(或者两列以上),primary key中的设置的第一列用以计算Cassandra的PartitionKey，剩下的若干列称之为Clustering columns，这些Clustering columns是用来在同一个partition进行再次分区的，再分区的排序是按照在primary key中定义的顺序来决定的。

## Cassandra墓碑类型 ##

1.作用于Partition Key的墓碑，这里暂时叫DeletionInfo
这种墓碑是作用于整个Partition Key的，简而言之就是作用于表格的primary key属性设置的第一列的

2.作用于Clustering Columns的墓碑，这里叫RangeTombStone
这种墓碑是作用于某一列或者多列Clustering Columns上的，简而言之就是作用于表格的primary key属性设置的第一列之后的列.

3.作用于普通Column上的墓碑，这里叫TombStone
这种墓碑是作用于Primary Key中设置的列之外的表格中的Columns

### 如何知道某次查询中搜索出来多少个墓碑 ###
在Cassandra的配置文件cassandra.yaml中，有两个比较重要的关于TombStone的配置项:

tombstone_warn_threshold 设置一次查询中超过多少墓碑就在日志中告警

tombstone_failure_threshold 设置一次查询中超过多少墓碑就返回查询失败

或者可以在CQLSH命令行中开启TRACE，也能知道该次查询涉及的墓碑数量

通过阅读源码发现，会被进行告警和计算的墓碑只有上述说的第三种墓碑，即作用于普通Column上的墓碑，这里还有一点要说明的，所有类型的墓碑都会在一次查询中被查询出来，所以不是说其他墓碑不会对查询性能产生影响,但是计入threshold统计的墓碑只有第三种墓碑类型而已。

以Cassandra2.1.14为例,相关源码可以查看:[SliceQueryFilter](https://github.com/apache/cassandra/blob/cassandra-2.1.14/src/java/org/apache/cassandra/db/filter/SliceQueryFilter.java)的collectReducedColumns方法

## CQL -DELETE语句对墓碑的影响 ##
删除同一行记录，不同的CQL语言会产生不一样的墓碑
这里设定表格如下:

    Create table mykeyspace.test2 (
           id int,
           age int,
           name text,
           primary key(id,age) );

1.对Partition Key进行删除

    Delete from mykeyspace.test2  where  id = 1

这里是根据表格的partition key来删除表中的整行记录的，将会生成DeleteInfo,利用cassandra自带的sstable2json工具，转化后的可视化的格式如下:

    {"key": "1",
     "metadata":{"deletionInfo": {"markedForDeleteAt":1456907233514778,"localDeletionTime":1456907233}},
     "cells": []}
    ]

从该段JSON可以看到Partition Key的值，被标记删除的时间

2.对Clustering Columns进行删除

    Delete from mykeyspace.test where id = 1 and age = 1;

这里是根据表格的partition key和clustering columns来联合删除表格中的整行记录,将会生成RangeTombStone，可视化格式如下:

    {"key": "1",
     "cells": [
    ["13:_","13:!",1456909165592151,"t",1456909165]
    }

输出中的”t”表明该段墓碑是RangeTombStone

3.对普通的Columns进行删除

    Delete name from mykeyspace.test where id = 1 and age = 1;

这里是根据表格的partition key和clustering columns来联合删除表格中的某一行某一列记录,将会生成TombStone，可视化格式如下:

    {"key": "1",
     "cells": [
           ["12:","",1456908695276686],
           ["12:name",1456908706,1456908706444735,"d"],
    }

输出中的”d”表明该段墓碑是TombStone

4.CQL删除语句总结

1.Where中只指定Partition Key 产生的是DeletionInfo

2.Where中指定了Partition Key和若干个Clusering Columns但是不指定normal column，产生的是RangeTombStone

3.Where 中指定了Partition Key 和所有Clustering Columns和若干个normal column产生的是TombStone

## 怎么清理墓碑 ##

常见的对墓碑的处理有两种：Compaction 和 Repair

### Repair ###
首先说明的是，Repair不能够清理墓碑，Repair主要是用来实现Cassandra中anti-entropy这一概念的，主要负责两部分的工作:

1.保持集群中primary数据和replica数据的一致性，确保数据都是最新的.

2.管理集群中的Consistency Level，比如因为节点故障而丢失了某些replica数据，利用repair工具可以进行修复

Repari对墓碑行的处理也是将每个节点上的行做一下同步，确保同一条记录的所有副本都处于被标记状态，但是不能够清除墓碑

### Compaction ###

经常有一种误解，以为Compaction执行了之后就能够清除墓碑，其实是错误的，Compaction确实是能够有效的清除墓碑，但是Compaction清除墓碑是要有条件的，就是只有超时的墓碑才会被清除，墓碑的超时时间是由gc_grace_second控制的，gc_grace_second是table中的一个属性,所以墓碑的清理要同时符合两个条件:

1.墓碑所在的sstable进行compaction操作了

2.墓碑已经超时过期了

这里设置gc_grace_second要注意到一个问题，gc_grace_second设置过短，很容易出现被删掉的数据重现的异常情况，举个例子：

A B C三个机器中都一条记录R，在某个时刻，C机器脱离集群了，现在我们执行一个删除记录R的操作，此时A B上的R记录都已经被TombStone标记了，然后超过了gc_grace_second时间，A 和 B机器进行compaction操作，接着C集群回来，C集群上的R记录仍然没被删除，我们进行查询，可以查询到被删除的R记录的消息

想要快速的清除墓碑，只能通过设置较小的gc_grace_second并且执行compaction操作

## 插入新的行时指定了TTL时间的数据 ##

在插入数据的时候，要是指定了TTL时间，在超过数据的TTL时间之后，超时的那一行数据除了Partition Key之外，所有的列都会变成TombStone，这种墓碑是会被查询线程统计到的，要是墓碑太多会出现查询失败的情况。
典型的应用，举个例子：IM应用的消息，可能会设置一个聊天记录保存时间，超过这段时间之后聊天就消失，这个时候我们插入的数据可能会使用到TTL的功能，要是用户的数据长时间没有进行墓碑清理，就会出现查询失败的情况。

极端的消除TTL数据墓碑的方法：设置gc_grace_second为0，并且定期执行compaction

这里要说明一下，在这种情况下设置gc_grace_second设置为0，只要不主动的去删除数据，而是等待数据超时了自动删除的话（无论节点在不在线），是不会出现因为gc_grace_second时间过短，当出现集群异常的时候，被删除的数据重现的情况。

因为在Cassandra中，被标记了TTL的列，都是用特殊的BufferExpiringCell对象来表示的，这个对象中含有TTL的一个属性，在查询的时候会利用TTL进行一个时间的对比，只要超时了，数据就不会被查出来，并且进行压缩的时候就自动变为TombStone。

##如有错误之处，希望大家提出


