---
layout: post
title: "Clickhouse Day6"
date: 2021-05-26 10:30:00 +0800
categories: clickhouse
---

# 找到一点证据

在读文件的时候，clickhouse的确是按照老版本的命名方式在取bin文件，奇怪的是并没有报错。
```C++
bool data_file_exists = data_part->checksums.files.count(stream_name + DATA_FILE_EXTENSION);

/** If data file is missing then we will not try to open it.
  * It is necessary since it allows to add new column to structure of the table without creating new files for old parts.
  */
if (!data_file_exists)
    return;
```
这里取的名字分别是"array%2E1"和"array%2E2".

尝试分开create table和insert，发现生成的attach table语句时正确的。而且文件名也是用E1/E2这样的格式来命名的。看起来这个版本是正确的了。看来又是改SQL parser这块儿改的问题。Clickhouse的SQL Parser写的实在是太烂了。之前我fix的的那个alias相关的bug就是parser而且还有一大堆没fix issue。

# 文件名生成

文件名生成在IDataType
```C++
String IDataType::getFileNameForStream(const String & column_name, const IDataType::SubstreamPath & path)
{
    ...
    stream_name += "%2E" + escapeForFileName(elem.tuple_element_name);
    ...
}

```
这里这个tuple_element_name就是1和2。而错误的版本这里填的是'string'和number。接近真相了。