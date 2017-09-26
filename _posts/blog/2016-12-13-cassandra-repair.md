---
layout: post
title: Cassandra repair机制
category: blog
description: 深入一点点看Repair
---

## repair的作用 ##
在Cassandra运行了一段时间之后，由于Cassandra的多副本存储机制，在某个时间点同一条记录的各个副本的数据可能是不一致的，当读请求采用较低的一致性级别的时候，可能会读取到脏数据，因此定期执行repair去同步集群中的是一个良好的习惯，能够有效的保证集群的数据的一致性。

## Repair机制 ##

### 基本原理 ###
Cassandra使用Merkle Tree作为Repair时的基本数据结构，主要是因为Merkle Tree在分布式环境下进行验证操作时，可以大大减少数据的传输量以及计算的复杂度，举个简单的例子：当我们要校验某个Token Range内的数据是否一致的时候，不需要将每一条数据都上报到发起Repair的节点，每个需要校验的节点将某个Token Range内的数据计算出来一个哈希值，然后将各个Token Range数据的哈希值生成一棵Merkle Tree，最后将Merkle Tree发往发起Repair请求的节点，在该节点上面进行数据校验即可。除了这个特点之外，Merkle还可以只比较某一个单独的分支，而不需要进行整棵树范围内的对比，也可以跟有效的进行并行任务。因此非常适合C*这种分布式的，大数据量下的数据验证场景。

### Repair主要流程 ###



1. 某个节点发起Repair请求，成为该次Repair操作的主节点
2. 主节点根据本次需要Repair的Token范围，计算出相关的邻居节点(拥有相同数据副本的节点)
3. 主节点向邻居节点发出请求，让其生成Merkle Tree
4. 主节点收集邻居节点生成的Merkle Tree，开始进行数据验证
5. 若Merkle Tree完全一致，则代表该Token范围内数据一致，完成Repair；否则开启数据同步流程，在该流程中各节点之间交换不一致的数据信息，对比验证之后进行数据更新
6. 如果使用了incremental repair，最后还需要做一次anticompaction，anticompaction会将一个sstable中repaired和unrepaired的数据分开，最终生成两个sstable

### Repair的范围 ###
Repair的粒度是Partition Key Range(-9223372036854775808到9223372036854775807 ),由于Repair行为是需要有一台发起节点的，所以每次Repair操作的范围就是发起节点在集群中所占有的Partition Key Range再加上由于副本分配策略决定的其他Partition Key Range.这里举一个例子:

