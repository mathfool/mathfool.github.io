---
layout: post
title: "Clickhouse Day7"
date: 2021-05-27 10:30:00 +0800
categories: clickhouse
---

# LowCardinality

Clickhouse提供了一个数据类型叫LowCardinality. 简单的说这个就是把任意类型，在CH的内部转化为ENUM来进行存储，其实也没有那么任意，根据官方的文档Decimal就不行，而且对于8bit以下长度的类型，需要开启一个特殊的开关allow_suspicious_low_cardinality_types才能使用。

这个东西在clickhouse内部的结构差不多是这样的：
<div class="mermaid">
classDiagram
ColumnLowCardinality *-- Dictionary
ColumnLowCardinality *-- Index
ColumnLowCardinality *-- Positions
</div>

* Dictionary是一个字典，存的就是值
* Index是一个值到值的序号/ID的一个倒排索引。
* Positions就是每一行对应的值的Index

看到这里，就很容易理解为什么8bit一下长度的类型要确认才能使用。因为如果本身就没几个值，存值和存个index的开销就差不多了。

那么使用LowCardinality类型的好处，除了节省空间，还可以用到很多CH做的专门的有针对性的优化。比如[这片文章](https://github.com/yandex/clickhouse-presentations/raw/master/meetup19/string_optimization.pdf)中提到的关于string的一系列的优化。

# 使用场景

根据官方文档，推荐在值不超过10K个值的情形下使用，而对于超过100K个值的情形通常会有一定的性能损失。但网上也有同行把这个用到了10M，一样可以获得性能提升。不过这种做法应该就是用内存的开销来换查询的效率，毕竟OLAP动辄就要查成百上千个B那么多行的数据。

另外直觉上讲，这东西似乎只能用在dimension上，而用在metrics上似乎不妥。但是在我们的测试中，许多metrics列也可以省掉75%的存储空间。当然写入会慢大约30%左右。我们测试用的值类型是Int64，而估计字典的index的长度要大大小于这个，所以会省空间。但我们又必须保留int64因为确实理论上有可能会出现如此大的值。CH有这么个参数low_cardinality_max_dictionary_size，根据文档，这个参数配的是最大的字典记录数，那么超了这个字典并不会有什么问题，只是当做一个普通的值写入，而不是写入字典值的index。假设这个metrics大部分的值在10000以下，设low_cardinality_max_dictionary_size=10000，而恰好存了1-10000（不知道这个字典可不可以手动操作，low_cardinality_max_dictionary_size是不是可以按列指定），超过的反正也会直接记录，那么可能就会有一定的性能提升。不过感觉这个地方需要进一步的调研。

所以LowCardinality需要结合着那几个参数精细控制着使用。


