---
title: "Spark 学习笔记"
date: 2019-03-25T00:00:00+08:00
draft: false
tags: ["spark"]
slug: spark-learning-note
---

## 目标

- 概念
- 本地开发

## 本地环境(Mac OS X)

- [下载](http://spark.apache.org/downloads.html)

```sh
export JAVA_HOME=$(/usr/libexec/java_home -v 1.8)
export SPARK_HOME="$HOME/opt/spark-2.3.2-bin-hadoop2.6"
export PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.10.7-src.zip:$PYTHONPATH
export PATH=$SPARK_HOME/bin:$PATH
```

### python环境

```sh
virtualenv -p python3 spark-python3
source spark-python3/bin/activate
pip install pyspark
deactivate(仅从当前venv环境脱离时执行)
```

## helloworld

```sh
./bin/spark-submit examples/src/main/python/wordcount.py README.md
```
### spark-shell

## IDE(PyCharm)

- 首选项-Project Interpreter
- helloworld解读

## RDD

![rdd](/images/spark-learning/IMG_1501.jpg)

- partition
- 转换
- 行动
- lazy evaluation
- shuffle

## 并行化

- partition 与并发
- worker,executor,task,job
    ![rdd](/images/spark-learning/IMG_1500.jpg)

## Spark Streaming, DStream

### socket

```sh
./bin/spark-submit examples/src/main/python/streaming/network_wordcount.py localhost 9999
```
- web ui, http://localhost/4040

### kafka
- https://search.maven.org/search?q=g:org.apache.spark%20AND%20v:2.3.2 , 放置于jars目录,或者在命令行指定
- spark.io.compression.codec snappy (conf/spark-defaults.conf lz4依赖冲突,2.2.2没有此问题)
- 两种模式: receiver, direct

```sh
./bin/spark-submit --jars jars/spark-streaming-kafka-0-8-assembly_2.11-2.3.2.jar examples/src/main/python/streaming/direct_kafka_wordcount.py 192.168.16.22:9092 jiedian_adapter_heartbeat
```

## DataFrame, SparkSQL, Structured Streaming

- spark.sql.shuffle.partitions (default 200, 在单节点测试时,会造成极大的延迟).

```sh
./bin/spark-submit examples/src/main/python/sql/streaming/structured_network_wordcount.py localhost 9999
```
- udf
- 代码在哪里执行?
![rdd](/images/spark-learning/IMG_1502.jpg)

## 理论

- streaming 101
- [streaming 102](http://limuzhi.com/2017/04/09/%E6%B5%81%E5%BC%8F%E8%B6%85%E8%B6%8A%E6%89%B9%E9%87%8F-Streaming%20102%E7%BF%BB%E8%AF%91/)

## spark

- [子雨大数据之Spark入门教程(Python版)](http://dblab.xmu.edu.cn/blog/1709-2/)
- [RDD、DataFrame和DataSet的区别](https://www.jianshu.com/p/c0181667daa0)
- [Spark Streaming 不同Batch任务可以并行计算么？](https://www.jianshu.com/p/ab3810a4de97)
- [Spark Streaming 管理 Kafka Offsets 的方式探讨](https://www.jianshu.com/p/ef3f15cf400d)
- [Spark Streaming容错性和零数据丢失](https://zhuanlan.zhihu.com/p/26297594)
- [Structured Streaming Programming Guide结构化流编程指南](https://www.cnblogs.com/swordfall/p/8435987.html)

### pyspark

- py4j ![python,java](/images/spark-learning/IMG_1497.jpg) ![python app](/images/spark-learning/IMG_1499.jpg) 
- [Improving PySpark performance: Spark Performance Beyond the JVM](http://bit.ly/2bx89bn)
- [Python最佳实践指南](https://pythonguidecn.readthedocs.io/zh/latest/)
