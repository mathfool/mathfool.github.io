---
layout: post
title: "Clickhouse Day5"
date: 2021-05-22 10:30:00 +0800
categories: clickhouse
---

# build 20.10
本来想靠逻辑推理搞定这个bug，还是放弃了。最终还是checkout了20.10重新build。发现了一个重要的不同。用最新的版本运行完建表的sql，表名是长这个样子的，不知道为啥会用类型名做了表名。
```
-rw-r-----  1 lqin  staff    30008 May 22 00:07 array%2E%27string%27.bin
-rw-r-----  1 lqin  staff     2976 May 22 00:07 array%2E%27string%27.mrk2
-rw-r-----  1 lqin  staff  4005014 May 22 00:07 array%2Enumber.bin
-rw-r-----  1 lqin  staff     2976 May 22 00:07 array%2Enumber.mrk2
-rw-r-----  1 lqin  staff    36169 May 22 00:07 array.size0.bin
-rw-r-----  1 lqin  staff     2976 May 22 00:07 array.size0.mrk2
```
而用旧版本创建出来的，表名是长这样的，这里显然是新版本的有问题，毕竟类型是可以重复的。
```
-rw-r-----  1 lqin  staff    30008 May 21 23:09 array%2E1.bin
-rw-r-----  1 lqin  staff     2976 May 21 23:09 array%2E1.mrk2
-rw-r-----  1 lqin  staff  4005014 May 21 23:09 array%2E2.bin
-rw-r-----  1 lqin  staff     2976 May 21 23:09 array%2E2.mrk2
-rw-r-----  1 lqin  staff    36169 May 21 23:09 array.size0.bin
-rw-r-----  1 lqin  staff     2976 May 21 23:09 array.size0.mrk2
```
所以这里可能有两个地方有问题，一个就是解析语法树的地方除了问题，解析出了两个新的嵌套的列名。或者就是存储的时候不知道为啥用了新的列名。

额，看着看着发现，似乎CH有逻辑来处理这种列名冲突。
```C++
        /*
         * Generate a column name that is not present in the sample block, using
         * the given prefix and an optional numeric suffix.
         */
        String getUniqueName(const String & prefix)
        {
            auto result = prefix;

            // First, try the name without any suffix, because it is currently
            // used both as a display name and a column id.
            while (hasColumn(result))
            {
                result = prefix + "_" + toString(next_unique_suffix);
                ++next_unique_suffix;
            }

            return result;
        }
```

# 回过头来再看看问题出在哪里

<div class = "mermaid">
 graph TD
    A[Nested Select] -->|Extract Type Info| B(Create Table)
    B -->|Based on Column| C(Update In-Memory Catalog)
    B -->|Based on Column| D(Generate on-disk files)
    D -->|Based on attach table sql| E(load table on server restart)
</div>
根据上面的图，可以确定的是下面的两个分叉出现了某种形式的不一致。分析来看大概率是select语句生成的类型和存在磁盘上文件的attach table语句生成的列数据类型不一致。看来想要段时间内解决这个问题是不太可能了。难道要从解析AST语法树看起。