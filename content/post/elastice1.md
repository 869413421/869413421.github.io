---
title: "ElasticSearch系列（集群内部原理）"
date: 2023-02-07T15:52:40+08:00
draft: false
description: "集群内部原理"
tags: ["ElasticSearch"]
categories: ["ElasticSearch"]
---

# 空集群

* #### ElasticSearch集群是什么？

  一个运行中的Es实例我们称之一个节点，一个集群是指由一个或多个有相同`cluster.name`的节点组合而成，集群中所有节点会共同负载和分担所有压力。当集群内新增或者删除节点时，集群会`重新平均分配`所有的数据到每个节点。

* #### 集群的主节点

  当一个运行中的节点被选举为主节点的时候，他会负责整个`集群内的所有变更`。例如`索引的增加和删除`，或者`增加或删除节点`。任何节点都可以成为主节点，但是主节点`不会负责文档级别的管理`。所以即使系统的压力怎么增加，主节点都不会成为性能的瓶颈。

* #### 操作时需要将请求发送到集群的哪一个节点？

  `因为每个节点都知道需要操作的文档所在的节点，并且节点会帮我们将请求发送到文档所在的节点当中`。所以我们需要对Es进行操作时，我们可以对集群中任意一个节点进行请求。

# 集群健康
  * 有时候我们需要对集群做一些监控,命令如下
```
   GET /_cluster/health
```
```
{
   "cluster_name":          "elasticsearch",
   "status":                "green", 
   "timed_out":             false,
   "number_of_nodes":       1,
   "number_of_data_nodes":  1,
   "active_primary_shards": 0,
   "active_shards":         0,
   "relocating_shards":     0,
   "initializing_shards":   0,
   "unassigned_shards":     0
}
```
`status` 字段指示着当前集群在总体上是否工作正常。它的三种颜色含义如下：

- green 所有的主分片和副本分片都正常运行
- yellow 所有的主分片都正常运行，但不是所有的副本分片都正常运行。
- red 有主分片没能正常运行
