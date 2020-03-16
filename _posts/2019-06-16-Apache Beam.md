---
layout: post
title: Apache Beam
date: 2019-06-16 16:42:31
categories: bigdata
share: y
excerpt_separator: <!--more-->
---


<!--more-->

# Apache Beam 

FlumeJava 的思想是将所有的数据都抽象成名为 PCollection（Parallel Collection）的数据结构，意思是可并行计算的数据集，和 RDD 十分相似。无论是从内存中读取的数据，还是在分布式环境下所读取的文件。

而 FlumeJava 在 MapReduce 框架中 Map 和 Reduce 思想上，抽象出 4 个了原始操作（Primitive Operation），分别是 parallelDo、groupByKey、 combineValues 和 flatten，让工程师可以利用这4 种原始操作来表达任意 Map 或者 Reduce 的逻辑。

同时，FlumeJava 的架构运用了一种 Deferred Evaluation 的技术，来优化我们所写的代码。你可以理解为 FlumeJava 框架会首先会将我们所写的逻辑代码静态遍历一次，然后构造出一个执行计划的有向无环图。这在 FlumeJava 框架里被称为 Execution Plan Dataflow Graph。有了这个图之后，FlumeJava 框架就会自动帮我们优化代码。例如合并一些 map 和 reduce。

FlumeJava 框架还可以通过我们的输入数据集规模，来预测输出结果的规模，从而自行决定代码是放在内存中跑还是在分布式环境中跑。

但是，FlumeJava 也有一个弊端，那就是 FlumeJava 基本上只支持批处理（Batch Execution）的任务，对于无边界数据（Unbounded Data）是不支持的。所以，Google 内部有着另外一个被称为 Millwheel 的项目来支持处理无边界数据，也就是流处理框架。

所以Dataflow Model就诞生了。2015年Google 公布了 Dataflow Model 的论文，同时也推出了基于 Dataflow Model思想的平台 Cloud Dataflow，但是只能在Google 的云平台上面运行。

Google 在 2016 年的时候联合了 Talend、Data Artisans、Cloudera这些大数据公司，基于 Dataflow Model 的思想开发出了一套 SDK，并贡献给了 Apache Software Foundation。



