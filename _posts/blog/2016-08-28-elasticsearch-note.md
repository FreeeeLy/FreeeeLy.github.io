---
layout: post
title: 小记关于Elasticsearch日常使用中的一事
description: 小记关于Elasticsearch日常使用中的一事
category: blog
---

在使用Elasticsearch的过程中，如果elasticsearch建立的分片数较多，数据量较大的情况下，贸然将集群重启的话，会导致集群在重启之后shard进行大规模的recovery，根据数据量和shard情况可能会有长达数小时甚至更长的恢复时间，造成大量带宽的开销和增大集群的压力。因此如果有计划需要将集群进行重启，建议：

一.暂时切断集群与外部的通信，不再接受新的读写请求

二.执行[synced flush](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-synced-flush.html#indices-synced-flush)操作,进行primary shard 和 replica shard的同步，这个操作一般进行的很快，因为在正常的情况下，primary shard 和 relica shard 基本是一致的，但是如果某个shard上面正在处理读写请求，那么在这个shard上面的同步是会失败的，因此最后在进行同步之前切断读写请求

三.重启集群，你会发现当集群节点都加入进来时，基本上所有shard都不会进行recovery操作（因为已经有同步标记了），这样子比起直接重启集群，要快上很多
