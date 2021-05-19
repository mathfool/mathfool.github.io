---
layout: post
title: "Clickhouse Day2"
date: 2021-05-19 10:30:00 +0800
categories: clickhouse
---

# Clickhouse的存储
昨天说到数据写入的基本数据结构是Block。那么这个数据block写到磁盘上又是啥样子呢？调试代码发现数据是由MergeTreeDataPartWriterOnDisk写入的，这个东西有两个具体的实现，一个叫compact，一个叫wide。相对应的，每一种DataPart都对应一个DataPartWriter

<div class="mermaid">
classDiagram
IMergeTreeDataPartWriter <|-- MergeTreeDataPartWriterOnDisk
MergeTreeDataPartWriterOnDisk <|-- MergeTreeDataPartWriterCompact
MergeTreeDataPartWriterOnDisk <|-- MergeTreeDataPartWriterWide

IMergeTreeDataPart <|-- MergeTreeDataPartCompact
IMergeTreeDataPart <|-- MergeTreeDataPartMemory
IMergeTreeDataPart <|-- MergeTreeDataPartWide
</div>

<div class="mermaid">
classDiagram
IMergeTreeDataPart *-- IMergeTreeDataPartWriter
IMergeTreeDataPart *-- IMergeTreeDataPartReader
</div>

根据文档，这两种格式可以通过下面的参数来控制。也就是说，如果不设这些参数，默认就是wide，那么每一个insert，将产生一个文件。如果设了，那么可以将一堆小的插入写在一起。（顺序咋办？）
> Data storing format is controlled by the min_bytes_for_wide_part and min_rows_for_wide_part settings of the table engine. If the number of bytes or rows in a data part is less then the corresponding setting's value, the part is stored in Compact format. Otherwise it is stored in Wide format. If none of these settings is set, data parts are stored in Wide format.

进一步看代码，在20.8.1.75版本增加了一个新的参数，可以再设一个叫min_bytes_for_compact_part，这样似乎是会写到memory里, 在第一次merge的时候，再写到磁盘。这个过程非常类似HBASE的compaction。总的来说这一类的数据库都是这个过程，比如rocksdb也是这样。Clickhouse也实现了一个WAL，在写in-memory的同时写这个WAL，这样clickhouse重新启动的时候会用这个WAL重建这个in-memory的block。如果再想提升一丢丢的性能，可以把WAL关了，就是会有丢数据的风险。嗯不过感觉这个东西还不是太稳定比如
[这个issue](https://github.com/ClickHouse/ClickHouse/issues/17758)

# 磁盘上的文件
那么一次写入在磁盘上都会创建哪些文件？首先这个写入会创建一个目录，名字由partition_id-minblock-maxblock-level组成。随着后台的merge发生，后面三个部分的值会发生变化。在这个目录里面会有这么几个文件：

* count.txt 存的是当前目录中的行数
* columns.txt 存的是列名和他们对应的类型
* checksums.txt 目录里面所有文件的checksum
* default_compression_codec.txt 默认的codec
* primary.idx primary index文件
* [Column].bin - 每列的具体数据
* [Column].mrk - marks文件。用来帮助快速检索数据。
* partition.dat - 好像存的是和partition相关的数据，不知道是个啥
* min_max_[Column].idx - 这个存的是当前目录partition key的最大最小值

那么在数据量不足够的情况下，有可能所有的列都会写入

* data.bin
* data.mrk

# 问题

* 今天发现我们自己数据库存domain的列和存device的列的size居然是一个数量级，明天可以研究研究是为了啥。
经过调查，发现device应该使用LowCardinality数据类型。根据文档，如果cardinality小于10000，那么就应该使用low cardinality。如果大于100000，那么就会有性能损失，看来可以在这方面做一些优化。这里有个小发现，数据库的默认设置下，有些类型比如int64不允许被定义为low cardinality。但是可以通过打开allow_suspicious_low_cardinality_types来允许。对于我们的case，通过改变设置，大约可以减少六成的存储。

* 线上似乎并没有配min_bytes_for_wide_part和min_rows_for_wide_part，但是也有一些folder里面的数据呗合并写入了data.bin/data.mrk。最小的只有四五百行，而稍大一些的油279416026。
这里的原因是因为在4个控制选择什么样的存储方式的选项，只有min_bytes_for_wide_part又一个default的值：10485760。可以看看选择的代码，比较的简单明了
```C++
MergeTreeDataPartType MergeTreeData::choosePartType(size_t bytes_uncompressed, size_t rows_count) const
{
    const auto settings = getSettings();
    if (!canUsePolymorphicParts(*settings))
        return MergeTreeDataPartType::WIDE;

    if (bytes_uncompressed < settings->min_bytes_for_compact_part || rows_count < settings->min_rows_for_compact_part)
        return MergeTreeDataPartType::IN_MEMORY;

    if (bytes_uncompressed < settings->min_bytes_for_wide_part || rows_count < settings->min_rows_for_wide_part)
        return MergeTreeDataPartType::COMPACT;

    return MergeTreeDataPartType::WIDE;
}
```
* 系统设置在哪里查，可以查system.settings表。Mergetree的设置在merge_tree_settings表中。另外MergeTree引擎的设置可以针对不同的表做改变，在建表的时候通过settings传入，具体可以看
[这个页面](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree/)

