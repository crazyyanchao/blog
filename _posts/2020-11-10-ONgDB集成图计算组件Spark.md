---
title: ONgDB集成图计算组件Spark
tags: [ONgDB,Spark]
author: Yc-Ma
show_author_profile: true
key: 2020-11-10-ONgDB集成图计算组件Spark
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 版本信息
```
Spark 2.4.0  http://archive.apache.org/dist/spark/spark-2.4.0/
Neo4j 3.5.x
Driver 1.7.5
Scala 2.11
JDK 1.8
hadoop-2.7.7
https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/
neo4j-spark-connector-full-2.4.1-M1 https://github.com/neo4j-contrib/neo4j-spark-connector
```
- 下载包
```
hadoop-2.7.7
spark-2.4.0-bin-hadoop2.7
winutils
neo4j-spark-connector-full-2.4.1-M1 【把jar包放到spark/jars文件夹里】
scala-2.11.12
```

## 基本操作
- 下载对应安装包
- 配置对应PAK_HOME和PATH

## 替换Hadoop的bin文件夹
```
替换对应版本Hadoop的bin文件夹
https://github.com/cdarlint/winutils
```

## 启动
```
C:\Users\mayc01>spark-shell
```

## 访问地址
```
http://localhost:4040/jobs/
```

## 运行测试Spark计算
- 进入终端后输入sc，查看一下Spark-shell帮我们自动生成的SparkContext的实例
```
sc
```
- 生成一个最简单的 List集合RDD
```
val rdd=sc.parallelize(List(1,2,3,4,5,6,7,8,9))
```
- 对集合的每个元素都乘以3
```
val mappedRDD=rdd.map(3*_)
```
- 通过collect 查看结果
```
mappedRDD.collect
```
- 测试运行
```
C:\Users\mayc01>spark-shell
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://HPO-MAYC.js.local:4040
Spark context available as 'sc' (master = local[*], app id = local-1604977446437).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.0
      /_/
Using Scala version 2.11.12 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_251)
Type in expressions to have them evaluated.
Type :help for more information.
scala> sc
res0: org.apache.spark.SparkContext = org.apache.spark.SparkContext@147892be
scala> val rdd=sc.parallelize(List(1,2,3,4,5,6,7,8,9))
rdd: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at parallelize at <console>:24
scala> val mappedRDD=rdd.map(3*_)
mappedRDD: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[1] at map at <console>:25
scala> mappedRDD.collect
res1: Array[Int] = Array(3, 6, 9, 12, 15, 18, 21, 24, 27)
scala> mappedRDD.collect
res2: Array[Int] = Array(3, 6, 9, 12, 15, 18, 21, 24, 27)
scala> mappedRDD.collect
res3: Array[Int] = Array(3, 6, 9, 12, 15, 18, 21, 24, 27)
scala>
```
## 运行测试Spark+ONgDB计算

