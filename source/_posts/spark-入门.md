---
title: Spark 学习入门
date: 2017-12-08 20:40:00
categories: BigData
tags:
- Spark

---

## 什么是spark

spark 是一个快速的、通用的大数据处理平台。

<!--more-->

## spark 的四大特性

### 快速性

在内存中运行程序比 hadoop mapReduce 快 100 倍， 比在磁盘中快 10 倍。

### 易用性

支持 Java，Scala，Python，R 等编程语言。支持 python，scala，R 的 shell 交互。

### 通用性

由 SQL，streaming，和负责的复杂的分析。

### 运行在任何地方

Spark 可以运行在 hadoop，Mesos，standalone 或者在云端。它可以访问不同的数据源，包括 HDFS,Cassandra,HBase,Hive和S3


## Spark 四种部署模式

### hadoop

spark on yarn 用 yarn 资源管理器来管理 spark 的资源，这也是国内用的最多的模式。

### mesos

也是类似于 yarn 的资源管理器，但是这个资源管理器，在国内用的不多，大多数还是在国外使用。

### standalone

spark 自己来管理资源，也是用的比较多的一种模式。

### cloud

部署到云端

## quick start

```
val textFile = sc.textFile("README.md");  
// textFile: org.apache.spark.rdd.RDD[String] = README.md MapPartitionsRDD[3] at textFile at <console>:27

textFile.count() // 返回RDD对象的items数，即有多少行。
// res: Long = 3

textFile.first() // 返回RDD对象的第一个item，即第一行
// res: String=hello world !

// 使用filter转换，返回一个新的RDD
val linesWithJump = textFile.filter(line => line.contains("jump"))
// linesWithJump: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[4] at filter at <console>:29

// 链式操作，转化+求和
val count = textFile.filter(line => line.contains("jump")).count()

```

### 创建第一个 scala 程序

```
package top.ljming.insist.model

import org.apache.spark.{SparkConf, SparkContext}

/**
  * @author lijiangming
  */

object SimpleApp {

  def main(args: Array[String]) {

    val reademeFile = "D:\\develop-tools\\spark-1.6.1-bin-hadoop2.6\\README.md"
    val conf = new SparkConf().setAppName("Simple Application")
    val sc = new SparkContext(conf)
    val readmeData = sc.textFile(reademeFile, 2).cache()
    val numAs = readmeData.filter(line => line.contains("a")).count()
    val numBs = readmeData.filter(line => line.contains("b")).count()
    print("Lines with a: %s, Line with b: %s".format(numAs, numBs))

  }
}
```

## Spark 编程指南

### 概述

Spark 应用程序由一个在集群上运行着用户的 main 函数和执行各种秉性操作的 driver program(驱动程序)组成。Spark 提供的主要是一个弹性分布式数据集。

RDD: 弹性分布式数据集，是一个可以执行并行操作且跨集群节点的元素集合。

Spark 支持两种类型的共享变量： broadcast variables(广播变量)，它可以用于在所有节点上缓存一个值；accumulators(累加器)，它是一个只能被 “added” 的变量，例如 counters 和 sums。

### Spark 依赖

```
groupId = org.apache.spark
artifactId = spark-core_2.11
version = 2.2.0
```

### Spark 初始化

首先要创建一个 SparkContext 对象，它会告诉 Spark 如何访问集群。为了创建 SparkContext 需要构建一个包含应用程序信息的 SparkConf 对象。每个 JVM 可能只能激活一个 SparkContext，所以在创建新对象之前，必须调用 stop() 方法停止活跃的 SparkContext。

```
val conf = new SparkConf().setAppName(appName).setMaster(master)
val sparkContext = new SparkContext(conf)
```

- appName 集群 UI 上展示应用程序的名称。
- master 是一个 Spark、Mesos或 YRAN 集群的 URL 地址。

这里指定 local 表示在 local mode 中运行，在实际工作中，不会这样给 master 硬编码，而是通过 spark-submit 启动应用程序来接受它。

### shell 的使用

在 shell 中，一个特殊的 interpreter-aware (可用的解析器) SparkContext 已经创建好了，称之为 sc 变量。创建自己的 SparkContext 将不起作用。

* --master: 设置 SparkContext 连接到哪一个 master 上。
* --jar: 传递一个逗号分隔的列表来添加 jars 到 classpath 中。
* --packages: 用逗号分隔的 maven coordinates(maven 坐标) 方式来添加依赖到 shell session 中。
* --repositories: 添加依赖仓库。

```
./bin/spark-shell --master local[4]

./bin/spark-shell --master local[4] --jars code.jar

./bin/spark-shell --master local[4] --packages "org.examples:example:0.1"

```

## 弹性分布式数据集 (RDDS)

Spark 主要以一个弹性分布式数据集的概念为中心，它是一个容错且可以执行并行操作的元素的集合。有两种方法可以创建RDD：driver program 中 parallelizing 一个已存在的集合。或者在外部存储系统中引用一个数据集，例如，一个共享文件系统，HDFS、HBase或者提供 Hadoop InputFormat 的任何数据源。

### 并行集合

通过调用 SparkContext 的 parallelize 方法来常见并行集合。该集合的元素可以从一个可以并行操作的 distribute dataset 中复制到另一个 dataset（数据集） 中去。

```
val data = Array(1, 2, 3, 4, 5)
//data: Array[Int] = Array(1, 2, 3, 4, 5)

val distData = sc.parallelize(data)
//distData: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[2] at //parallelize at <console>:29

distData.reduce((a, b) => a + b)
```

并行集合中一个重要的参数是 partitions 的数量，它用来切割 dataset。Spark 将在集群中的每个分区上运行一个任务。

