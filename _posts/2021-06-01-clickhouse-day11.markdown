---
layout: post
title: "Clickhouse Day11"
date: 2021-06-01 10:30:00 +0800
categories: clickhouse
---

# Replicated表的写入

1. 在本地创建temp block，如果开启了deduplicate，计算blockid，就是partitionid+hashvalue，这个会写入zk的block节点。
2. commit这个block到zookeeper。此处的各种情况处理非常复杂，具体逻辑在ReplicatedMergeTreeBlockOutputStream::commitPart。

# Replicated表的同步

这段是看代码加上自己的一些猜想，不一定对
1. 每一个StorageReplciatedMergeTree对象，会构建一组task，用来监听对应表的对应shard在zookeeper的log节点中的新任务。同时在对应/replica节点下，会有个log_pointer用来存当前已经“处理”到的log记录，有点儿像kafka的offset。
2. 这里的“处理”并不是真的处理，而是把这个任务拿过来放replica的queue中。比如一个merge任务可能就长下面的这个e样子：
```
format version: 4
create_time: 2021-06-01 07:05:08
source replica: clickhouse4
block_id:
merge
20210601_54760_54760_0
20210601_54761_54761_0
20210601_54762_54762_0
20210601_54763_54763_0
20210601_54764_54764_0
into
20210601_54760_54764_1
deduplicate: 0
part_type: Compact 
```
3. 再由StorageReplciatedMergeTree中的backgroundexecutor去执行里面具体的task。比如上面的那个merge，最终会由`StorageReplicatedMergeTree::tryExecuteMerge(const LogEntry & entry)`来执行。这里面有个配置，如果配置了always_fetch_merged_part，那么merge不回被执行，而是直接去取merge好了的part。不过在这个task里面是直接返回了，不知道取回merge结果的那个是不是也是由master创建好了，这边儿直接fetch过来，如果没配置那个参数的话就会报一个duplicate block？这都是我猜的。

# Replicated表的读取

如果没有配置select_sequential_consistency，那就基本上一样。如果配置了select_sequential_consistency，那么读取的时候就回去zookeeper看看该有的block是不是都有。如果Quorum里面有，但part不全，就抛异常。如果有的part还没写入quorum，那么就不读。