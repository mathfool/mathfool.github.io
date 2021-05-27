---
layout: post
title: "Clickhouse Day8"
date: 2021-05-27 10:30:00 +0800
categories: clickhouse
---

# 接着看写入一个part
```C++
void MergeTreeDataPartWriterWide::write(const Block & block, const IColumn::Permutation * permutation)
{
...
    auto granules_to_write = getGranulesToWrite(index_granularity, block.rows(), getCurrentMark(), rows_written_in_last_mark);
```
根据表的granularity setting把数据block切成granules
```C++
    auto offset_columns = written_offset_columns ? *written_offset_columns : WrittenOffsetColumns{};
```
不知道这个offsetcolumns是干啥的
```C++
    Block primary_key_block;
    if (settings.rewrite_primary_key)
        primary_key_block = getBlockAndPermute(block, metadata_snapshot->getPrimaryKeyColumns(), permutation);
```
这里这个permutation没太看明白是干啥的，好像是把列换了换顺序。

```C++
    Block skip_indexes_block = getBlockAndPermute(block, getSkipIndicesColumns(), permutation);
```
这里这个skip index相当于secondary index。不过clickhouse的skip index可以弄个表达式，比如说有一列是string，你的查询如果会经常用到where length(string)>xxx这种，那么你就可以建一个min_max类型的index，把(length(string))作为他的index。还有一些bloomfilter类型的index，对于很多场景也很有用。具体可见[官方文档](MergeTreeData::MutableDataPartPtr MergeTreeDataWriter::writeTempPart)的例子
```C++
    auto it = columns_list.begin();
    for (size_t i = 0; i < columns_list.size(); ++i, ++it)
    {
        const ColumnWithTypeAndName & column = block.getByName(it->name);

        if (permutation)
        {
            if (primary_key_block.has(it->name))
            {
                const auto & primary_column = *primary_key_block.getByName(it->name).column;
                writeColumn(*it, primary_column, offset_columns, granules_to_write);
            }
            else if (skip_indexes_block.has(it->name))
            {
                const auto & index_column = *skip_indexes_block.getByName(it->name).column;
                writeColumn(*it, index_column, offset_columns, granules_to_write);
            }
            else
            {
                /// We rearrange the columns that are not included in the primary key here; Then the result is released - to save RAM.
                ColumnPtr permuted_column = column.column->permute(*permutation, 0);
                writeColumn(*it, *permuted_column, offset_columns, granules_to_write);
            }
        }
        else
        {
            writeColumn(*it, *column.column, offset_columns, granules_to_write);
        }
    }
```
之后会分别把primary_key, skip_indexes,和非primary的列
写出去，这里写的是列。这里有个疑问，为啥要分3个if写而不是直接写了。感觉除了else里面的做了个permute，primary和skip_index并没有什么特殊的处理。至少那俩可以合起来。
```C++
    if (settings.rewrite_primary_key)
        calculateAndSerializePrimaryIndex(primary_key_block, granules_to_write);

    calculateAndSerializeSkipIndices(skip_indexes_block, granules_to_write);

    shiftCurrentMark(granules_to_write);
}
```
这里轮到写primary key和index文件。在writeColumn里面是一个granule一个granule写的。每写一个granule，更新一次mark。