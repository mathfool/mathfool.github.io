---
layout: post
title: "Clickhouse Day4"
date: 2021-05-10 10:30:00 +0800
categories: clickhouse
---

# Moving目录

clickhouse可以配置多个Volumn或者多个磁盘，再配合不同的策略可以实现冷热数据分别存储。今天刚好有个人问为什么目录里面有个folder交moving。这个moving就像tmp_insert或者merge一样，是数据在两个volumn/disk之间移动时先创建的临时目录。

# 一个bug
官方的Github上今天有人提交了一个bug，简单的说就是建一个表，其中一列是Array<Tuple<String, UINT64>>类型的。插入10万行，然后查询没问题，重启server，然后Array的那列就为空了。很难想象居然有这么重大的bug。调试了一下代码，大概情形差不多就是下面这个图描述的。

<div class="mermaid">
graph TD
    A[Create Table] --> B[Write Data]
    A -->|correct column name| C[Update In Memory Metadata]
    B --> D[./Data]
    B --> E[./Meta]

    F[System Start] --> G[Load Meta] 
    H[./Meta] --> |Wrong column name| G
</div>

如图，第一次之所以没有问题，应该是在建表的时候直接用某种方式更新了内存中的metadata_snapshot。而server重启之后，是从磁盘上的meta重建的metadata_snapshot。Somehow在重建列名的时候，直接创建的会用类型来命名类，这点和磁盘上文件的命名方式一致。而重启服务器之后，列名会变成1，2这样的类似列的index。那种正确现在说不好，反正后一种看起来是无法找到正确的文件，或者无法读取正确的记录。

## 建表是在哪里更新的metadata_snapshot
## 重启服务器是在哪里更新的metadata_snapshot