![ring.png](http://www.liangye.info/images/cassandra/ring.png)

这里我们有一个4个节点的集群，每个节点所占的Token范围标注在上图中，此外每份数据有3个副本，副本分配策略我们用SimpleStartegy，即找到应该分配数据的第一个节点之后，顺时针再找两个不同的节点来放置数据副本,比如说一条记录的Token值是1，那么这条记录应该被放置在节点A，B和C
当我们直接对节点C执行命令nodetool repair的时候，这时候节点C会去修复Token范围在0-30的所有数据.

### Repair的种类 ###
1. 全量repair
每次对某个Token Range范围内的所有数据进行全量的验证和修复，即会对磁盘上所有相关的sstable都进行校验
2. 增量repair
顾名思义，只对磁盘上新增的，未进行过repair的sstable进行数据校验，每次repair之后都会给sstable做一个已经repair过的标记。

## Repair使用指导 ##

执行repair操作主要是通过C\*提供的nodetool工具，该工具在C*的bin目录下，命令的基本用法可以查看官方指导:
http://docs.datastax.com/en/cassandra/2.1/cassandra/tools/toolsRepair.html?hl=repair

### 常见使用场景 ###

#### 某个节点DOWN掉了，或者磁盘损坏了导致数据丢失 ####
这在C\*的日常运行之中最为常见，处理方式也很简单，由于只是单个节点的数据不一致了，我们只需要在该节点上面执行一次全量的repair，便可恢复该节点的数据，命令如下: nodetool repair <keyspace><column family>
TIPS:在节点数据量较大的时候，repair执行的时间会很长，而且repair操作会和C*竞争磁盘和CPU等资源，建议通过Token分段repair的方式去进行repair操作，在命令之中加入-st和-et参数，来指定这次repair涉及的数据范围。这样在服务器压力大的时候暂停repair，等到闲时重新进行；还可以避免repair遭遇异常而退出的时候，我们不知道repair的进度，而被迫重新repair，浪费大量时间的情况。命令可以参照:

           nodetool repair <keyspace> -st 1 –et 1000000

st和et的范围是-9223372036854775808到9223372036854775807，此外在使用分段repair的时候只能指定到keyspace级别，而不能指定到具体某个column family

#### 集群数据定期一致性维护 ####

##### 全量repair #####

我们要在集群范围内同步数据，即要完整修复-9223372036854775808到9223372036854775807的Token，但是每次repair的数据范围是有限的（2.3中提及），由于多副本的机制的存在，我们很难知道某个节点具体占有那些token范围的数据（除了与本身第一匹配的数据还有副本策略分配的过来的数据），所以我们不得不在每一个节点上轮流进行全量的repair，这是一种最基本的做法，这种做法从最终结果上面看是正确的，但是当中会遇到一个问题，举个例子:
如在2.3中的一个集群，当我们在每个节点上面都执行repair的时候，每个Token段都被重复执行了3次repair操作，这样会大量浪费我们的时间和资源.
我们做repair最理想的情况就是每份数据只被repair一次，经过官方文档指导和源码阅读，可以通过repair命令的-pr参数来满足我们的需要，这里贴一下-pr参数的作用:

            Use -pr to repair only the first range returned by the partitioner

意思是repair只会涉及到分区器计算出来的Token所在firstRange）,比如说节点C只会修复Token在21-30范围内的数据，不会去修复0-20范围内的数据,通过增加-pr参数，我们避免了对同一条数据的重复repair，极大的提高了repair的效率。所以正确的repair策略应该是在**每一台**机器上执行全量repair的时候加入-pr参数。

TIPS:用-pr参数来进行repair，请确保在**每一台**机器上面的repair操作都执行成功，否则的话会使得某些Token段的数据还处于不一致的状态。


##### 增量repair #####

增量repair的实质就是在repair过某个sstable里面的数据之后，给这个sstable做一个标记(repair的时间戳)，由于sstable是不可变的，在被成功repair过之后，那么sstable里面的数据是没有任何问题的。
但是官方的给出的迁移到incremental repair的步骤在生产环境中操作起来太困难:[官方指导](http://docs.datastax.com/en/cassandra/2.2/cassandra/operations/opsRepairNodesMigration.html?hl=incremental%2Crepair)

    To migrate to incremental repair, one node at a time:
		1.	Disable compaction on the node using nodetool disableautocompaction.
		2.	Run the default full, sequential repair.
		3.	Stop the node.
		4.	Use the tool sstablerepairedset to mark all the SSTables that were created before you disabled compaction.
		5.	Restart cassandra

试想一下生产环境中哪怕就一个几十节点的C*集群，上述操作是如此的艰难:

1. 停止自动压缩之后，可能会影响节点的读取性能
2. 每个节点轮流做全量的repair，时间太长了
3. 需要重启节点，并且用工具手动标记sstable为repaired状态

哪怕这些操作可以通过写一些脚本来执行，这个过程也是很难让人接受的。
因此我探究了一下简化步骤的可行性（主要是想能不能直接就执行增量repair，不去重启什么的），通过官方相关的文档，我对incremental repair的理解是这样子的：

1. full repair不会对sstable做一个repairedAt 的标记，只有incremental repair会做这个标记
2. 在做incremental repair的时候，已经被repairedAt 标记的sstable会被自动跳过
3. CompactionStrategy会区别repair和unrepair的sstable，做不同的处理，Size-Tiered区分repair和unrepair的sstable，分别执行Size-Tiered压缩方式,Leveled compaction会对unrepair的做Size-Tiered compaction，对reapired的做Leveled Compaction
4. incremental repair完成之后会额外多执行一步anticompaction.

我对官方指导的迁移到incremental repair如此复杂的步骤的产生了一些疑问:

> 我设想一个极端的情况，我的集群里面根本没有数据，官方指导的1 2 3 4 就是直接跳过了，我们相当于直接incremental repair。按照我的个人理解就是，这么复杂的原因就是因为CompactionStrategy会区别repair和unrepair，会影响我们数据的压缩状态(尤其是Leveled Compaction)

为了解答这个疑问，我对相关源码进行了阅读(基于C* 2.1.15)，这里只挑重要的地方简单说一下:

1. 当中有一个类:org.apache.cassandra.db.compaction.WrappingCompactionStrategy，从名字可以看出来这是个包装类，而且从源码上面去追溯，在进行Compaction相关的流程时，都由这个类执行，继续看这个类，有两个关键的属性:

    private volatile AbstractCompactionStrategy repaired;
	private volatile AbstractCompactionStrategy unrepaired;

其中AbstractCompactionStrategy是Compaction相关类的抽象类,可见在进行Compaction的时候，repaired和unrepaired的sstable确实会被分开来的.

2. 这两个属性的修改入口只有一个

    repaired = CFMetaData.createCompactionStrategyInstance(compactionStrategyClass, cfs, options);
    unrepaired = CFMetaData.createCompactionStrategyInstance(compactionStrategyClass, cfs, options);

可以看出来，调用相同的方法，入参也相同，可见返回值一定是一样的，因此repaired和unrepaired肯定是用的同一个CompactionStrategy，这个官方的说法有差异，经过搜索，我发现了一个JIRA单:[JIRA](https://issues.apache.org/jira/browse/CASSANDRA-8004),意思是这个问题是在2.1.2修改了，意思是unrepaired和repaired的压缩是分开的，拥有各自的分级，互不影响.

3. incremental repair之后会做一次anticompaction，,具体的方法是:
org.apache.cassandra.db.compaction.CompactionManager.doAntiCompaction

> 这个方法的具体作用是将一个sstable中repaired和unrepaired的数据区分出来，比如说一个sstable包含的Token范围是【0-20】，如果这次repair只涉及到【10-20】，那么执行doAntiCompaction之后，会生成一个【0-10】的sstable和【10-20】的sstable

4. incremental repair可能会生成新的sstable或者删除某些sstable，但是不会修改原有sstable的Leveled级别

## 我的观点 ##

在C* 2.1.15版本上是可以尝试直接对节点进行incrementalrepair，无需官方提及的过多的步骤的。

但是还有几点应该是需要注意的:

1. 直接做incremental repair也需要很长的时间，第一次做的时候相当于做全量的repair
2. doAntiCompaction这个步骤可能会影响到Compaction Status,试想一个Leveled Compaction的level 1 的sstable，里面都是fresh数据，但是在这次可能只repair了一部分Token，这个sstable也会被分成若干个sstable，都处于level 0,应该可以通过repair这个节点的所有Token来避免这种情况(待验证)
3. 开始做incremental repair之后就要持续的做下去，因为repaired和unrepaired的sstable是不会放在一起压缩的，就会导致数据分散，在查询的时候需要查询更多的sstable，影响查询效率.














