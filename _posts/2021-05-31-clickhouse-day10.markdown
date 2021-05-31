---
layout: post
title: "Clickhouse Day10"
date: 2021-05-31 10:30:00 +0800
categories: clickhouse
---

# Replicated中zookeeper的作用

clickhouse比较吸引我的原因之一就是它有大量的参数可供定制。比如你可以选择单副本，可以选择多副本。而且副本的配置可以不同。你可以让clickhouse帮你管理副本的同步，也可以选择自己upload两次。因此有的人形容clickhouse是个手动挡的跑车。

Clickhouse本身提供的副本管理是通过ReplicatedMergeTree这个引擎来实现的，其实是一系列的replicated引擎，就是将各种mergetree前面加了个Replicated。在大多数场景下，zookeeper实际上起到的是一个分布式队列的作用。当master有数据写入/merge等操作的时候，会讲这些操作写入zookeeper，相应的slave就会侦测到有新的任务然后在本地进行相应的操作。

在StorageReplicatedMergeTree.h的开头有一段话简要的描述了zookeeper的作用。在ZK里面的clickhouse节点下面会有两个字节点，一个叫Tables，一个叫task_queue。这个taskqueue里面存的是ddl的各种任务。另外的tables里面按shard存的表相关的信息，结构类似于`/clickhouse/tables/{shard}/{table}`，只有用到Replicated引擎的表才会出现在这里。对于每一个table，节点如下：
```
metadata: 这里面感觉就是存了engine的几个参数，index granularity，primary key，partitionkey等
temp：不知道是啥
mutations：根据comments，这里似乎放的是ALTER DELETE, ALTER UPDATE，那和对应的ddl里面的内容不知道有没有关系。
log：操作日志
leader_election：顾名思义
columns：无须解释
blocks：最近插入的block信息以及他们的checksum
nonincrement_block_numbers：不知道是啥，根据comments，这个貌似没用。
replicas：replica的server相关信息，以及是否活着在线。另外在parts字节点下有parts的header，和磁盘上的目录一一对应。干啥的不知道。
quorum：不知道是啥
block_numbers：存在的partitions/block信息
```

# System中的zookeeper表

如果你的clickhouse配置的zookeeper，那么在system数据库中会出现zookeeper表，这个表相当于给你提供了一个嵌入式的zookeeper cli。

