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
# Clickhouse Insert语句的写入过程

# 番外：C++11的shared_ptr，右值引用和move

想我当年写C++的时候，还是auto_ptr的时代，右值引用也还不知道是个什么东西。

shared_ptr看起来就是给对象加上了引用计数，当引用计数归零的时候销毁对象。还有另外的一种职能指针叫unique_ptr，相当于shared_ptr且引用计数<=1。不能被复制，只能被转移(move)。
那右值引用又是个啥呢(Rvalue Reference)? 书里面是这么写的
> In the C++ lingo, the temporaries are called Rvalues because they can only appear on the right side in an assignment.
看起来，Rvalues就是临时变量啊。
    
那std::move是干啥的呢？简单的说就是std::move了之后的object，被宣布为没用了，通常情况下，里面的值也会被清理掉或者是置为某种初始的状态。那么作为开发者，通常会重用传入对象已有的数据去构造新的对象。std::move的前因后果，使用场景和具体实现有人写了一本书，叫《C++ Move Semantics - The Complete Guide》，作者是Nicolai M. Josuttis. 这哥们是C++标准委员会的成员，写了一堆C++相关的书籍。这本书又大量的实例，通俗易懂。

    
关于右值引用和move，注意下面的语句：
```
T&& b = a
```
那再使用的时候还是要move一下，因为b被表示出来了，就变成了左值。所以上面那段outputstream的代码里面`out_streams.emplace_back(std::move(out))`中的std::move并不能
去掉。但是这段代码可以改写成`out_streams.emplace_back(std::move(out_wrapper))`，原代码不知道为啥要写成那个样子。
