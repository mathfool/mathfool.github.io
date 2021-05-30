---
layout: post
title: "Clickhouse Day9"
date: 2021-05-30 10:30:00 +0800
categories: clickhouse
---

最近的一些博客有些内容参考了[虎哥的博客](https://bohutang.me/)。

# Clickhouse的pipeline

之前在调试clickhouse代码的时候，可以看到大量的pipelineExecutor对象。看起来是在Threadpool上获取thread，由pipelineExecutor去执行解析好的query。根据虎哥博客中的描述，这里应该是个有环图。找找代码中对应的部分应该大概是下面这个关系：
<div class="mermaid">
classDiagram
    QueryPipeline *-- Pipe
    PipelineExecutor *-- ExecutingGraph
    PipelineExecutor *-- Processors
    Processors <|-- AllKindsOfTransforms
    Processors <|-- AllKindsOfOutputFormats
    Processors <|-- SourcesAndSinks
    PullingAsyncPipelineExecutor *-- QueryPipeline
    PullingPipelineExecutor *-- QueryPipeline
    PullingPipelineExecutor *-- PipelineExecutor
    PullingAsyncPipelineExecutor *-- Data
    Data *-- PipelineExecutor
    Pipe *-- Processors
    class Processors{
        InputPorts
        OutputPorts
    }
</div>
这里面有三类主要的对象。Processor，也叫Transform，包装具体的处理逻辑。ExecutingGraphe包装了这些processor之间的e关系。PipelineExecutor负责根据这个有环图，来执行各种processor/transform。这个图上看感觉像是有两条组装的路线，不知道Pipe->QueryPipeline那边儿管啥。虎哥的博客上说processor负责执行pipeline，但上面的结构看起来是PipelineExecutor负责执行。

Processors里面有两个对象，一个是InputPort，一个是OutputPort。一个负责输入，一个负责输出。如果只有output，就是source。相应的如果只有input，就是sink。还有一些列通用的操作，具体可以建IProcessor.h。

根据虎哥的博客，这一系列设计来自一片论文:[Naiad: A Timely Dataflow System](https://dl.acm.org/doi/pdf/10.1145/2517349.2522738)。在ConnectedPapers查了一下，很多数据处理框架都引用了这片文章。比如Spark的[RDD](https://www.cs.princeton.edu/courses/archive/fall13/cos518/papers/spark.pdf)，又比如[Storm](http://cs.brown.edu/courses/cs227/archives/2015/slides/week6/6-storm-harsha.pdf)，再比如[Flink](https://www.researchgate.net/profile/Asterios-Katsifodimos/publication/305869785_Apache_Flink_Stream_Analytics_at_Scale/links/5a099320a6fdcc65eab5810a/Apache-Flink-Stream-Analytics-at-Scale.pdf)。还有一片引用量巨大的文章叫[Pregal](https://blog.acolyer.org/2015/05/26/pregel-a-system-for-large-scale-graph-processing/)不知道干啥的。

# 番外 - poco是个什么库

根据官方的说法：
>The POCO C++ Libraries are powerful cross-platform C++ libraries for building network- and internet-based applications that run on desktop, server, mobile, IoT, and embedded systems.
