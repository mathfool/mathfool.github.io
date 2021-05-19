---
layout: post
title: "Clickhouse Day3"
date: 2021-05-20 10:30:00 +0800
categories: clickhouse
---

接着看写入的过程这部分，回顾一下前面看的几个主要的泪之间的关系

<div class="mermaid">
classDiagram
IStorage <|-- MergeTreeData
MergeTreeData *-- IMergeTreeDataPart
MergeTreeData <|-- StorageMergeTree
MergeTreeData <|-- ReplicatedStorageMergeTree
StorageMergeTree *-- MergeTreeDataSelectExecutor
StorageMergeTree *-- MergeTreeDataWriter
StorageMergeTree *-- MergeTreeDataMergerMutator
StorageMergeTree *-- BackgroundJobsExecutor
StorageMergeTree *-- BackgroundMovesExecutor

MergeTreeBlockOutputStream *-- StorageMergeTree
ReplicatedMergeTreeBlockOutputStream *-- ReplicatedStorageMergeTree
IBlockOutputStream <|-- ReplicatedMergeTreeBlockOutputStream
IBlockOutputStream <|-- MergeTreeBlockOutputStream
IBlockOutputStream <|-- IMergedBlockOutputStream
IMergedBlockOutputStream <|-- MergedBlockOutputStream
IMergeTreeDataPartWriter *-- MergeTreeData
IMergedBlockOutputStream *-- MergeTreeData
class  IMergedBlockOutputStream{
    MergeTreeData
    StorageMetadataPtr
    IMergeTreeDataPart::MergeTreeWriterPtr
}
class MergedBlockOutputStream{
    NamesAndTypesList;
    IMergeTreeDataPart::MinMaxIndex;
    rows_count;
    CompressionCodecPtr;
}
</div>

最终的写入部分代码是在`MergeTreeBlockOutputStream`中完成的。
```C++
void MergeTreeBlockOutputStream::write(const Block & block)
{
    auto part_blocks = storage.writer.splitBlockIntoParts(block, max_parts_per_block, metadata_snapshot);
    for (auto & current_block : part_blocks)
    {
        Stopwatch watch;

        MergeTreeData::MutableDataPartPtr part = storage.writer.writeTempPart(current_block, metadata_snapshot, optimize_on_insert);
        storage.renameTempPartAndAdd(part, &storage.increment);

        PartLog::addNewPart(storage.global_context, part, watch.elapsed());

        /// Initiate async merge - it will be done if it's good time for merge and if there are space in 'background_pool'.
        storage.background_executor.triggerTask();
    }
}
```
- 第一步是把block根据partition分成几个子的block。
- 第二部是把block相关的内容持久化道一个temp目录，大概长这个样子temp_insert_partition_id-minblock-maxblock-level，之后再move到正常的目录名。Merge的时候会有类似的操作，只是临时目录的名字会是merge_开头。
 - 这个具体的写入过程是在`MergeTreeDataWriter::writeTempPart`中写入的。
 - 先试对block进行了诸如sort之类的操作，然后构建了datapart对象，然后再利用这个datapart对象构建了MergedBlockOutputStream。
 - 根据上面的图，可以看到在MergedBlockOutputStream以及他的父类，也就是那个IMergedBlockOutputStream接口中，可以找到磁盘文件上每一个文件对应的对象。
- 之后是记上加了新的part的log
- 最后是看看是不是需要执行一下merge。根据这里的comments，应该是后台有个pool，如果能执行就执行，执行不了就算了。


# 番外：很多类都集成的std::enable_shared_from_this是个啥

这个是为了shared_ptr的对象可以在多个智能指针下共享。
```C++
struct Good : std::enable_shared_from_this<Good> // 注意：继承
{
public:
	std::shared_ptr<Good> getptr() {
		return shared_from_this();
	}
	~Good() { std::cout << "Good::~Good() called" << std::endl; }
};
//some code
std::shared_ptr<Good> gp1 = std::make_shared<Good>();
std::shared_ptr<Good> gp2 = gp1->getptr();
```
上面的代码如果不继承enable_shared_from_this就会造成两次析构而崩溃。

#问题：detached目录是干啥的

