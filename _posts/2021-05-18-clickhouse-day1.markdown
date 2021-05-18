---
layout: post
title: "Clickhouse Day1"
date: 2021-05-18 10:30:00 +0800
categories: clickhouse
---

这里应该写点儿前言吧，留着以后再写。

# Clickhouse中主要的类和他们之间的相互关系

<div class="mermaid">
classDiagram
IStorage <|-- MergeTreeData
MergeTreeData <|-- SttorageMergeTree
MergeTreeData <|-- StorageReplicatedMergeTree
MergeTreeData <|-- SomeOtherStorageClsses
Block *-- ColumnsWithTypeAndName
ColumnsWithTypeAndName *-- IColumn
ColumnsWithTypeAndName *-- IDataType
IColumn *-- COW
</div>

Clickhouse是一个典型的列存储，面向OLAP场景的数据库系统。这一点从它底层storage的设计上就可见一斑。Clickhouse数据处理的基本单元是Block，这个block内部由一系列的column组成，由ColumnsWithTypeAndName来表示。其中每个column可以简单的理解为<列名，类型，数据>的一个三元组。在类型DataType中包含了对这个类型进行序列化和反序列化的相关方法。
```
class Block
{
private:
    using Container = ColumnsWithTypeAndName;
    using IndexByName = std::unordered_map<String, size_t>;

    Container data;
    IndexByName index_by_name;

```
```
struct ColumnWithTypeAndName
{
    ColumnPtr column;
    DataTypePtr type;
    String name;
}
```
而为了操作这些具体的存储，Clickhouse又定义了一些可以串起来使用的Input/Output Streams。很像一些大数据处理框架的算子。
<div class="mermaid">
classDiagram
IBlockOutputStream <|-- CountingBlockOutputStream
IBlockOutputStream <|-- AddingDefaultBlockOutputStream
IBlockOutputStream <|-- CountingBlockOutputStream
IBlockOutputStream <|-- CheckConstraintsBlockOutputStream
IBlockOutputStream <|-- MergeTreeBlockOutputStream
IBlockOutputStream <|-- PushingToViewsBlockOutputStream
IBlockOutputStream <|-- ReplicatedMergeTreeBlockOutputStream
</div>
比如在Clickhouse在执行Insert语句写数据的时候，就会将上面的几个OutputStream串起来使用
    
```
BlockOutputStreamPtr out;
out = std::make_shared<PushingToViewsBlockOutputStream>(table, metadata_snapshot, context, query_ptr, no_destination);
out = std::make_shared<CheckConstraintsBlockOutputStream>(query.table_id, out, out->getHeader(), metadata_snapshot->getConstraints(), context);
out = std::make_shared<AddingDefaultBlockOutputStream>(out, query_sample_block, out->getHeader(), metadata_snapshot->getColumns(), context);
out = std::make_shared<SquashingBlockOutputStream>(
                    out,
                    out->getHeader(),
                    context.getSettingsRef().min_insert_block_size_rows,
                    context.getSettingsRef().min_insert_block_size_bytes);
            }

auto out_wrapper = std::make_shared<CountingBlockOutputStream>(out);
out_wrapper->setProcessListElement(context.getProcessListElement());
out = std::move(out_wrapper);
out_streams.emplace_back(std::move(out));
```
    
# 番外：C++11的右值引用和Move是个什么东西