### 外部数据集

Spark 可以从 hadoop 所支持的任何存储源中创建 distributed dataset，包括本地文件系统、HDFS、Cassandra、HBASE、Amazon S3等。Spark 支持文本文件，SequenceFiles，以及任何其他的 Hadoop InputFormat。

可以使用 SparkContext 的 textFile 方法来创建文本文件的 RDD。此方法需要有一个文件的 URI，并且读取它们作为一个 lines 的集合。

```
val distFile = sc.textFile("data.txt")
```

除了文本文件外，Spark 的 Scala API 也支持一些其他的数据格式：

- SparkContext.wholeTextFiles 可以读取包含多个小文本的目录，并返回它们的每个（filename，content）对，这与 textFile 形成对比，它的每一个文件中的每一行将返回一个记录。

- 针对 SequenceFiles ，使用 SparkContext 的 SequenceFile[K, V]方法，其中 K 和 V 指的是它们在文件中的类型。

- 针对其它的 Hadoop InputFormats，您可以使用 SparkContext.hadoopRDD 方法，它接受一个 JobConf 和 input format 类。通过相同的方法可以设置你 Hadoop Job 的输入源。

- RDD.saveAsObjectFile 和 SparkContext.objectFile 支持使用简单的序列化的 Java Object 来保存 RDD。虽然不想 Avro 这种专用的格式一样高效，但是其提供了一种更简单的方式来保存任何的 RDD。

## RDD 操作

RDD 支持两种类型的操作：

* transformation： 在一个已存在的 dataset 上创建一个新的 dataset
* actions: 将在 dataset 上运行的计算结果返回到驱动程序。

Spark 中的所有 transformations 都是 lazy（懒加载）的，只有当需要返回结果给驱动程序时，transformations 才开始计算。 这种设计使得 Spark 的运行更高效。

### 传递函数给 Spark

当驱动程序在集群上运行时，Spark 的 API 很大程度上依赖于传递函数。有2种推荐方式来做到这点。

* 匿名函数的语法Anonymous function syntax， 它可以用于短的代码片段。
* 在全局单例对象中的静态方法。例如，你可以定义对象 MyFunctions 传递 MyFunctions.fun1, 具体如下：

```
object MyFunctions {
    def func1(s: Sttring): String = {...}
}

myRdd.map(MyFunctions.func1)
```


### 打印 RDD 的元素

常见的用于打印 RDD 的所有元素使用 rdd.foreach(println) 或 rdd.map(println)。然而，在集群 cluster 模式下，stdout 输出正在执行写操作 executors 的 stdout 代替，而不是在一个驱动程序上，因此 stdout 的 driver 程序不会显示这些。 要打印 driver 程序的所有元素，可以使用 collect() 方法首先把 RDD 放到 driver 程序节点上: rdd.collect().foreach(println)。 这可能导致 driver 程序内存耗尽，因为 collect() 获取整个 RDD 到一台机器如果你只需要 
打印 RDD 的几个元素，一个更安全的方法是使用 take(): rdd.take(100).foreach(println)。

## 共享变量

通常情况下，一个传递给 Spark 操作的函数 func 是在远程的集群节点上执行的。该函数 func 在多个节点执行过程中使用的变量，使用一个变量的多个副本。这些变量以副本的形式拷贝到每个机器上，并且各个远程机器上变量的更新并不会传播到 driver program。 通用且支持 read-write 的共享变量在任务间是不能胜任的。所以，Spark 提供了两种特定类型的共享变量: broadcast variables(广播变量)和 accumulators(累加器)。

### 广播变量

Spark 的 actio 操作是通过一系列的 stage 进行执行的，这些 stage 是通过分布式的 `shuffle` 操作进行拆分的。

Spark 会自动广播出每个 stage 内任务所需要的公共数据。这种情况下广播的数据使用序列化的形式进行缓存，并在每个任务运行前进行反序列化。

广播变量通过在一个变量 `V` 上调用 `SparkContext.broadcast(v)` 方法进行创建。广播变量是 `V`  的一个 `wrapper`（包装器），可以通过调用 `value` 方法来访问它的值，代码示例:

```
val broadcast = sc.broadcast(Array((1, 2, 3))
//broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

broadcastVar.value

res0: Array[Int] = Array(1, 2, 3)
```

### Accumulators (累加器)

这是一个仅可以执行 `added` 的变量来通过一个关联和交换操作，因此可以高效地执行并行操作。累加器可以
用于实现counter（计数，类似在 MapReduce 中的那样）或者 sums (求和)。

可以通过调用 SparkContext.longAccumulator() 或 SparkContext.doubleAccumulator() 方法创建数值类型的 accumulator 以分别累加 Long 和 Double 类型的值。集群上正在
运行的任务就可以使用 add 方法来累加数值，但是不能读取值，只有在 driver program 才可以使用 value 方法读取累加器的值。

一个展示 accumulator 被用于对一个数字中的元素求和

```
val accum = sc.longAccumulator()

sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))

accum.value
```

## 部署应用到集群中

在将应用打包成一个 JAR 或者一组 `.py`, `.zip` 文件后，就可以通过 `bin/spark-submit` 脚本将应用提交到任何支持的集群管理器中。

### 单元测试

Spark 可以友好的使用流行的单元测试框架进行单元测试。在将 master url 设置为 local 来测试时会简单的创建一个 SparkContext，运行你的操作，然后调用 SparkContext.stop() 方法将该作业停止。因为 Spark 不支持在同一个程序中并行的运行两个 contexts，所以需要确保使用 `finally` 块或则测试框架的 `tearDown` 方法将 context 停止。

