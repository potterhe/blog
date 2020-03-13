---
title: "Spark调研笔记"
date: 2019-02-21T23:59:59+08:00
draft: false
tags: ["spark"]
---


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
- [是时候放弃 Spark Streaming, 转向 Structured Streaming 了](https://zhuanlan.zhihu.com/p/51883927)

## 选项

- spark.io.compression.codec snappy (lz4依赖冲突)
- spark.streaming.concurrentjobs (spark streaming实时大数据分析4.4.4)
- spark.streaming.receiver.writeaheadlog.enable (spark streaming实时大数据分析5.6节)
- spark.sql.shuffle.partitions (default 200, 在单节点测试时,会造成极大的延迟).

## pyspark

- [Improving PySpark performance: Spark Performance Beyond the JVM](http://bit.ly/2bx89bn)
- [Python最佳实践指南](https://pythonguidecn.readthedocs.io/zh/latest/)

## 本地环境搭建

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
