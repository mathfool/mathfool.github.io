---
layout: post
title: "Clickhouse Day2"
date: 2021-05-19 10:30:00 +0800
categories: clickhouse
---

# Clickhouse的存储
昨天说到数据写入的基本数据结构是Block。那么这个数据block写到磁盘上又是啥样子呢？调试代码发现数据是由MergeTreeDataPartWriterOnDisk写入的，这个东西有两个具体的实现，一个叫compact，一个叫wide

<div class="mermaid">
classDiagram
IMergeTreeDataPartWriter <|-- MergeTreeDataPartWriterOnDisk
MergeTreeDataPartWriterOnDisk <|-- MergeTreeDataPartWriterCompact
MergeTreeDataPartWriterOnDisk <|-- MergeTreeDataPartWriterWide
</div>

根据文档，这两种格式可以通过下面的参数来控制。也就是说，如果不设这些参数，默认就是wide，那么每一个insert，将产生一个文件。如果设了，那么可以将一堆小的插入写在一起。（顺序咋办？）
> Data storing format is controlled by the min_bytes_for_wide_part and min_rows_for_wide_part settings of the table engine. If the number of bytes or rows in a data part is less then the corresponding setting's value, the part is stored in Compact format. Otherwise it is stored in Wide format. If none of these settings is set, data parts are stored in Wide format.

进一步看代码，在20.8.1.75版本增加了一个新的参数，可以再设一个叫min_bytes_for_compact_part，这样似乎是会写到memory里, 在第一次merge的时候，再写到磁盘。这个过程非常类似HBASE的compaction。总的来说这一类的数据库都是这个过程，比如rocksdb也是这样。Clickhouse也实现了一个WAL，在写in-memory的同时写这个WAL，这样clickhouse重新启动的时候会用这个WAL重建这个in-memory的block。如果再想提升一丢丢的性能，可以把WAL关了，就是会有丢数据的风险。嗯不过感觉这个东西还不是太稳定比如[这个issue]
(https://github.com/ClickHouse/ClickHouse/issues/17758)

