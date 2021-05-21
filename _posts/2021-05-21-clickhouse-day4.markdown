---
layout: post
title: "Clickhouse Day4"
date: 2021-05-10 10:30:00 +0800
categories: clickhouse
---


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

# clickhouse如何建立的metadata

然后又看了一下生成metadata的方式。不管是server重启还是创建表，clickhouse都是根据create table语句去生成AST语法树（这部分很多bug，有机会在写）。目前看起来这两种情形下创建出来的语法树都是对的。后面再看看从这个语法树到table中的column对象有什么问题。

# 和metadata相关的一些类

<div class = "mermaid">
classDiagram
IDatabase <|-- DatabaseDictionary
IDatabase <|-- DatabaseWithOwnTablesBase
DatabaseWithOwnTablesBase <|-- DatabaseMemory
DatabaseWithOwnTablesBase <|-- DatabaseOnDisk
DatabaseOnDisk <|-- DatabaseLazy
DatabaseOnDisk <|-- DatabaseWithDictionaries
DatabaseWithDictionaries <|-- DatabaseOrdinary
DatabaseOrdinary <|-- DatabaseAtomic

DatabaseWithOwnTablesBase *-- Tables
StorageFactory --Tables:Create
</div>
在系统启动的时候，数据库根据存储的meta信息（主要是建数据库和表的sql），来创建AST语法树，StorageFactor会根据sql中定义的引擎来创建对应的storage对象。

特别关注了一下建立columns的过程。现实解析了column相关的语法树的最上层，然后调用`ColumnsDescription::flattenNested`来递归的处理嵌套的结构。

而在建表的时候，调用的是InterpreterCreateQuery::execute()来做建表的操作。
看了一下重新启动加载表创建的columns和在client端创建的columns的区别：

## 系统启动创建的array列
```
 [1] = {std::__1::shared_ptr<DB::IAST>} std::__1::shared_ptr<DB::IAST>::element_type @ 0x000000013cf5c398 strong=1 weak=2
   __ptr_ = {DB::ASTColumnDeclaration * | 0x13cf5c398} 0x000000013cf5c398
    DB::IAST = {DB::IAST} 
     std::__1::enable_shared_from_this<DB::IAST> = {std::__1::enable_shared_from_this<DB::IAST>} 
     children = {DB::ASTs} size=1
      [0] = {std::__1::shared_ptr<DB::IAST>} std::__1::shared_ptr<DB::IAST>::element_type @ 0x000000013cf5c478 strong=2 weak=2
       __ptr_ = {DB::ASTFunction * | 0x13cf5c478} 0x000000013cf5c478
        DB::ASTWithAlias = {DB::ASTWithAlias} 
         DB::IAST = {DB::IAST} 
          std::__1::enable_shared_from_this<DB::IAST> = {std::__1::enable_shared_from_this<DB::IAST>} 
          children = {DB::ASTs} size=1
         alias = {DB::String} ""
         prefer_alias_to_column_name = {bool} false
        name = {DB::String} "Array"
        arguments = {DB::ASTPtr} std::__1::shared_ptr<DB::IAST>::element_type @ 0x000000013cfa51f8 strong=2 weak=2
        parameters = {DB::ASTPtr} nullptr
        is_window_function = {bool} false
        window_name = {DB::ASTPtr} nullptr
        window_partition_by = {DB::ASTPtr} nullptr
        window_order_by = {DB::ASTPtr} nullptr
        no_empty_args = {bool} true
    name = {DB::String} "array"
    type = {DB::ASTPtr} std::__1::shared_ptr<DB::IAST>::element_type @ 0x000000013cf5c478 strong=2 weak=2
    null_modifier = {std::__1::optional<bool>} Has Value=false
    default_specifier = {DB::String} ""
    default_expression = {DB::ASTPtr} nullptr
    comment = {DB::ASTPtr} nullptr
    codec = {DB::ASTPtr} nullptr
    ttl = {DB::ASTPtr} nullptr
```

## create table创建的array列
```
[1] = {std::__1::shared_ptr<DB::IAST>} std::__1::shared_ptr<DB::IAST>::element_type @ 0x000000013c29f058 strong=1 weak=2
 __ptr_ = {DB::ASTColumnDeclaration * | 0x13c29f058} 0x000000013c29f058
  DB::IAST = {DB::IAST} 
   std::__1::enable_shared_from_this<DB::IAST> = {std::__1::enable_shared_from_this<DB::IAST>} 
   children = {DB::ASTs} size=0
  name = {DB::String} "array"
  type = {DB::ASTPtr} std::__1::shared_ptr<DB::IAST>::element_type @ 0x000000013c29f138 strong=1 weak=2
  null_modifier = {std::__1::optional<bool>} Has Value=false
  default_specifier = {DB::String} ""
  default_expression = {DB::ASTPtr} nullptr
  comment = {DB::ASTPtr} nullptr
  codec = {DB::ASTPtr} nullptr
  ttl = {DB::ASTPtr} nullptr
```
发现在create table创建的array那列，并没有创建子成员。而在系统启动时创建的这个array列，包括整个array->tuple(string, uint64)的结构。不知道是不是select * 的结果。而在create table的时候，并没有走到flattenNested那个逻辑。

