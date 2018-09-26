---
layout: post
title: Spark学习之本地环境搭建和模板工程
date: '2018-09-25'
tags:
  - Spark
  - 大数据
categories: 
  - 技术
---


## 一、背景

Spark是现在非常流行的一个大数据分析引擎，许多大公司的数据分析都在使用它。简单来讲，Spark主要有以下几个特点：

- 速度快：得益于其DAG计算模型，更容易在内存中一次性完成操作，使得Spark比MapReduce要快很多。
- 支持多种语言：支持Java、Scala、Python等编程语言，甚至支持SQL语法，提供了丰富的API用于数据的处理，使用起来非常方便。
- 支持多种环境部署：Spark可以运行在Hadoop、Apache Mesos、Kubernetes、standalone，还可以读取不同来源的数据。

让我们赶紧进入Spark的大数据世界！

<!-- more -->

## 二、准备工作

以我使用的Mac OSX为例，搭建一个基本的Spark环境。

### 安装Java/Scala SDK

此处省略。

### 下载Spark

从[官网](https://spark.apache.org/downloads.html)download 2.x版本的Spark，然后解压出来放到一个目录。这里我下载的是2.2.2版本的Spark。（2.x版本的Spark和1.x版本的Spark运行时用到Hadoop环境也不同，目前主流应用都是2.x版本了。）


## 三、基本示例

### 1. 从Spark命令行运行

由于下载的Spark内置了Hadoop的Java Library，我们可以直接在命令行里面启动Spark，体验它的功能。

以经典的word count为例，我们从命令行体验一下Spark。

```
// 进入刚才解压好的Spark目录，运行下面spark-shell
$ ./bin/spark-shell
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
18/09/26 16:33:31 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Spark context Web UI available at http://10.22.123.167:4040
Spark context available as 'sc' (master = local[*], app id = local-1537950811993).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.2.2
      /_/

Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_66)
Type in expressions to have them evaluated.
Type :help for more information.
```

然后读取一个文本文件到RDD（Resilient Distribute Data，关于Spark RDD，可以看一下官方的[spark-rdd论文](http://spark.apachecn.org/paper/zh/spark-rdd.html)）中：

```
scala> val textRDD = spark.sparkContext.textFile("README.md")
textRDD: org.apache.spark.rdd.RDD[String] = README.md MapPartitionsRDD[1] at textFile at <console>:23
```

然后对这个RDD进行操作，计算各个单词的数量（word count有很多思路，我这里用了一种比较简单的）：

```
scala> val wordCountRDD = textRDD.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
wordCountRDD: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[4] at reduceByKey at <console>:25
```

把结果打印出来看下：

```
scala> wordCountRDD.foreach(println)
(package,1)
(For,3)
(this,1)
(Programs,1)
(Version"](http://spark.apache.org/docs/latest/building-spark.html#specifying-the-hadoop-version),1)
(Spark,16)
(Because,1)
(particular,2)
... 省略 ...
```

### 2. 打包成完整的程序运行

在命令行中运行只是一种基本的体验，真正的生产环境都是要打包后放到集群上去跑的，下面了解一下怎么打包。

官网的Quick Start文档中其实有sbt打包的说明，但本人比较喜欢gradle，所以也希望使用gradle来打包Spark应用。

我借助了[https://github.com/faizanahemad/spark-gradle-template.git](https://github.com/faizanahemad/spark-gradle-template.git)这个模板工程来编写、打包Spark应用。（其实Spark程序就是一个可以执行的jar，所以不管是用gradle还是sbt都可以。）

#### 下载并修改模板工程

下载模板之后，我们简单做一下修改，把Spark依赖改成前面我下载的`2.2.2`。


简单修改一下`build.gradle`以及`gradle.properties`文件（注意：这里只是最小的改动了一下，能满足基本体验，如果要作为一个干净的模板，还需要稍微多改点）。

```
// gradle.properties
scalaVersion=2.11.8
sparkVersion=2.2.2


// build.gradle
compile 'org.apache.spark:spark-core_2.11:'+sparkVersion
compile 'org.apache.spark:spark-sql_2.11:'+sparkVersion
```

然后模板工程中new一个scala对象`WordCount`，在main方法中编写单词统计的代码：

```
package template.spark

import org.apache.spark.sql.SparkSession

object WordCount {
  def main(args: Array[String]) = {
    val spark = SparkSession.builder()
      .appName("Word count example")
      .master("local[*]")
      .getOrCreate()
    val inputFile = if (args.length >= 1) args(0) else "people-example.csv"
    val textRDD = spark.sparkContext.textFile(inputFile)
    val wordCountRDD = textRDD.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
    wordCountRDD.foreach(println)
  }
}
```

#### 打包

进入工程目录，运行下面的命令，即可打包出jar文件，在工程目录下的`/build/libs/spark-gradle-template-1.0-SNAPSHOT.jar`：

```
./gradlew jar
```

#### 提交到本地的Spark环境运行

回到命令行，输入：

```
/你解压的Spark根目录/spark-2.2.2-bin-hadoop2.7/bin/spark-submit --class "template.spark.WordCount" --master "local[2]" ./build/libs/spark-gradle-template-1.0-SNAPSHOT-all.jar ./README.md
```

即可运行打包好的Spark-WordCount程序。注意，这里需要指定MainClass的名称为`template.spark.WordCount`，并在jar文件后面带上参数即可。

> 如果你的工程还需要依赖第三方Library（包括远程依赖和本地libs依赖），且打包的时候需要把它们都打入到你的jar包时，可以运行`./gradlew shadowjar`，这样就可以打包出一个包含所有依赖的fatjar文件了。


## 四、小结

万事开头难，动手让程序跑起来最重要。Spark对我们提供了很清晰的API（tranform和action算子），屏蔽掉了很多分布式大数据背后的细节，使得我们可以专注于编写业务代码，实现数据的统计分析。当然，为了让自己的程序更优秀，我们在集群上运行Spark任务的时候，还需要进行资源的设置（spark-conf），以及代码的优化。Spark官方提供了很多学习文档，需要好好阅读，希望接下来可以学习到更多大数据处理的思想和最佳实践，继续加油吧。


## 五、参考资料和延伸阅读（必看）

- [Spark Quick Start](https://spark.apache.org/docs/latest/quick-start.html)
- [Spark RDD](http://spark.apachecn.org/paper/zh/spark-rdd.html)
- [RDD Programming Guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html)
- [Cluster Mode Overview](https://spark.apache.org/docs/latest/cluster-overview.html)
- [Spark SQL, DataFrames and Datasets Guide](https://spark.apache.org/docs/latest/sql-programming-guide.html)
