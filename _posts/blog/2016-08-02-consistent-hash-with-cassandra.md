---
layout: post
title: Cassandra中的一致性哈希应用
description: Cassandra中的一致性哈希
category: blog
---

什么是一致性哈希
======

[一致性哈希](https://zh.wikipedia.org/wiki/%E4%B8%80%E8%87%B4%E5%93%88%E5%B8%8C)

关于一致性哈希具体的概念在维基百科中已经有很详细的叙述了，这里就不再搬运了，简单的总结一下就是

**一致性哈希主要是为了解决在分布式的环境中，当网络拓补发生变化的时候（节点的新增或移除），如何高效地更新数据记录与存储节点的映射关系**

一致性哈希在Cassandra中的应用
======

[Cassandra](http://cassandra.apache.org/)是一套开源分布式NoSQL数据库，Cassandra的分布式也是基于一致性哈希算法来实现的,这里借用一下DataStax社区的图片来简单介绍一下

![Consistent Hash](http://docs.datastax.com/en/cassandra/3.x/cassandra/images/arc_hashValueRange.png)

先科普一个基础知识：**在Cassandra中，每一个表除了有传统的Primary Key之外，还有一个概念叫做Partition Key，代表Partition Key的列的Value会通过Cassandra一致性算法得出一个哈希值，来决定这一行数据该放到哪个节点上面**

图中每个节点拥有一段数字区间，这个数字区间的含义是：如果某行记录的Partition Key的哈希值落在这个区间范围之内，那么该行记录就该被存储到这个节点上边

一致性哈希进阶-虚拟节点
============

从上述的图中可以看出，每个节点都占据一段数字区间，那么问题来了，每个节点占据的数字区间的范围大小很有可能不是一样的，这样子就会造成数据分布的不均匀，这样子在生产环境中主要的危害是：

1.一般来说负载数据多的节点接受到的读写请求也越多，因此该节点机器的负载会远远比其他节点高. 

2.负载数据多的节点磁盘空间的消耗也快，需要增加磁盘或者增加节点来解决，但是实际上其他的节点的磁盘空间可能都没有被充分利用，造成浪费

为了解决数据不均匀的问题，一致性哈希提出虚拟节点的概念，简单的去理解虚拟节点就是：**将某个节点根据一个映射算法，映射出若干个虚拟子节点出来，再把这些节点分布在哈希环上面**，当有数据记录进来时，若是通过一致性哈希计算落到某个虚拟子节点上，这条记录就会被存在这个虚拟子节点的母节点上，这样子能够极大促进数据的均衡分布。

在Cassandra，虚拟节点叫做token，在Cassandra的配置文件中有一项配置叫做:**num_tokens**，这个配置项可以控制一个节点映射出来的虚拟节点的个数，在一个健康的集群中，可以通过Cassandra的自带的工具nodetool去查看集群的哈希环具体情况，命令为:**nodetool ring**

一致性哈希进阶-副本分布
========

一致性哈希可以帮我们解决哪条数据该放在哪个节点上面的问题，但是我们还有一个问题：数据库为了保证数据的可用性，一般同一条数据都会存多份副本，怎么在一致性哈希环上面存多份副本，方法很多，效果也是见仁见智，这里我主要就Cassandra的NetworkTopologyStrategy策略来说明一种副本分配的方法。

[NetworkTopologyStrategy](http://docs.datastax.com/en/cassandra/3.x/cassandra/architecture/archDataDistributeReplication.html)：简单的来说就是：当有多个Datacenter的时候(Cassandra中的概念，可以理解为多个机房)，把副本分别放在不同的Datacenter里面，在一个Datacenter里面，尽量将副本放在不同的机架上，来达到最高的可用性。

下面结合源码来阐述一下这个过程，源码为Cassandra2.1.14中的NetworkTopologyStrategy类：

    @SuppressWarnings("serial")
    public List<InetAddress> calculateNaturalEndpoints(Token searchToken, TokenMetadata tokenMetadata)
    {
        // we want to preserve insertion order so that the first added endpoint becomes primary
        Set<InetAddress> replicas = new LinkedHashSet<InetAddress>();
        // replicas we have found in each DC
        Map<String, Set<InetAddress>> dcReplicas = new HashMap<String, Set<InetAddress>>(datacenters.size())
        {{
            for (Map.Entry<String, Integer> dc : datacenters.entrySet())
                put(dc.getKey(), new HashSet<InetAddress>(dc.getValue()));
        }};
        Topology topology = tokenMetadata.getTopology();
        // all endpoints in each DC, so we can check when we have exhausted all the members of a DC
        Multimap<String, InetAddress> allEndpoints = topology.getDatacenterEndpoints();
        // all racks in a DC so we can check when we have exhausted all racks in a DC
        Map<String, Multimap<String, InetAddress>> racks = topology.getDatacenterRacks();
        assert !allEndpoints.isEmpty() && !racks.isEmpty() : "not aware of any cluster members";

        // tracks the racks we have already placed replicas in
        Map<String, Set<String>> seenRacks = new HashMap<String, Set<String>>(datacenters.size())
        {{
            for (Map.Entry<String, Integer> dc : datacenters.entrySet())
                put(dc.getKey(), new HashSet<String>());
        }};
        // tracks the endpoints that we skipped over while looking for unique racks
        // when we relax the rack uniqueness we can append this to the current result so we don't have to wind back the iterator
        Map<String, Set<InetAddress>> skippedDcEndpoints = new HashMap<String, Set<InetAddress>>(datacenters.size())
        {{
            for (Map.Entry<String, Integer> dc : datacenters.entrySet())
                put(dc.getKey(), new LinkedHashSet<InetAddress>());
        }};
        Iterator<Token> tokenIter = TokenMetadata.ringIterator(tokenMetadata.sortedTokens(), searchToken, false);
        while (tokenIter.hasNext() && !hasSufficientReplicas(dcReplicas, allEndpoints))
        {
            Token next = tokenIter.next();
            InetAddress ep = tokenMetadata.getEndpoint(next);
            String dc = snitch.getDatacenter(ep);
            // have we already found all replicas for this dc?
            if (!datacenters.containsKey(dc) || hasSufficientReplicas(dc, dcReplicas, allEndpoints))
                continue;
            // can we skip checking the rack?
            if (seenRacks.get(dc).size() == racks.get(dc).keySet().size())
            {
                dcReplicas.get(dc).add(ep);
                replicas.add(ep);
            }
            else
            {
                String rack = snitch.getRack(ep);
                // is this a new rack?
                if (seenRacks.get(dc).contains(rack))
                {
                    skippedDcEndpoints.get(dc).add(ep);
                }
                else
                {
                    dcReplicas.get(dc).add(ep);
                    replicas.add(ep);
                    seenRacks.get(dc).add(rack);
                    // if we've run out of distinct racks, add the hosts we skipped past already (up to RF)
                    if (seenRacks.get(dc).size() == racks.get(dc).keySet().size())
                    {
                        Iterator<InetAddress> skippedIt = skippedDcEndpoints.get(dc).iterator();
                        while (skippedIt.hasNext() && !hasSufficientReplicas(dc, dcReplicas, allEndpoints))
                        {
                            InetAddress nextSkipped = skippedIt.next();
                            dcReplicas.get(dc).add(nextSkipped);
                            replicas.add(nextSkipped);
                        }
                    }
                }
            }
        }

        return new ArrayList<InetAddress>(replicas);
    }

1.入参的searchToken是某行记录的PartitionKey计算出来的哈希值，tokenMetadata主要为Cassandra集群的网络拓补情况

2.4-23行是一些要用到的数据结构的初始化，解释一下各个变量的含义:

Set\<InetAddress\> replicas:副本的所有节点所分布的节点IP

Map\<String, Set\<InetAddress\>\> dcReplicas:每个Datacenter中副本分布节点的IP

Topology topology:Cassandra的网络拓补情况

Multimap\<String, InetAddress\> allEndpoints:Cassandra中每个Datacenter里面的节点IP

Map\<String, Multimap\<String, InetAddress\>\> racks:Cassandra中每个Datacenter里面每个机架里面的节点IP

Iterator\<Token\> tokenIter:Cassandra中从行记录Token开始的哈希环的虚拟节点分布

3.24-64行是Cassandra去选择合适的节点来分布副本的具体逻辑总结：在哈希环上以某行记录的Token为起点，顺时针遍历哈希环，直到分配好所有的副本

具体逻辑如下：

(1).从迭代器tokenIter中取出一个token，获取这个token所在的Datacenter，如果该Datacenter已经分好了足够的副本，那么跳过该轮逻辑处理

(2).接下来判断是不是在Datacenter所有的机架中都已经含有该记录的副本了，如果是，那么直接将一条副本记录直接放进当前遍历的节点中，如果不是，进行步骤3

(3).判断该节点的所在机架是不是第一次被遍历到，如果不是，那么暂时忽略该节点（因为进行到这一步，证明该Datacenter副本数不足，但是还有未被发现的机架，需要把副本放到还没发现的机架中）；如果是第一次被遍历到的话，直接将副本加入到该节点(机架)中

(4).如果说所有的Datacenter，所有的Rack中都已经含有副本了，并且当前副本数量不足的话，就从以前被暂时忽略的节点中按顺序挑选足够的节点，然后放入副本


4.返回结果

Cassandra中使用NetworkTopologyStrategy来分配副本的方法到这里就解释完毕了，当然在一致性哈希环里面也有别的分布副本的方式，大家要是有好的建议我们可以一起交流一下！

